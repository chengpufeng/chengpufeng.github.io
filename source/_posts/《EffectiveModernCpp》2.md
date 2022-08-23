---
title: 《EffectiveModernCpp》-2-现代C++特性
tags: C++
categories: 《EffectiveModernCpp》笔记
date: 2022-06-17 17:10:54
---


# 现代C++的一些特性

## 条款十三：优先考虑`const_iterator`而非`iterator`

<!--more-->

## 条款十四：如果函数不抛出异常请使用`noexcept`

c++11中，无条件的`noexcept`保证函数不会抛出任何异常。当一个函数不会抛出异常却没有加上noexcept，那么这个接口说明就有点差劲。

noexcept允许编译器生成更好的代码。

鼓励使用noexcept的地方：

- 移动语义
- swap
- 内存释放函数
- 析构函数

## 条款十五：尽可能使用constexpr

编译期可知的值可能被存放到只读存储空间。值是编译器可知的常量整数会出现在“需要整型常量表达式”的上下文中，包括数组大小、整数模板参数（std::array对象的长度）、枚举名的值、对齐修饰符。

const不提供constexpr所能保证的事情，因为const对象不需要编译期初始化它的值。

constexpr函数必须能在编译器值调用的时候返回编译期结果。

## 条款十六：让const成员函数线程安全



确保const成员函数线程安全，除非你确定它们永远不会再并发上下文（concurrent context）中使用

使用std::atomic变量比互斥量性能更高，但只适合操作单个变量或内存位置。

注：需要在const成员函数里修改的一些和类状态无关的数据成员，那么这个数据成员应该由mutable来修饰。

```c++
class Widget {
public:
    …
    int magicValue() const
    {
        std::lock_guard<std::mutex> guard(m);   //锁定m
        
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;
            cacheValid = true;
            return cachedValue;
        }
    }                                           //解锁m
    …

private:
    mutable std::mutex m;
    mutable int cachedValue;                    //不再用atomic
    mutable bool cacheValid{ false };           //不再用atomic
};
```

## 条款十七：理解特殊成员函数的生成

特殊成员函数指c++自己生成的函数。C++98有四个：默认构造函数、析构函数、拷贝构造函数、拷贝赋值运算符。

C++11加入了移动构造函数和移动赋值运算符。

```c++
class Widget {
public:
    …
    Widget(Widget&& rhs);               //移动构造函数
    Widget& operator=(Widget&& rhs);    //移动赋值运算符
    …
};
```

移动操作仅在需要时生成，如果生成了，就会对类的non-static数据成员执行逐成员的移动。即移动构造函数根据rhs参数里的对应成员构造出新的non-static部分。移动构造函数也移动构造基类部分，移动复制运算符也移动赋值基类部分（如果有的话）。

两个拷贝操作是独立的：声明一个不会限制编译器生成另一个。如声明了一个拷贝构造，但没声明拷贝赋值，当你用到了拷贝赋值，编译器会帮助生成拷贝赋值。

两个移动操作不是独立的：如果声明了其中一个，编译器不会生成另一个。当声明一个移动构造函数，就表明对于移动操作的实现，于编译器生成的默认的逐成员移动有区别，如果移动构造有问题，那么逐成员的移动赋值也是可能有问题的。所以声明移动构造会阻止编译器生成移动赋值，反之亦然。

如果一个类显式声明拷贝操作，编译器就不会生成移动操作。解释是，如果声明了拷贝操作，就暗示了平常的拷贝对象方法（逐成员拷贝）不适用此类，编译器就会明白如果逐成员拷贝对拷贝操作不合适，那么逐成员移动对于移动操作也是不合适的。

Rule of Three规则：如果声明了拷贝构造函数、拷贝赋值运算符或者析构函数三者之一，就应该声明其余两个。用户接管拷贝操作一般都意味着该类会对其他资源进行管理（1）如果一种资源管理需要在一个拷贝操作中完成，那么也会在另一个拷贝操作内完成（2）类的析构函数也需要参与资源的管理（释放）。通常需要管理的是内存，标准库里管理内存的类都声明了the big three。

当且仅当：类中无拷贝操作、类中无移动操作、类中无用户定义的析构，才会生成移动操作。

C++11抛弃了已声明拷贝操作或析构函数的类的拷贝操作的自动生成。如果还想自动生成，用=default

因为移动构造函数的生成条件很苛刻，所以可能会导致很隐蔽的性能问题，即拷贝代替移动，这时候将拷贝构造函数声明为=default可以解决。

