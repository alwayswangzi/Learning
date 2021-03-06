# 动态绑定是如何实现的？

## 1. 定义

如果将基类的成员函数声明为virtual的，然后用指向派生类对象的基类指针或者引用来调用该成员函数，那么程序会在运行时选择该派生类的函数而不是基类的函数，这种特性成为运行时绑定（动态绑定、晚绑定）。

## 2. 功能

主要实现接口复用。

## 3. 实现机制

首先，每一个含有虚函数的类叫做多态类，编译器会给每个多态类至少创建一个虚函数表，它其实是一个函数指针数组，存放着这个类所有的虚函数地址以及该类的类型信息，其中也包括哪些继承但未被改写（overwrite）的虚函数。

其次，每一个多态类的对象都有一个隐含的指针成员：虚函指针vptr，它指向所属类中的vtable

## 4.特点

由于多态类在运行时才进行函数寻址，因此编译时无法进行函数的静态类型检查，因此为了解决函数调用的问题，编译器采用了最简单粗暴的方法：**派生类中定义的函数名将义无反顾地隐藏掉基类中任何同名的函数（不管它们参数列表是否相同）**。

因此，派生类虚函数要达成运行时动态绑定的效果，必须和基类的函数名、参数列表完全相同，否则仅仅是对基类虚函数的隐藏。