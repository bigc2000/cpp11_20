# cpp11_20
cxx practice 

##　C++ 11 常用特性 
整体上分为语法层面和库的改进。语法层面的改动在C11是一个特别大的提升。而库在11,17和20中也加了非常多的特性。
### 初始化列表
使用大括号，避免（）带来的歧义性
``` c++
  std:vector<int> a = {1,2,3};
  struct A {
    std::string name;
    A(std::string name) : name{"test"} {}
  };

而原来使用()会有如下歧义
int b();//这个是定义函数
int b(3);//这个是初始化 等价于 int b=3; int b{3};
```
这种初始化特别有利于vector，map这种容器的初始化 ，比如
``` c++
  std::map<int, int> test{{3, 4},{4, 5}};
```
初始化列表同时通过库来支持
template<class T> class initializer_list; 在<initializer_list>中定义

### 类成员初始化话
``` c++
  struct A {
    int id = 1;
    std::string name = "alpha";
    std::string info{"none"};
  };
```
这已经是类似java方式一样，简化了原来构造函数指定默认值的缺陷。
更早的98/0x 类成员仅仅支持static const变量，如：
``` c++
  struct A {
    const static int VAR = 3;
    int id = 1; // 98/0x 下这个是错误的 
  } 
```
多采用初始化赋初值，可以极大减少bug引入

### auto 和 decltype 静态类型推导
用于for循环等内部特别合适,需要注意可选择使用const 还有是否使用引用符号& 
``` c++
  std::map<int, int> test{{3, 4},
                          {4, 5}};
  for (const auto &it:test) { //类型推导为 *std::map<int,int>::const_iterator
    std::cout << it.first << " " << it.second << std::endl;
  }
  for (auto it = test.begin(); it != test.end(); it++) {
    std::cout << it->first << " " << it->second << std::endl;
  }
```
注，是静态类型编译期间推导，因此，不能使用运行期间的类型。
```
template<typename T>
auto add(T x, t y) -> decltype(x + y) {
  return x + y;
}
```
上述列子，如果要在98/0x中实现，是一个非常麻烦需要重载很多方法，而且还不能应对那些新的类型。大大简化了代码编写量。
这个在opencv库中的早期代码中可以看到，为了实现乘法，32位float相乘，double相乘都重新写一遍。

+ auto优缺点
-  使用类型推导可以避免一些不起眼的错误
```c++
string str = .....;
unsigned n = str.find("ABC");
if (n != string::npos)
```
这里n本来应该使用size_t类型，但不经意使用的uint32类型，这个在多数情况下是正确的，但在64位是肯能出错的（超出范围）。
该用auto就可以避免该错误
auto n = str.find("ABC");
- 减少代码 特别是循环中，容器迭代器
- 过多的auto 让代码维护困难
因为auto的时候难看出原始类型，特别是在模板类的函数中，嵌套多层。应局限于局部变量，且类型明确含义下使用。auto非常类似动态语言，如早期js的程序使用var。

### using 用法
+ Inherited constructors 继承构造函数
``` c++
  struct B {
    void f(double);
  };

  struct D : B {
    void f(int);
  };

  B b;   b.f(4.5);  // fine
  D d;   d.f(4.5);  // surprise: calls f(int) with argument 4
    // 在C++11中，使用using来解决如下
  class Derived : public Base { 
  public: 
    using Base::f;    // lift Base's f into Derived's scope -- works in C++98
    void f(char);     // provide a new f 
    void f(int);      // prefer this f to Base::f(int) 

    using Base::Base; // lift Base constructors Derived's scope -- C++11 only
    Derived(char);    // provide a new constructor 
    Derived(int);     // prefer this constructor to Base::Base(int) 
    // ...
  }; 
```
+ 替代typedef （template alias 模板别名）
```c++
  template<class T>
  using Vec = std::vector<T,alloc<T>>;  // 使用using引用一个类型， 
  Vec<int> fib = { 1, 2, 3, 5, 8, 13 }; // allocates elements using My_alloc
  vector<int,alloc<int>> verbose = fib; // verbose 和 fib 是同一个类型 
```
```c++
typedef void (*PFD)(double);  // C style ，定义PFD为返回void带double参数的函数指针
using PF = void (*)(double);  // using plus C-style type  使用using 而不是typedef
using P = [](double)->void;  // using plus suffix return type 使用using 的lambda
```
但使用decltype容易有歧义的
```c++
int x = 1;
decltype(auto) f() { return x; }  // return type is int, same as decltype(x)
decltype(auto) f() { return(x); } // return type is int&, same as decltype((x)) 这里返回的是引用类型
```


### 函数的default 和 delete

default和delete 主要用于构造函数和析构函数
default即缺省，delete，即删除（不可用，编译期间使用会出错），早起实现单例不可copy的方式是通过声明一个private的copy构造函数和一个赋值构造函数
现在都可以改成delete如：
```c++
 struct Singleton {
    std::string name{"singleton"};
    static Singleton &instance() {
      static Singleton t{};
      return t;
    }
    Singleton(const Singleton &) = delete;
    Singleton operator=(const Singleton &) = delete;
  private:
    Singleton() {}
  };
```
### noexcpt
函数声明表明不会抛出异常，这类似于引入java中的检查异常概念;但如果和try catch混用还是容易让人迷糊

