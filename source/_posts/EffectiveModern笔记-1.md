---
title: EffectiveModernC++笔记1-类型推导
date: 2022-06-14 20:26:21
tags: C++
---


## Deducing Type

类型推导出现在函数模板调用处，大多数auto出现的地方，在decltype表达式出现的地方。

解释了模板类型推导如何进行工作，auto如何依赖类型推到的，解释了该如何强制编译器使类型推导的结果可视。

### 条款1：理解模板类型推导

考虑函数模板

```c++
template<typename T>
void f(ParamType param);

f(expr);
```
<!--more-->
如果ParamType为`const T&`，传入expr为int，那么T被推导为int，ParamType被推导为`const int&`，很自然的期望传入实参和T类型相同，但有的情况并非如此。

情景一：ParamType为指针或引用，但不是通用引用

类型推导的过程：1、如果expr类型是个引用，忽略引用部分；2、然后expr类型与ParamType进行模式匹配来决定T

```c++
template<typename T>
void f(T& param);
int x = 27; //T int, param int&
const int cx = x; //T const int, param const int&
const int& rx = x; //T const int, param const int&
```

const对象的不可改变性被保留，也就是说形参是reference-to-const。因此将const对象传递给T&类型为形参的模板是安全的，对象的常量性会被保留为T的一部分。

如果将ParamType改为const T&，情况会不同，此时const不会被推导为T的一部分

```c++
template<typename T>
void f(const T& param); // param是reference-to-const
int x = 27; //T int, param int&
const int cx = x; //T  int, param const int&
const int& rx = x; //T  int, param const int&
```

如果param是指针（or指向const的指针），本质也一样

```c++
template<typename T>
void f(T* param); // param是reference-to-const
int x = 27; //T int, param int
const int* px = &x;
f(&x); //T int, param int* 
f(px); //T int, param int*
```



情景二：ParamType为通用引用

形参被声明为右值引用一样，如果expr是左值，T和ParamType都会被推导为左值引用，不同寻常的是这是唯一一种T被推导为引用的情况，如果expr是右值，即是情景一的正常推导规则。

```c++
template<typename T>
void f(T&& param);
int x = 27;
const int cx = x;
const int& rx = cx;
f(x);  //expr是左值，T被推导int&， param 类型为int&
f(cx); //expr是左值，T是const int&, param const int&
f(rx); //expr是左值，T是const int&, param const int&
f(27); //expr是右值，T是int， param int&&
```

当通用引用被使用，类型推导会区分左值实参和右值实参，非通用引用时不会区分。

情景三：ParamType不是指针也不是引用

ParamType不是引用和指针，用传值方式处理。

```c++
template<typename T>
void f(T param);
int x = 27;
const int cx = x;
const int& rx = x;
f(x); //T param都是int
f(cx);//T param都是int
f(rx);//T param都是int
```

类型推导时：1、如果expr是引用，忽略引用部分；2、如果忽略引用后，还是个const，则忽略const，如果是volatile，忽略。

只有传值给形参时才会忽略const，对于reference-to-const和pointer-to-const的形参，const都会被保留。如果expr是const指针，指向const对象，通过传值传递给param，

```c++
template<typename T>
void f(T param);
const char * const ptr = " ads";
f(ptr); 
```

ptr自身的值会被传递给param，自身的常量性会被忽略，所以param类型是const char*，即可变指针指向const字符串，即传入后的ptr产生一个拷贝，自身从不可变变成可变指针了。

#### 数组指针

如果将一个数组传值给一个模板，数组类型会被推导为一个退化的指针类型，

```c++
template<typename T>
void f(T param);
const char name[13];
f(name); //T类型const char *
```

但当用了引用

```c++
template<typename T>
void f(T& param);
```

T会被推导为真正的数组，包括了数组的大小，T被推导为const char[13]，param类型为const char(&)[13]。

可声明指向数组的引用，可以创建一个模板函数来推导数组大小

```c++
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;
}
```

如现成的array

```c++
int keyVals[] = {1,2,4,5,6,7};
std::array<int,arraySize(keyVals)> mappedVals;
```

#### 函数实参

不止数组会退化为指针，函数类型也会退化为指针

```c++
void Func(int, double);
template<typename T>
void f1(T param);

template<typename T>
void f2(T & param);

f1(Func); // param类型推导为指向函数的指针，类型void(*)(int,double)
f2(Func); //param为指向函数的引用，类型void(&)(int,double)
```



### 条款二：理解auto类型推导

auto类型与模板类型推导有一个直接映射关系，通过一个非常规范化和系统化的转换流程来转换彼此。

当一个变量用auto来声明，auto扮演了模板中T的角色，变量的类型说明符扮演ParamType角色。

```c++
auto x = 27;
```

这个类型说明符说的是auto自己

```c++
const auto cx = x;
```

这个类型说明符说的是const auto

```c++
const auto& rx = x;
```

这个类型说明符是const auto& 

