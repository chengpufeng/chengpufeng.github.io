---
title: 《EffectiveModernCpp》-3-智能指针
tags: C++
categories: 《EffectiveModernCpp》笔记
date: 2022-06-18 14:05:58
---


# 智能指针

原始指针的问题：

1. 它的声明不能指示所指的是数组还是单个对象
2. 它的声明没告诉你用完是否销毁
3. 如果决定销毁，不知道使用delete还是其他析构机制
4. delete 还是 delete[]
5. 很难保证销毁的执行路径上都执行了恰为一次的销毁操作。少一条路径就会资源泄露，多次会导致未定义行为
6. 没有办法告诉你指针是否变成悬空指针（dangling pointers），即内存中无所指之物。

<!--more-->

## 条款十八：对于独占资源使用std::unique_ptr

unique_ptr体现转悠所有权（exclusive ownership）语义。移动一个std::unique_ptr将所有权从源指针转移到目的指针。（源指针设为null）。拷贝是不允许的，因此unique_ptr是一种止咳移动类型（move-only type）。默认情况下资源析构通过对unique_ptr内原始指针调用delete实现。

常用于继承层次结构中对象的工厂函数的返回类型。

## 条款十九：对于共享资源使用std::shared_ptr

std::shared_ptr通过引用计数来确保它是否是最后一个指向某种资源的指针，关联资源并跟踪有多少shared_ptr指向该资源。

引用计数有性能问题：

- `std::shared_ptr`大小是原始指针的两倍，内部包含一个指向资源的原始指针，还包含一个指向资源引用计数值得原始指针（不是标准的实现，但标准库都这么实现）
- 引用计数内存必须动态分配。引用计数和所指对象关联，但所指对象并不知道这件事。
- 递减递增引用计数必须是原子性的。多个reader、writer可能在不同线程，如shared_ptr在一个线程析构（递减所指对象的引用计数），另一个线程指向相同对象，进行拷贝操作（递增同一个引用计数）。原子操作通常比非原子的要慢，应该假定读写它们是由开销的。

std::shared_ptr构造函数不总是增加引用计数，另一种情况是移动构造函数的存在。从另一个std::shared_ptr移动构造新的会将原来的设置为null，意味着老的std::shared_ptr，不再指向资源，同时新的std::shared_ptr指向资源。这样不需要修改引用计数值。

std::shared_ptr也支持自定义删除器。区别于std::unique_ptr，对于独占指针，删除器是智能指针的一部分，对于共享指针则不是。

```c++
auto loggingDel = [](Widget *pw)        //自定义删除器
                  {                     //（和条款18一样）
                      makeLogEntry(pw);
                      delete pw;
                  };

std::unique_ptr<Widget, decltype(loggingDel)> 
    upw(new Widget, loggingDel);

std::shared_ptr<Widget>
    spw(new Widget, loggingDel);
```

不同的std::shared_ptr<Widget>可以传入不同的删除器，具有相同的类型，都可以放在相应类型的容器中。

但自定义删除器的std::unique_ptr不行，因为删除器是其自身的一部分。

std::shared_ptr的删除器放在堆上，std::shared_ptr的创建者利用其对自定义分配器的支持能力，可以将这部分内存放在任何地方。引用计数是另一个更大的数据结构的一部分，这个数据结构叫控制块（control block）。每个shared_ptr都有相应的控制块，控制块除了引用计数值还有自定义删除器的拷贝，自定义分配器的拷贝（如果指定了的话），次级引用计数weak count。

![item19_fig1](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206181138848.png)

控制块的创建规则：

- std::make_shared总是创建一个控制块。
- 从独占指针构造出std::shared_ptr时会创建控制块。
- 从原始指针构造出std::shared_ptr时会创建控制块。从std::shared_ptr和std::weak_ptr作为构造函数实参创建共享指针时不会创建新的控制块。

会出现的问题：重复从原始指针构造std::shared_ptr会出现重复销毁，是未定义的行为，替代方案是用std::make_shared；如果必须传给std::shared_ptr构造函数原始指针，可以直接传new出来的结果，`std::shared_ptr<Widget> spwq(new Widget, loggingDel)`，出现在一定需要传入自定义删除器的情况。

