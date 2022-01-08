---
header-includes:
  - \usepackage{algorithm2e}
---

# CUDA Managed Memory

So far we had to copy date from CPU memory to GPU memory before launching the CUDA kernel and after the kernel finishes
the computing transfer it back to CPU memory. This also entails keeping different pointers for the same data, one for CPU and one for GPU.

CUDA offers a simplified memory access using **Unified Memory**. One simply replaces  ```cudaMalloc``` with ```cudaMallocManaged```. So instead of, say,
```cpp
float *x,*dx;
x=(float *)malloc(n*sizeof(float));
cudaMalloc(&x,n*sizeof(float));
```
We write 
```cpp
float *x;
cudaMallocManaged(&x,n*sizeof(float));
```

After that, the CUDA runtime will take care of wether ```x``` is a host or device pointer. There is, however, one additional requirement. When the kernel finishes, the results
might not be transferred to the CPU yet. To make sure that it is, we call ```cudaDeviceSynchronize```.

1. Allocated memory using ```cudaMallocManaged```
1. Launch the kernel
1. Call ```cudaDeviceSynchronize```


## Managed Memory (details)

In this section we will look in more details of how Unified Memory works.
In pre-Pascal architecture, ```cudaMallocManaged``` will allocate memory on the device. When the CPU accesses  data that is actually on the device, it page faults and the data is transferred to host memory. That is basically demand paging, but instead of the data being on disk, it is in GPU memory. In pre-Pascal architecture, the opposite direction, from host to device, did not quite work the same way. When a kernel is launched, the migration engine on the GPU will transfer all allocated data to GPU memory, in case the GPU needs them.

Starting from the Pascal architecture, the GPU uses "demand paging" like the CPU. If a thread accesses a page that is not resident in GPU memory, it page faults and the migration engine will transfer said page to GPU memory.

We illustrate the above difference with the code below. Note that ```w``` points to a memory allocated but never used on the GPU.

```cpp
#include <iostream>
#include <cuda_runtime.h>

template <typename T>
__global__ void saxpy(T *z,T *x,T *y,T a,int n){
    /* since we are using a single block
     * threadIdx.x is the thread id
     */
     int i=blockDim.x*blockIdx.x+threadIdx.x;
     if (i <n)
        z[i]=a*x[i]+y[i];

}
int main(){
    const int n=1<<20;
    const float a=3.0;
/* Allocate three memory chunks*/
    float *x, *y, *z,*w;
    cudaMallocManaged(&x,n*sizeof(float));
    cudaMallocManaged(&y,n*sizeof(float));
    cudaMallocManaged(&z,n*sizeof(float));
    cudaMallocManaged(&w,n*sizeof(float));

    cudaError_t e=cudaGetLastError();
    if(e!=cudaSuccess)std::cout<<cudaGetErrorString(e)<<"\n";
/* populate x and y */
    for(int i=0;i<n;++i){
        x[i]=2;
        y[i]=4;
        z[i]=0;
        w[i]=0;
    }
    dim3 block (256);
    dim3 grid ((n+block.x-1)/block.x);
    saxpy<<<grid,block>>>(z,x,y,a,n); 
    cudaDeviceSynchronize();
    e=cudaGetLastError();
    if(e!=cudaSuccess)std::cout<<cudaGetErrorString(e)<<"\n";
/* check if the result is correct. We expect all values 
 * of z=10
 */
    int sum=0;
    for(int i=0;i<n;++i)
        sum+=z[i];
    if (sum!=n*10)std::cout<<"sum error"<<sum<<"\n";
    else 
        std::cout<<"check passed. Sum= "<<sum<<"\n";
    cudaFree(x);
    cudaFree(y);
    cudaFree(z);
}
```
To compare we run the code on Tesla K80 (Kepler) and Tesla P100 (Pascal).
**NOTE**: Unified Memory features are limited on Windows (and WSL).

When we run ```nvprof``` on K80 we get the follwing :


