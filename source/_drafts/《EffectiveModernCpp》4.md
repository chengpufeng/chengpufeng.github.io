---
title: 《EffectiveModernCpp》4
tags:
---

# 右值引用、移动语义、完美转发

## 条款二十三：理解std::move和std::forward

std

## 条款二十六：避免在通用引用上重载

对使用通用引用形参的函数，无论是独立函数还是成员函数（尤其是构造函数），进行重载都会导致一系列问题。

## 条款二十七：熟悉通用引用重载的替代方法



### 放弃重载

logAndAdd的不同功能可以改名logAndAddName和logAndAddNameIdx。但不能用在构造函数上

### 传递const T&

退回C++98，将传递通用引用替换为传递lvalue-reference-to-const，缺点是效率不高。

### 传值

在你知道要拷贝时就按值传递。

```c++
class Person {
public:
    explicit Person(std::string n)  //代替T&&构造函数，
    : name(std::move(n)) {}         //std::move的使用见条款41
  
    explicit Person(int idx)        //同之前一样
    : name(nameFromIdx(idx)) {}
    …

private:
    std::string name;
};
```

没有std::string构造函数可以接受整型参数（除了0和NULL空指针），所以所有整型变量会用int类型重载的构造函数。

### 使用tag dispatch