进行x、cx、rx类型推导时，编译器的行为看起来像认为这些声明都有一个模板，然后用合适的初始化表达式进行调用

```c++
template<typename T>
void func_for_x(T param);
func_for_x(27);

template<typename T> //概念化的模板
void func_for_cx(const T param);
func_for_cx(x); // 概念化的调用，param的推导类型时cx的类型

template<typename T>
void func_for_rx(const T& param);
func_for_rx(x);
```



对于特殊的情景二：类型说明符是个通用引用

```c++
auto&& uref1 = x; // x是int左值，uref1是int&
auto&& uref2 = cx;// cx是int&，uref2是int&
auto&& uref3 = rx;// 27是int右值，uref3是int&&

```

对于数组和函数

```c++
const char name[] = "Jack";
auto arr1 = name; //arr1类型是const char*
auto& arr2 = name; //arr2类型是const char(&)[5]

void Func(int, double);
auto func1 = Func; //func1类型是void(*)(int, double)
auto& func2 = Func; //func2类型是void(&)(int, double)

```

不同点是auto声明一个变量，并用花括号进行初始化，auto类型推导会得出`std::initializer_list`的结果。

```c++
auto x = {12};
auto x1{12};//都会被推导为std::initializer_list<int>
auto xerr = {1,2,3.0}//无法推导std::initializer_list<T>中的T
```

实际上，auto x用{...}做推导时，必须被推导为std::initializer_list<T>，而必须被某种类型T实例化，意味着T也要被推导。推导落入了第二种类型推导--模板类型推到的范围。

```c++
auto x = { 11, 23, 9 };         //x的类型是std::initializer_list<int>

template<typename T>            //带有与x的声明等价的
void f(T param);                //形参声明的模板

f({ 11, 23, 9 });               //错误！不能推导出T
```

如果指定T是std::initializer_list<T>而留下未知T，模板推导可以正常工作。

```c++
template<typename T>            
void f(std::initializer_list<T> param);                

f({ 11, 23, 9 });               //T推导为T，param为std::initializer_list<int>
```

这只是c++11的情况，c++14允许auto用于函数返回值并会被推导，并且允许lambda函数中在形参声明使用auto。但实际上都是用模板类型推导的规则运行。

因此有的方法是不能通过编译的

```c++
auto createInitList()
{
	return {1,2,3};
}

std::vector<int> v;
…
auto resetV = 
    [&v](const auto& newValue){ v = newValue; };        //C++14
…
resetV({ 1, 2, 3 });            //错误！不能推导{ 1, 2, 3 }的类型
```



### 条款三：理解decltype

与模板类型推导和auto类型推导相比，decltype只是简单返回名字或表达式的类型。

C++11中`decltype`最主要用处用于声明函数模板，而这个函数的返回类型依赖于形参类型。

```c++
//c++11用法
template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}

```

上例如果不用->尾置返回类型，auto会根据模板类型推导的规则，将返回值推断为非引用类型，这是因为传值类型推导时会忽略掉引用。

C++14中可以直接用下例，decltype(auto)：auto说明符表示这个类型将会被推导，decltype说明decltype规则将会被用到这个推导过程中。

```c++
template<typename Container, typename Index>    
decltype(auto) authAndAccess(Container& c, Index i)       
{
    authenticateUser();
    return c[i];
}
```

也可用到变量的推导上：

```c++
Widget w;
const Widget& cw = w;
auto myWidget1 = cw; //推导为Widget
decltype(auto) myWidget2 = cw;// 推导为const Widget&
```

如果容器通过传引用的方式传递非常量左值引用，意味着不能给这个函数传递右值容器，右值不能被绑定到左值引用上（除非左值引用是const）

```c++
std::deque<std::string> makeStringDeque();      //工厂函数

//从makeStringDeque中获得第五个元素的拷贝并返回
auto s = authAndAccess(makeStringDeque(), 5);
```

要想支持上例的用法，需要修改声明使其支持左值和右值，重载，注意用forwad实现通用引用。

```c++
template<typename Container, typename Index>    
decltype(auto) authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

对于decltype作用到单纯变量名上时，会产生其声明类型，但对于更复杂的左值表达式，decltype会确保类型始终是左值引用。

```c++
    int x = 0;
    decltype(x) ix = x;
    ix = 3;
    std::cout << x << std::endl; // 0
    decltype((x)) rx = x;
    rx = 1111;
    std::cout << x << std::endl; // 1111
```



### 条款四：学会如何看类型推导结果

Know how to view deduced types

`std::type_info::name`并不保证返回任何有意义的东西

Boost TypeIndex库是更好的选择

```c++
template <typename T>
void f(const T &param)
{
    using boost::typeindex::type_id_with_cvr;
    using std::cout;

    //显示T
    cout << "T =     "
         << type_id_with_cvr<T>().pretty_name()
         << '\n';

    //显示param类型
    cout << "param = "
         << type_id_with_cvr<decltype(param)>().pretty_name()
         << '\n';
}

```