另一个问题是用this指针作为std::shared_ptr构造函数实参的时候可能会导致创建多个控制块。可以用std::enable_shared_from_this<T>模板类作为基类继承，T为派生类，这种设计模式为奇异递归模板模式（*The Curiously Recurring Template Pattern*（*CRTP*）），当传入this实参时，用shared_from_this()。为了防止客户端在存在一个指向对象的共享指针之前就调用含shared_from_this的成员函数，派生类通常将其构造函数声明为private，并让客户端通过返回std::shared_ptr的工厂函数创建对象。

通常情况下使用默认删除器和默认分配器，用std::make_shared创建std::shared_ptr，产生的控制块大小为3个word。对于每个被std::shared_ptr指向的对象来说，控制块中的虚函数机制产生的开销通常只在对象销毁时承受一次。

## 条款二十：当std::shared_ptr可能悬空时用std::weak_ptr

std::weak_ptr不能解引用，不能测试是否是空值，它是std::shared_ptr的增强。

## 条款二十一：优先用std::make_unqiue和std::make_shared而非new

std::make_shared是C++11标准，std::make_unique是C++14标准。

```c++
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
	return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```

std::make_unqiue和std::make_shared是三个make函数中的两个：接收任意的多参数集合，完美转发到构造函数去动态分配一个对象，返回这个对象的指针。第三个make是std::allocate_shared，行为和std::make_shared一样，区别是第一个参数是用来动态分配内存的allocator对象。

用make的原因：

- 减少重复代码，源代码中的重复增加了编译的时间，导致目标代码冗杂。

- 与异常安全有关。内存泄露

```c++
processWidget(std::shared_ptr<Widget>(new Widget),  //潜在的资源泄漏！
              computePriority());
```

编译器不按照执行顺序生成代码，new Widget必须在std::shared_ptr的构造函数之前执行，但computePriority()可能在构造函数之前，如果这个函数异常，就会产生内存泄露。

```c++
processWidget(std::make_shared<Widget>(),   //没有潜在的资源泄漏
              computePriority());
```

- 相比于直接new的效率提升。std::make_shared允许编译器生成更小更快的代码。`std::shared_ptr<Widget> spw(new Widget)`会进行内存分配，实际上进行了两次内存分配，分别是new Widget和控制块的分配。而用make只会分配一块内存，这种优化减少了程序的静态大小。

make的限制：

- make函数不允许指定自定义删除其。
- 语法细节中，完美转发使用小括号而非花括号。当构造函数重载，有使用std::initializer_list作为参数的重载形式和不用其作为参数的重载形式，花括号创建的对象倾向于使用初始化列表作为形参的重载形式。不能用花括号初始化所指对象，必须直接用new。可以用auto类型推导从花括号初始化创建std::initializer_list对象，然后传递给make函数。

```c++
auto upv = std::make_unique<std::vector<int>>(10,20);//创建10个元素，每个值20

auto initList = {10,20}; //std::initializer_list
auto spv = std::make_shared<std::vector<int>>(initList);
```



对于std::shared_ptr和它的make函数，还有两个问题：

- 有些类重载了operator new和delete。意味着对这些类型对象的全局内存分配和释放是不合常规的。分配的内存往往会大于sizeof(Widget)，因此用make函数去创建重载了new和delete类的对象是个不好的方法。
- make函数大小和速度的优势是将控制块和所指对象放在同一块内存中。直到最后的控制块要被销毁，所匹配的对象占用的内存才会被释放。对于用到weak_ptr的情况，只要它引用了一个控制块（weak count>0），这个控制块就必须存在，即它的内存就必须保持分配。make分配的内存直到最后一个shared_ptr和weak_ptr被销毁，才会释放。如果对象类型很大，销毁对象和释放所占用内存之间可能有很大延迟。



## 条款二十二：当使用Pimpl惯用法，请在实现文件中定义特殊成员函数

Pimpl（pointer to implementation）惯用法，可以将类数据成员替换成一个指向包含具体实现的类的指针，并将主类的数据成员们移动到实现类去，这些数据成员的访问通过指针间接访问。

