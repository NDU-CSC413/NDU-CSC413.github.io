# C++ Review
Below are the list of topics  covered in this review. This document is __not__ by any stretch 
a comprehensive reference of cpp, it covers some of the topics that we will be using in our course only. All the code here are can be imported into MSVC as a single solution from this [repository](https://github.com/NDU-CSC413/c-review).

1. [Variables and References](#variables-and-references)
2. [Classes](#classes)
3. [Move semantics](#rvalue-reference-and-move-semantics)
4. [Return values](#return-values)
5. [Pointers](#pointers)
6. [Templates](#templates)
7. [STL](#algorithms-in-stl)
# Variables and references

When we define (declare) a variable the system reserves space in memory to store the
value associated with that variable. This is the reason why we need to specify the 
type of the variable since the required space depends on it. For example (typically), 
an ```int``` and ```float``` need 4 bytes whereas ```long long``` and ```double``` need 
8 bytes. Because every variable is associated with a location in memory we can
determine the memory address where the variable is located using the ```&``` operator. Note 
that the ```&``` operator can have different meaning depending on context.
Example.
```cpp
#include <iostream>
int main(){
//a location in memory is reserved and labeled x
int x=2;
//y is just another name for the same location. no reservation is done.
int& y=x;
//a location is reserved for z and the value of x is copied
int z=x;
z=17;
y=13;
//print the value of the variables and their respective addresses
std::cout<<"x= "<<x<<" and &x="<<&x<<std::endl;
std::cout<<"y= "<<y<<" and &y="<<&y<<std::endl;
std::cout<<"z= "<<z<<" and &z="<<&z<<std::endl;

}
```

Note that ```int& y=x;``` declares _y_ as a reference to _x_ whereas ```&x``` gives the memory address of _x_. The different declarations used above carry to the parameters in function calls. For example,

```cpp
#include <iostream>
void byValue(int n){
    n=17;
}
void byRef(int& n){
    n=12;
}
int main(){
  int x=2;
  byValue(x);
  std::cout<<x<<std::endl;
  byRef(x);
  std::cout<<x<<std::endl;

}
```

So in the call to the function ```byValue(x)``` it is __as if__ we declare ```int n=x;``` and therefore _n_ is a __copy__ of _x_. By contrast, ```byRef(x)``` is is __as if__ we declare ```int& n=x;``` so no copy is made and _n_ is a reference to _x_.
Usually we call by reference when either we want to change the input or  when the input is large and copying becomes expensive. We can use the best of both by using a const reference

```cpp
int byCRef(const int& n){
    n=37;//error cannot modify n
    return 2*n;
}
```
Also, const allows us to pass literals and temporaries.

```cpp
int byT(int n){
    return 7*n;
}
int byCRef(const int & n){
    n=n+1;//error n is const
    return 2*n;
}
int byRef(int& n){
    n=n+1;//changes the value of parameter
    return 2*n;
}
int main(){
    byRef(2);//error cannot bind a non-const lvalue to rvalue
    byCRef(2);//OK
    byRef(byT(2));//error since the return value of byT is a temp
    byCRef(byT(2));//OK
    int& r=byT(2);//error cannot bind 
    int&& res=byT(2);//ok
}
```

You can run the above code [here](https://godbolt.org/z/e8v8cx).

One important property of references is that they __cannot be reassigned__.
```cpp
#include <iostream>
int main(){
int x=19,y=23;

//int &xr;//error a reference must be initialized
int& xr=x;
int& yr=y;
/* this does NOT reassign xr to reference y
* it merely assigns the value of y to xr and thus to x
*/
xr=yr;
std::cout<<"value of x="<<x<<"\n";
xr=45;
std::cout<<"value of x="<<x<<"\n";
std::cout<<"value of y="<<y<<"\n";
}
```
You can try it [here](https://godbolt.org/z/dzqe7e)

We can prevent modification to a variable we reference to. For example in the above if we declare ```const int& xr=x;``` it means that ```x``` cannot be modified using ```xr``` (it can still be modified). In this case  the line ```xr=yr;``` will generate an error. Note that in some text they write ```int const& xr=x;``` which is equivalent but i prefer the first syntax which conveys the fact that the ```int``` is unmodifiable not the reference, since by definition a reference is unmodifiable.
As we will see later a class containing a member of type reference cannot be assigned (assignment operator is deleted)

# Classes

In C++ new types are created using classes. Once a class is defined new objects can be instantiated from such a class. Minimal syntax of a (useless) class

```cpp
class Test{};
int main(){
    Test t;
}
```

A class can have __member variables__ and __member functions__.

```cpp
class Test{
    int _x;
    public:
    int& x(){
        return _x;
    }
    
};
int main(){
    Test t;// at this point _x is undefined
    t.x()=17;
}
```
By default all members of a class are __private__ and hence inaccessible from outside the scope of the object. To make a member accessible we use the keyword __public__. Note that the __member function__ ```x()``` returns a __reference__ to _x and this allows us to change the value of _x. We can use pointers as usual where the code below is equivalent to the one above but using the arrow instead of the dot operator.

```cpp
int main(){
    Test * p=new Test();
    p->x()=17;
}
```

In fact the private and public qualifiers can be used for any and all members. For example

```cpp
class Test {
    private: int _x;//_x is private
    public: int _y;// _y is public
     int _z;// _z is public. The keyword carries over until it changes
     private: void f(){}
     public: int& x(){return _x;}
}
```
But usually all public members are grouped together using a single keyword and the same for private members;

```cpp
int main(){
    Test t;
    t._x;//error _x is inaccessible
    t._y=t._z;//OK both are public
    t.x();//OK
    t.f();//Error f is inaccessible
}
```

## Constructors and destructors

For builtin types like ```int``` and ```double``` a variable is "created" (memory is reserved) when the variable is declared. Once the variable is out of scope is it "destroyed" (memory is released). The same thing is done for objects instantiated from classes. This is done by using __constructor__ and __destructor__. When we don't supply our own versions a default version is used by the compiler which basically calls the constructors and destructors of the member variables.

```cpp
class Test {
    public:
    int _x;
    double _y;
}
int main(){
    Test t;
    std::cout<<t._x<<std::endl;
    std::cout<<t._y<<std::endl;
}
```
No constructor is supplied so the compiler uses a default that creates variables _x and _y. 

A constructor builds an object bottom up.
1. the constructor of the base class (if any) is called
1. members instructors are called
1. Finally the constructor body is executed.

For the _destructor_ the __opposite__ happens.
For example

```cpp
struct Item {
    Item(){std::cout<<"Item ctor\n";}
    Item(int i){std::cout<<"Item ctor with input\n";}
};
struct Test {
    Item _i;
    int x;
};
void noinit(){
    int x=12,y=77,z=99;
    Test t;
    std::cout<<t.x<<std::endl;
}
void init(){
    int x=12,y=77,z=99;
    Test t {};//initialize to zero
    std::cout<<t.x<<std::endl;
}
int main(){
    noinit();
    init();
}
```
Run to get the output
```cpp
item ctor
723520304
item ctor
0
```
As we can see from the above example built-in types are __not__ initiaized: some times they are zero sometimes they are not, it depends on the compiler. For class types the default constructor is called. We can control the constructor and the initialization of members as follows
```cpp
struct Test {
    Test(int x,int i):_x(x),_i(i){}
}
```

As mentioned before classes containing references or constants have their assignment operator automatically deleted 
```cpp
#include <iostream>
struct TestRef {
    int& _x;
    TestRef(int& x) :_x(x) {}
};
struct TestConst {
    const int _x;
    TestConst(int x) :_x(x) {}
};
int main()
{
    int x = 12;
    TestRef t(x);
    TestRef p(x);
    TestConst c(x);
    TestConst d(x);
    p = t;//error assignment operator deleted
    c = d;//error assignment operator deleted
}
```
You can see the errors [here](https://godbolt.org/z/GY564h)

# Rvalue reference and move semantics
Since C++11 there is a new type of references called **rvalue** references.
The variable _res_ below extends the lifetime of the temporary object created by the ```RT()``` function.  To see that consider when the destructor is called in the following code

```cpp
#include <iostream>
struct Test {
  int _x;
  Test(int x=0):_x(x){}
  ~Test(){
    std::cout<<"dtor "<<_x<<std::endl;
  }
};
Test RT(int val){
   return Test(val);
}
int main() {
Test&& res=RT(8);
std::cout<<"creating 7\n";
RT(7);
std::cout<<" done\n";

}
```
You can run the above code [here](https://godbolt.org/z/4EacM3)


__Note__ that  when an rvalue reference  is used, it is used as a lvalue reference. This is called **move semantics is not passed through** . For the example the following recursive function gives an error

```cpp
void doit(std::string&& s){
  if(s!="hello")
    doit(s);

}
```
this is a fix

```cpp
void doit(std::string&& s){
  if(s!="hello"){
    s="hello"; //this line so we don't go into infinite recursion
    doit(std::move(s));
  }
    
}
```
Move semantics allows us to transfer ownership of resources.

```cpp
#include <iostream>
#include <string>
#include <vector>

int main(){
std::string s{"hello there"};
std::string ts=std::move(s);
std::cout<<"the string s is "<<s<<"\n";
std::cout<<"the string ts is "<<ts<<"\n";
std::vector<int> v{1,2,3,4};
std::cout<<"values of v before move are ";
for(auto& x:v)std::cout<<x<<",";
std::vector<int> tv=std::move(v);
std::cout<<"\nafter move they are ";
for(auto& x:v)std::cout<<x<<",";
std::cout<<"\nand the values of tv are ";
for(auto& x:tv)std::cout<<x<<",";
std::cout<<std::endl;

}
```
you can try it [here](https://godbolt.org/z/79a6zY). We illustrate further with our own, very simple, container.
```cpp
#include <iostream>
struct Container {
    int* p = nullptr;
    int _size;
    Container(int size) :_size(size), p(new int[size] {}) {}
    Container(const Container& rhs) {//copy constructor
        if (&rhs != this) {
            _size = rhs._size;
            p = new int[_size];
            for (int i = 0; i < _size; ++i)
                p[i] = rhs.p[i];
        }
    }
    //usually there is also an assignment operator
    Container(Container&& rhs) {
        p = rhs.p;
        rhs.p = nullptr;
        rhs._size = 0;
    }
    Container& operator=(Container&& rhs) {
        if (p)delete[] p;
        p = rhs.p;
        _size = rhs._size;
        rhs._size = 0;
        rhs.p = nullptr;
        return *this;
    }
    void print() {
        for (int i = 0; i < _size; ++i)
            std::cout << p[i] << ",";
        std::cout << "\n";
    }
    ~Container() {
        if (p)delete[] p;
    }
};

int main()
{    
    Container c(10);
    std::cout<<"Content of c \n";
    c.print();
    Container d(std::move(c));
    std::cout<<"Content of c \n";
    c.print();

    Container e(3);
    d = std::move(e);
    std::cout<<"Content of d \n";
    d.print();
}
```
You can try it [here](https://godbolt.org/z/1Gnvhb)
# Return values
unless the compiler performs return value optimization (rvo) the following occurs
(in g++ or clang++ specify -fno-elide-constructors to skip optimization)

```cpp
#include <iostream>
struct Test {
        Test(){
          std::cout<<"ctor\n";
        }
        Test(const Test& rhs){
          std::cout<<"copy ctor\n";
        }
        ~Test(){
          std::cout<<"dtor\n";
        }
};
Test retTest(){
        return Test();
}
int main(){
  Test t=retTest();
  std::cout<<"temporary returned by refTest is destroyed\n";
  std::cout<<"end of main t will be destroyed\n";
}
```
what happens is the following
1. inside function ```retTest()``` an object of type Test is created on the stack
1. a tmp object of type Test is copy constructed from that object
1. the object on the stack is dtored
1. t in main is copy ctored from the tmp
1. tmp is destroyed
1. when main exists t is destroyed

You can test the code below [here](https://godbolt.org/z/sYa9KG). **Note** the -fno-elide-constructors option in the bottom right of the screen.
```
$g++-10 -fno-elide-constructors -std=c++11 rvopt.cpp
$./a.out
ctor
copy ctor
dtor
copy ctor
dtor
temporary returned by refTest is destroyed
end of main t will be destroyed
dtor
```
If we remove the -fno-elide-constructors you get this output. Try it [here](https://godbolt.org/z/r8nhYE)
```
$g++-10 -std=c++20 rvopt.cpp
$./a.out
ctor
temporary returned by refTest is destoryed
end of main t will be destroyed
dtor
```
__Note__: since c++17 this is no longer possible. Try it [here](https://godbolt.org/z/9687h6)

# Pointers
A pointer variable is a variable that holds an address. We say variable _p_ points to variable _x_ if _p_ holds the address of _x_: ```int *p=&x;```.

```cpp
int main(){
int x=17,y=45;
int* p=&x;
std::cout<<p<<std::endl;//prints the value of p, i.e. the address of x
std::cout<<*p<<std::endl;//prints the value store at the location p, i.e. x
*p=23;//change the value of x
p=&y;//p now stores the address of y
}
```

Pointers usually are used when we need to dynamically allocate memory.

```cpp
int main(){
    int *p=new int;//reserve space for int. Value undefined
    int *q=new int(8);//reserve space for int and store 8
    *p=55;//store value 55 at address p
    delete p;//release the reserved memory;
}
```
The __new__ operator can be used with any object. In particular, we can use it to create an array dynamically
```cpp
int main(){
    const int n=8;
    int* p=new int[n];//Note the difference from int(n)
    /* fill the array with values */
    /* p IS a variable so we make
     * copy before changing it
     */
    for(int i=0,*q=p;i<n;++i,++q)
         *q=i;
     /* we use it as an array
      * good thing we kept the
      * original p
      */
    for(int i=0;i<n;++i)
      std::cout<<p[i]<<",";
    std::cout<<"\n";

}

```
As we mentioned before we can use a pointer to any type.

```cpp
#include <iostream>
class Test {
int _x,_y;
public:
 Test(int x,int y):_x(x),_y(y){}
 int& getX(){return _x;}
 int& getY(){return _y;}
};
int main(){
    Test* t=new Test(13,18);
    t->getX()=3;
    t->getY()=7;
    std::cout<<++t->getX()<<"\n";
    std::cout<<++t->getY()<<"\n";
}

```
You can try the above code [here](https://godbolt.org/z/zMach4)

## new, malloc, operator new

A __new__ expression is used both for dynamically allocating memory(on the heap) __and__ calling the constructor of an object. The function __operator new__ allocates memory __only__. In that sense it is similar to malloc in C. Unless you are designing your own container you __almost never__ need to use __operator new__. Usually it is used to _place_ the constructed object at a _preallocated_ place.
Example

```cpp
#include <iostream>
#include <new>
struct Test {
 int _x,_y;
 Test(int x,int y):_x(x),_y(y){std::cout<<"ctor\n";}
 ~Test(){std::cout<<"dtor\n";}
};
int main(){
void* p=operator new(sizeof(Test));
std::cout<<"finished allocating memory\n";
Test* t=new (p) Test{1,2};
delete t;
}
```
You can run the code [here](https://godbolt.org/z/faYq3Y).

We can override the implementation of __operator new__ and __operator delete__. As can be seen below new and delete are the C++ "versions" of C malloc and free.

```cpp
#include <iostream>
class Test {
int _x,_y;
public:
 Test(int x,int y):_x(x),_y(y){std::cout<<"ctor\n";}
 int& getX(){return _x;}
 int& getY(){return _y;}
 ~Test(){std::cout<<"dtor\n";}
};

void * operator new(std::size_t size){
   std::cout<<"allocating size ="<<size<<" \n";
   void *p=malloc(size);
   return p;

}
void operator delete(void *p) noexcept {
    std::cout<<"freeing memory\n";
    free(p);
}
int main(){
    Test *t=new (Test){1,2};
    delete t;
    Test *p=new Test{3,4};
    p->~Test();
    operator delete(p);
  
  
}
```
https://godbolt.org/z/orqKPq

## Smart pointers

While pointers provide flexibility they can cause a variety of problems. One of the problems is shared ownership of a resource. As discussed before, in many situations a pointer variable contains the address of a dynamically allocated memory. We also saw that, in order not to have memory leaks, we need to free the allocated memory when done. This particular problem occurs when two or more pointer variables have the address of the same dynamically allocated resource. In such a situation we either try to free the memory multiple times, access a resource that no longer exists , or forget to release the memory which causes memory leaks.
We illustrate with the following simple example.

```cpp
#include <iostream>
#include <memory>
#include <thread>
#include <chrono>
using Long = unsigned long long;

void* memory_block(size_t size) {
	return operator new(size);
}
void release_memory(void* p) {
	operator delete(p);
}

struct Leaker {
	int* _values=nullptr;
	int _size;
	Leaker(Long size) :_size(size) {
		_values = (int*)memory_block(_size * sizeof(int));
	}
	int* values() {
		return _values;
	}
    ~Leaker(){//Do we free memory here ?
    }
};
int main() {
	const unsigned long long n = 1 << 20;
	for (int i = 0; i < 50; ++i) {
		Leaker x(n);
		int* p = x.values();
    }
    /* keep the program running long enough */
	int y;
	std::cout << "type anything\n";
	std::cin >> y;
}
```
In the above code, the class ```Leaker``` allocates 1MB of memory. It has a choice, either it __does not free__ the allocated memory, as we did in the example above, or frees it in the destructor with the danger that ```p``` will also free it. Using the performance profiler in VS (Debug->Performance profiler) we see that the above code uses about 200MB
which makes sense since each iteration allocates 4MB.

![Figure 1](figs/large_memory.png)

The other choice is to modify the destructor as follows
```cpp
~Leaker(){
    if(_values!=nullptr)delete _values;
}
```
This works provided the user (i.e. the code calling ```int p*=x.value()```) does not execute ```delete p;``` which crashes the program. You can try that scenario [here](https://godbolt.org/z/WT7rjx)


To avoid problems like these we use either std::unique_ptr<T> or std::shared_ptr<T>. The first enforces __exclusive__
ownership whereas the second allows __shared__ ownership. We start with an example of the second.

```cpp
	#include <iostream>
	#include <memory>
	#include <thread>
	#include <chrono>
    using Long = unsigned long long;

void* memory_block(size_t size) {
	return operator new(size);
}
void release_memory(void* p) {
	operator delete(p);
}
struct Shared_Owner {
	std::shared_ptr<int> _values;
	Long _size;
	Shared_Owner(Long size) :_size(size) {
		/* in three steps for clarity */
		void* raw = memory_block(_size * sizeof(int));
		std::shared_ptr<int> q((int*)raw);
		_values = q;
		/*q will be destroyed here. It is ok
		* it holds no resource since it was
		* transfered to p
		*/
	}
	std::shared_ptr<int> values() {
		return _values;
	}
    long count(){
        return _values.use_count();
    }
};
int main(){
    const unsigned long long n = 1 << 20;
    Shared_Owner x(n);

	{
		std::shared_ptr<int> p = x.values();
        std::cout<<"after p is created="<<x.count()<<"\n";
        std::shared_ptr<int> q=x.values();
        std::cout<<"after q is created="<<x.count()<<"\n";
    }
    std::cout<<"count outside block= "<<x.count()<<"\n";
}
```
First, there is no __delete__ of a ```std::shared_ptr``` because it is automatically destroyed (i.e. dtor is called) when it goes out of scope __and__ it frees the resource it is pointing to __only if__ it is the __last__ shared_ptr copy.  Second, it is used exactly like pointers.
In the example above there are three ```std::shared_ptr```, all pointing to the resource allocated by the ```Shared_Owner``` object _x_. Inside the block two ```std::shared_ptr``` objects are created _p_ and _q_, both pointing to the memory block allocated by _x_. When they go out of scope, and therefore they are destroyed,  the allocated memory is not freed, the number of copies is just decremented.
Now when _x_ goes out of scope, it is the last shared_ptr pointing to the resource and therefore it frees it.
You can try the above example here [here](https://godbolt.org/z/c9159K).


The ```std::unique_ptr<T>``` is similar except it enforces __exclusive__ ownership. Below is a similar  example illustrating the memory automatically released by the ```std::unique_ptr``` when it is destroyed. Note the two changes due to noncopiable nature of ```std::unique_ptr```. First, in function ```mod``` the ```std::unique_ptr``` must be passed and returned by reference because we cannot make a copy. Also
in the statement ```    std::unique_ptr<int> t = std::move(mod(p->res));``` the resource held by _p_ is transferred to _t_.
```cpp
#include <iostream>
#include <memory>
#include <thread>
#include <chrono>
using Long = unsigned long long;

void* memory_block(size_t size) {
	return operator new(size);
}
void release_memory(void* p) {
	operator delete(p);
}

struct Unique_Owner {
	std::unique_ptr<int> _values;
	Long _size;
	Unique_Owner(Long size) :_size(size) {
		/* in three steps for clarity */
		void* raw = memory_block(_size * sizeof(int));
		std::unique_ptr<int> q ((int *)raw);
		_values = std::move(q);
		/*q will be destroyed here. It is ok 
		* it holds no resource since it was
		* transfered to p
		*/
	}
	std::unique_ptr<int> values() {
		return std::move(_values);
	}
};
int main(){
    const unsigned long long n = 1 << 20;
	for (int i = 0; i < 50; ++i) {
		Unique_Owner x(n);
		std::unique_ptr<int> p = x.values();
		std::this_thread::sleep_for(std::chrono::microseconds(10));
    }
}
```
Using the performance profiler in VS (Debug->Performance profiler) we see that the above code uses only 4MB
which means each iteration  4MB are allocated and then freed.

![Figure 1](figs/small_memory.png)

# Templates

On many occasions we write multiple versions of the same code to handle different types. For example suppose we want to write a function to add two numbers (using the + operator) we write

```cpp
int add(int x,int y){
    return x+y;
}
int add(double x,double y){
    return x+y;
}

```
Recall also that the + operator can be used to concatenate strings so we have to add that also. Since the all of those versions only the type changes, c++ allows us to pass the type as a parameters using templates.

```cpp
#include <iostream>
#include <string>

template<typename T>
T add(T x,T y){
    return x+y;
}
int main(){
    int x=2,y=3;
    double u=3.4,v=3;
    std::string s="hello",k="there";

    std::cout<<add(x,y)<<std::endl;
    std::cout<<add(u,v)<<std::endl;
    std::cout<<add(s,k)<<std::endl;
}

```

In the above example the compiler automatically deduces the type which sometimes it cannot and we have to specify it as follows:

```cpp
add<int>(x,y);
add<double>(u,v);
add<std::string>(s,k);
```

Note that the template is instantiated __as needed__ at compile time. Also, we can pass parameters to the template other than types. For example

```cpp
template <int n>
void doit(){
    int a[n];
}
```
## Template specialization
 The ```add``` example above doesn't make any sense if used with ```char```. Try adding the two characters 'a' and 'b'. Therefore we would like to change the definition of ```add``` when the parameter type is ```char```. This is done using __template specialization__. One reasonable "addition" of characters would be to obtain a character a the same distance from the last one. For example, the distance (in ASCII code) between 'a' abd 'b' is 1 so the result of ```add('a','b')```
 would be 'c' since ```b+1=c```. Below is the syntax for __template specialization__.
 ```cpp
 template<typename T>
T add(T x,T y){
    return x+y;
}
 template<>
 char add<char>(char a,char b){
   return a<b?b+(b-a):a+(a-b);
 }
 ```

 Note that the specialization starts with ```template<>``` and every occurence of ```T``` was replaced by ```char```.
 Even though the second implementation of ```add``` for ```char``` makes more sense that the first it is not satisfactory.
 What we would like is for the result of ```add('a','b')``` to give "ab". This means that the signature would become
 ```std::string add(char,char )```. We could be tempted to specialize it as follows
 ```cpp
 template<>
 std::string add<char>(char a,char b){
     return std::string(1,a)+std::string(1,b);
 }
 ```
 But that is __NOT__ a specialization. Notice that in the "general" version the return value is the same type as the input parameters wich is not the case for our specialization. The compiler will give an error. You can check it [here](https://godbolt.org/z/GKG57W).

 We can accomplish our aim by using ```auto``` as the return value, this way both versions will have the same signature.
 ```cpp
template<typename T>
auto add(T a,T b){
    return a+b;
}
template<>
auto add<char>(char a,char b){
    //return std::string({a,b})//different way
    return std::string(1,a)+std::string(1,b);
}
 ```
 You can try it [here](https://godbolt.org/z/569Yfn)

 # Algorithms in STL
The standard template library STL defines a set of general purpose containers and algorithms. You are almost always advised to use those instead of writing your own. Furthermore, since C++17, most of the algorithms can take advantage of parallelisation. In this section we look at a few examples. Most of these algorithms need the <algorithm> or <numeric> header and they are defined over a range [start Iterator, end Iterator).

## STL containers and iterators
Iterators are generalization of pointers and present a common interface to all STL containers and algorithms. For an array a pointer is sufficient since the elements of an array form a __contiguous__ location in memory. What if the elements are not stored contiguously? Since every container stores the elements differently, it implements its own methods to _iterate_ over its elements. This has the added value that the user does not need to know the internal workings of the container in order to be able to use it.
 Given a container ```c``` an iterator ```itr``` points at an element stored in ```c```. Therefore dereferencing an iterator ```*itr``` will return the element itself. Also iterators can be incremented and decremented like pointers: ```itr++``` and ```itr--```. Furthermore, every container ```c``` has a ```begin``` and ```end()``` method. 

```cpp
#include <vector>
#include <iostream>
int main(){
  std::vector<std::string> sv;
  sv.push_back("one");
  sv.push_back("two");
  for(auto itr=sv.begin();itr!=sv.end();itr++){
      std::cout<<(*itr)<<std::endl;
  }
}  
```

The auto keyword is useful since otherwise we have to write down the long type of the iterator: (since it is an iterator to container of type ```std::vector<std::string>```).

```cpp
std::vector<std::string>::iterator itr;
```
vectors,unlike ```std::list```, are required to store their content at contiguous locations. Therefore they support
random access to elements and thus define the index operator

```cpp
std::vector<int> iv;
iv.push_back(1);
iv.push_back(2);
for(int i=0;i<iv.size();i++)
  iv[i]=i;
```

Since vectors are required by the c++ standard to use contiguous memory it is best to add and remove(as opposed to change) from the end of a vector. While we will deal mostly with ```std::vector``` there are many other types of container/container adaptor  in the STL such as ```std::list```, ```std::stack```, ```std::map```,etc.
## Algorithms
In this section we explore a few algorithms provided by the STL.

1. ```std::count``` and ```std::count_if``` ([signature](https://en.cppreference.com/w/cpp/algorithm/count))

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
bool is_even(int x){
    return x%2==0;
}
struct is_odd{
 bool operator()(int x){
     return x%2!=0;
 }
};
int main(){
    std::vector<int> v {7,2,4,2,2,8,2};
    int r=std::count(v.begin(),v.end(),2);
    std::cout<<" number of 2's is "<<r<<"\n";
    int even=std::count_if(v.begin(),v.end(),is_even);
    int odd=std::count_if(v.begin(),v.end(),is_odd());
    std::cout<<" number of even is "<<even<<" and odd is "<<odd<<"\n";
}
```
As you can see count_if takes a unary predicate as a third parameter and this can be  either a function pointer or a function object. Recall, a function object is one that defines an operator().
[click here to run](https://godbolt.org/z/nY35hr)

1. ```std::find``` and ```std::find_if``` [reference](https://en.cppreference.com/w/cpp/algorithm/find). Find the __first__ occurrence of an object in a container and returns an iterator pointing to it.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
int main(){
  std::vector<int> v {3,7,9,1,1,8,19,2,1,4};
  auto first=std::find(v.begin(),v.end(),1);
  /* print all the elements from first
  * occurrence of the value 1 to the end 
  */
  for(auto itr=first;itr!=v.end();++itr)
    std::cout<<*itr<<",";
  std::cout<<"\n";
  first=std::find_if(v.begin(),v.end(),
            [](int x){return x%2==0;});
  /* all the elements from the first
   * occurrence of an even number to 
   * the end
   */
  for(auto itr=first;itr!=v.end();++itr)
    std::cout<<*itr<<",";
  std::cout<<"\n";
}
```
You can try it [here](https://godbolt.org/z/75jKq7)


1. ```std::remove```, ```std::remove_if``` and ```std::erase```

The STL functions ```std::remove``` and ```std::remove_if``` do __not__ remove anything. They just partition the input range into a left part which contains the original objects, in the same order, where the desired object is removed, and a right part which contains "non useful" objects: either the one to be removed or additional copies of the already existing objects.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main(){
    std::vector<int> v {1,2,3,4,2,5,2,6};
    auto new_end=std::remove(v.begin(),v.end(),2);
    for(auto itr=v.begin();itr!=new_end;++itr)
      std::cout<<*itr<<",";
    std::cout<<"\n";
    /* "non useful" part */
    for(auto itr=new_end;itr!=v.end();++itr)
     std::cout<<*itr<<",";
    std::cout<<"\n";
}
```
As you might have guessed ```std::remove_if``` works the same way but using a predicate instead of a value.
These two functions are useful, especially, for the remove-erase idiom. Once we have the "useful" range with the desired object removed from it we can actually erase it.
```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main(){
    std::vector<int> v {1,2,3,4,2,5,2,6};
    auto new_end=std::remove(v.begin(),v.end(),2);
    v.erase(new_end,v.end());
    /* the vector contains only 
     * the "useful" part
     */
    for(auto x:v)
     std::cout<<x<<",";
    std::cout<<"\n";
}
```
You can try it [here](https://godbolt.org/z/6h9M9T)

In many situation we need to print all the values in a container using a range-based for loop and auto.
We can type a little less if we defined an operator ```<<``` that can handle containers.
```cpp
template<typename Ostream,typename Container>
Ostream& operator<<(Ostream& os,Container& c) {
	for (auto x : c)
		os<< x << ",";
	return os;
}
```
The above works well with all sorts of containers, except with strings since will will print a string
as characters separated by commas. We can fix this by using template specialization
```cpp
template <>
std::ostream& operator<< <std::ostream, std::string>
			(std::ostream& os, std::string& s) {
	/* cannot use os<<s because we enter infinite recursion */
	os.write(s.c_str(), s.size());
	return os;
}

```
## substrings and subranges

The ```std::string``` class has a convenient member ```std::string::find``` which returns the position of
the first character of the substring in the string if found, and ```std::string::npos``` otherwise. 
A general "substring" finding method is ```std::search``` which will find a "subrange" in any container,
including strings.
```cpp
#include <iostream>
#include <string>
int main(){
    std::string s {"First Middle Last Middle"};
    std::string sub {"Middle"};
    auto pos=s.find(sub);
    if(pos!=std::string::npos)std::cout<<"found at "<<pos<<"\n";
    else std::cout<<"not found\n";
    std::vector<int> v{4,2,7,3,1,5};
    std::vector<int> needle {7,3,1};
    /* returns an iterator to the beginning of the match
    * or v.end() if no match is found
    */
    auto itr=std::search(v.begin(),v.end(),needle.begin(),needle.end());
}

```

### Lambda expressions

As we saw, many algorithms, especially in STL, take a __callable__ object as a parameter.  Lambda expression are a convenient way to define such a callable object. We rewrite the previous example using lambda expressions.

```cpp
#include <iostream>
#include <algorithm>
#include <vector>

int main(){
    std::vector<int> v {7,2,4,2,2,8,2};
    int r=std::count(v.begin(),v.end(),2);
    std::cout<<" number of 2's is "<<r<<"\n";
    int even=std::count_if(v.begin(),v.end(),
        [](int x){return x%2==0;});
    int odd=std::count_if(v.begin(),v.end(),
        [](int x){return x%2!=0;});
            std::cout<<" number of even is "<<even<<" and odd is "<<odd<<"\n";
    //counting the number of 2's in a different way
    // just to introduce capture
    int val=2;
    std::count_if(v.begin(),v.end(),
        [val](int x){return x==val;});
}
```
[click here to run](https://godbolt.org/z/MsE1jY)
The brackets "[]" introduces the lambda expression, sometimes called the capture list. The "()" are the parameters passed to the expression, very similar to function arguments. Finally, between "{" and "}" is the body.

Note the __capture__ of the variable _val_. We could have used "[&val]" to capture it by reference. If we want to capture all the variables we use "[=]" and by reference "[&]".

##


