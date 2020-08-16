---
title: Barton-Nackman 技巧
date: 2019-12-24 18:37:47
tags: 
    - C++
    - Boost
categories: C++语言
---

### 1. 简介  

`Barton-Nackman`技巧是通过传递自己本身作为模板参数, 在模板中实现一些依赖于派生类的方法,从而减少了派生类的代码量
<!-- more -->

### 2. 代码示意

``` cpp
template<typename Derived> class less_than{
public:
    //通过小于符号定义其他符号,例如大于,大于等于等
    friend bool operator >(const Derived& lhs, const Derived& rhs){
        return rhs < lhs;    
    }

    friend bool operator >=(const Derived& lhs, const Derived& rhs){
        return !(lhs < rhs);
    }
    
};
//这个类需要自己定义小于符号,然后通过继承less_than,可以自动生成大于符号
class SomeClass : public less_than<SomeClass>{
public:
    int value_;

    SomeClass(int value) : value_(value) { }
    friend bool operator <(const SomeClass& lhs, const SomeClass& rhs){
        return lhs.value_ < rhs.value_;
    }  
};
```
测试代码:

``` cpp
    SomeClass a(1);
    SomeClass b(2);
    (a < b);
    (a > b); //work well
    (a >= b); //work well
```

### 3. 如何使用

`boost/operators.hpp`中提供了很多这样的模板类,包含该头文件,然后自己类继承即可.

``` cpp
less_than_comparable : 派生类提供小于,自动生成 > , >= , <=
equality_comparable : 派生类提供等于, 自动生成 !=

 class MyOwnClass : public less_than_comparable<MyOwnClass>{
    public:
        //实现小于符号,该类会自动生成 <=, >, >=
        friend bool operator <(const SomeClass& lhs, const SomeClass& rhs);
 };


```