``` c++
// function with an exception specification and a function try block
int f2(std::string str) noexcept try
{ 
    return std::stoi(str);
}
catch(const std::exception& e)
{
    std::cerr << "stoi() failed!\n";
    return 0;
}

```

### 变长模板参数（Parameter pack的一种特例）
C语言中使用...作为函数变长参数，最常见的就是printf函数，其原型是
int printf (const char * fmt, ...)
这个在cpp98中，如果想通过cout取代printf，并实现类似的功能是行不通的。
变长参数实际应用广泛，如在java中通过转换为数组来支持如spring框架中 void run(String ... args)
C语言定义了变长参数的宏__VA_ARGS__
在c++11中，模板参数允许变长，可以扩展类似printf功能
变长参数，必须放在最后面，这与C语言的变长参数函数类似
``` c++
template<class ... Types> struct Tuple {};
Tuple<> t0;           // Types contains no arguments
Tuple<int> t1;        // Types contains one argument: int
Tuple<int, float> t2; // Types contains two arguments: int and float
Tuple<0> error;       // error: 0 is not a type

//实际上#include<tuple>真的tuple类实现要复杂的多

//基于变长参数模板扩展printf，允许递归定义
void tprintf(const char* format) // base function
{
    std::cout << format;
}
 
template<typename T, typename... Targs>
void tprintf(const char* format, T value, Targs... Fargs) // recursive variadic function
{
  for ( ; *format != '\0'; format++ ) {
    if ( *format == '%' ) {
       std::cout << value;
         tprintf(format+1, Fargs...); // recursive call
         return;
      }
    std::cout << *format;
  }
}

```
+ 模板参数列表
这个用法是在模板参数列表中的Pack expanision 展开。
``` c++
template<typename... T> struct value_holder
{
  template<T... Values> // expands to a non-type template parameter 
  struct apply { };     // list, such as <int, char, int(&)[5]>
};
```
+  sizeof... 运算符
``` c++
template<class... Types>
struct count {
  static const std::size_t value = sizeof...(Types);
};
```
+ 类构造初始化
```
template<class... Mixins>
class X : public Mixins... {
  public:
    X(const Mixins&... mixins) : Mixins(mixins)... { }
};
```
+ 可以与右值联合使用
``` c++
template<class... Types>
void func(Types && ...types){
 // ....
}
```
+ 与lambda结合
```
template<class ...Args>
void f(Args... args) {
  auto lm = [&, args...] { return g(args...); };
  lm();
}
```

### lambda

lambda表达式是一个prvalue表达式，具有唯一，没命名的类型，也成为闭包
``` c++
struct S2 { void f(int i); };
void S2::f(int i)
{
    [&]{};          // OK: by-reference capture default
    [&, i]{};       // OK: by-reference capture, except i is captured by copy
    [&, &i] {};     // Error: by-reference capture when by-reference is the default
    [&, this] {};   // OK, equivalent to [&]
    [&, this, i]{}; // OK, equivalent to [&, i]
}
注[=]这种捕获方式，在cpp20中修改
```

### nullptr 和nullptr_t
nullptr被定义为一个类型，主要是取代NULL这个使用方式。原有NULL到底是一个类型还是0或者是(void*)0?
有不同的定义，比如常见的#define NULL 0，但是NULL其实不是0也不是(void*)0 
nullptr_t表示的是nullptr的类型，等价于typedef decltype(nullptr) nullptr_t;

### override和final
override 类似java的默认覆盖属性，用于子类实现或者替代父类实现
final 则是禁止继承类修改方法的实现

### union关键字的扩展



### 用户自定义字面量类型
98/0x都只允许默认定义的如1.2F，3L，4.0d这种，如果需要表示代度量的常量
``` c++
10km  //里程
1.2i //复数
100ml //容量
8MB //存储容量
```
### 声明式属性注解 (attributes)
使用双[[]]来实现 如noreturn表示函数不返回，carries_dependency表示还需要优化：
``` c++

void f [[ noreturn ]] ()  // f() will never return
{
  throw "error";   // OK
}

struct foo* f [[carries_dependency]] (int i);  // hint to optimizer
int* g(int* x, int* y [[carries_dependency]]);
```
当然，这种属性注解可以增强语言方言特性，但目前还没有很好的应用。比如openmp多核 #pragma omp parallel for指令，如果可以采用下面方式（目前还没有）
``` c++
for [[omp::parallel()]] (int i=0; i<v.size(); ++i) {
  // ...
}
```

### c20的一些特性

## 更好的库,

### chrono 日期和时间的库
这个主要从boost借鉴过来的。
### thread
+ 启动
+ 转交所有权
+ detach和join
+ thread_local
### 智能指针
废弃了auto_ptr，这个主要是没有很好的区分共享（基于引用计数实现）和唯一
### enable_if 和  static_assert
其实，static_assert是关键字。用于静态（编译期间）断言


### 参考文献
 1. [c++11 FAQ](https://www.stroustrup.com/C++11FAQ.html)
 2.  