```
==164== Unified Memory profiling result:
Device "Tesla K80 (0)"
   Count  Avg Size  Min Size  Max Size  Total Size  Total Time  Name
      12  1.3333MB  128.00KB  2.0000MB  16.00000MB  2.365652ms  Host To Device
     140  146.29KB  4.0000KB  0.9961MB  20.00000MB  3.146866ms  Device To Host
Total CPU Page faults: 70

```
As mentioned before, the pre-Pascal (K80) has limited support for page migration. It can be described as a "demand paging in one direction". Initially, when ```cudaMallocManaged``` is called, 4MB is allocated for each pointer, ```x,y,z,w``` _on the device_. After allocation, the four blocks are initialized _on the CPU_. This causes a copy of 16MB (via page faults) from device to host memory. When the kernel is launched, the runtime does not know what data will be used so it transfers **all** 16MB from to the host to the device. Finally, the values of ```z``` are accessed by the host which causes an additional 4MB to be transferred from device to host, which brings the total to 20MB.

On the other hand, the output of ```nvprof``` on the Pascal architecture (P100) is
```
Device "Tesla P100-PCIE-16GB (0)"
   Count  Avg Size  Min Size  Max Size  Total Size  Total Time  Name
     270  45.511KB  4.0000KB  960.00KB  12.00000MB  1.455065ms  Host To Device
      24  170.67KB  4.0000KB  0.9961MB  4.000000MB  351.4860us  Device To Host
      22         -         -         -           -  5.218884ms  Gpu page fault groups
Total CPU Page faults: 60

```
First notice the additional line of "Gpu page fault groups". In this case, allocation is performed when the data _is first used_, in this case at the host, when the variables are initialized. Therefore at this point the data is on the host and no transfer happens. When the kernel is launched and the threads start accessing the data, page faults occur _on the device_, and the _needed_ data is transferred from the host to the device. Since ```w``` is not used, only 12 MB are transferred from host to device. At the end, the host accesses the ```z``` data and thus 4MB are transferred from device to host.
## Compute Capabilities

From the CUDA docs:
"The compute capability of a device is represented by a version number, also sometimes called its "SM version". This version number identifies the features supported by the GPU hardware and is used by applications at runtime to determine which hardware features and/or instructions are available on the present GPU."

https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#compute-capabilities
https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#compute-capability

https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html#gpu-feature-list
https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html#gpu-compilation


1. The option ```--gpu-architecture=compute_XY``` alternatively ```-arch=compute_XY```. This option creates ptx code from the input file, in phase 1, for virtual architecture ```XY```. *If and only if* ```--gpu-code``` is NOT specified the ptx is included in the executable.

1. The option ```--gpu-code=compute_XY,sm_ab,sm_bc``` controls what code is generated in phase 2 (included in the final executable). The ```compute``` specifies a *virtual* architecture while the ```sm``` specifies a *real* architecture. Only one ```compute``` can be specified but multiple ```sm``` can be specified. The ```compute``` option is important in case the code is executed on a GPU different than the specified ```sm```. In that case the ptx code included in the executable will be *compiled* dynamically (on the fly) before it runs. 

For a list of GPU generations with corresponding SM's see https://arnon.dk/matching-sm-architectures-arch-and-gencode-for-various-nvidia-cards/

Google colab has Tesla K80 (kepler, sm_37),  Tesla P100 (Pascal,sm_60), and Tesla T4(Turing,sm_75). Most of the time you get Tesla K80


Each cache line is 4 sectors, each sector is 32 bytes, for a total of 128 bytes.
Ideally for a warp (32 threads) 4 sectors are transferred for each request.
IN  nsight look for sector/request ratio.
https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html#memory-chart

nv-nsight-cu-cli --metrics l1tex__average_t_sectors_per_request_pipe_lsu program.exe
nv-nsight-cu-cli --metrics smsp__average_warps_issue_stalled_long_scoreboard_per_issue_active program.exe

# Nsight
Note that the latest nsgight compute does not support pascal and earlier.

NVIDIA has two profiling tools: 
1. Nsight system for overall view of the application including the host
1. Nsight compute to profile CUDA kernels.
