# C++模板特化与偏特化

## 前言

说到C++模板，这个已经不是什么新东西了，自己在实际开发中也用过；对于C++模板特化和偏特化，对于别人来说，已经不是什么新东西了，但是对于我来说，的确是我的盲区，那天在群里讨论这个问题，自己对于这部分确实没有掌握，又联想到在《STL源码剖析》一书中，对于此也是有着介绍。所以，今天就对此进行详细的总结，以备后忘。

## C++模板

说到C++模板特化与偏特化，就不得不简要的先说说C++中的模板。我们都知道，强类型的程序设计迫使我们为逻辑结构相同而具体数据类型不同的对象编写模式一致的代码，而无法抽取其中的共性，这样显然不利于程序的扩充和维护。C++模板就应运而生。C++的模板提供了对逻辑结构相同的数据对象通用行为的定义。这些模板运算对象的类型不是实际的数据类型，而是一种参数化的类型。C++中的模板分为类模板和函数模板。

类模板如下：

```cpp
#include <iostream>
using namespace std;
 
template <class T>
class TClass
{
public:
     // TClass的成员函数
 
private:
     T DateMember;
};
```

函数模板如下：

```cpp
template <class T>
T Max(const T a, const T b)
{
     return  a > b ? a : b;
}
```



## 模板特化

有时为了需要，针对特定的类型，需要对模板进行特化，也就是所谓的特殊处理。

特化包括全特化和偏特化，全特化也叫简称特化，所以说特化的时候意思就是全特化。特化就是对所有的模板参数指定一个特定的类型，偏特化就是对部分模板参数指定特定的类型。比如有以下的一段代码：

```cpp
#include <iostream>
using namespace std;
 
template <class T>
class TClass
{
public:
     bool Equal(const T& arg, const T& arg1);
};
 
template <class T>
bool TClass<T>::Equal(const T& arg, const T& arg1)
{
     return (arg == arg1);
}
 
int main()
{
     TClass<int> obj;
     cout<<obj.Equal(2, 2)<<endl;
     cout<<obj.Equal(2, 4)<<endl;
}
```

类里面就包括一个Equal方法，用来比较两个参数是否相等；上面的代码运行没有任何问题；但是，你有没有想过，在实际开发中是万万不能这样写的，对于float类型或者double的参数，绝对不能直接使用“==”符号进行判断。所以，对于float或者double类型，我们需要进行特殊处理，处理如下：

```cpp
#include <iostream>
using namespace std;
 
template <class T>
class Compare
{
public:
     bool IsEqual(const T& arg, const T& arg1);
};
 
// 已经不具有template的意思了，已经明确为float了
template <>
class Compare<float>
{
public:
     bool IsEqual(const float& arg, const float& arg1);
};
 
// 已经不具有template的意思了，已经明确为double了
template <>
class Compare<double>
{
public:
     bool IsEqual(const double& arg, const double& arg1);
};
 
template <class T>
bool Compare<T>::IsEqual(const T& arg, const T& arg1)
{
     cout<<"Call Compare<T>::IsEqual"<<endl;
     return (arg == arg1);
}
 
bool Compare<float>::IsEqual(const float& arg, const float& arg1)
{
     cout<<"Call Compare<float>::IsEqual"<<endl;
     return (abs(arg - arg1) < 10e-3);
}
 
bool Compare<double>::IsEqual(const double& arg, const double& arg1)
{
     cout<<"Call Compare<double>::IsEqual"<<endl;
     return (abs(arg - arg1) < 10e-6);
}
 
int main()
{
     Compare<int> obj;
     Compare<float> obj1;
     Compare<double> obj2;
     cout<<obj.IsEqual(2, 2)<<endl;
     cout<<obj1.IsEqual(2.003, 2.002)<<endl;
     cout<<obj2.IsEqual(3.000002, 3.0000021)<<endl;
}
```

从语法上看，全特化的时候，template后面尖括号里面的模板参数列表必须是空的，表示该特化版本没有模板参数，全部都被特化了。

## 模板偏特化

上面对模板的特化进行了总结。那模板的偏特化呢？所谓的偏特化是指提供另一份template定义式，而其本身仍为templatized；也就是说，针对template参数更进一步的条件限制所设计出来的一个特化版本。这种偏特化的应用在STL中是随处可见的。比如：

```cpp
emplate <class _Iterator>
struct iterator_traits
{
     typedef typename _Iterator::iterator_category iterator_category;
     typedef typename _Iterator::value_type        value_type;
     typedef typename _Iterator::difference_type   difference_type;
     typedef typename _Iterator::pointer           pointer;
     typedef typename _Iterator::reference         reference;
};
 
// specialize for _Tp*
template <class _Tp>
struct iterator_traits<_Tp*> 
{
     typedef random_access_iterator_tag iterator_category;
     typedef _Tp                         value_type;
     typedef ptrdiff_t                   difference_type;
     typedef _Tp*                        pointer;
     typedef _Tp&                        reference;
};
 
// specialize for const _Tp*
template <class _Tp>
struct iterator_traits<const _Tp*> 
{
     typedef random_access_iterator_tag iterator_category;
     typedef _Tp                         value_type;
     typedef ptrdiff_t                   difference_type;
     typedef const _Tp*                  pointer;
     typedef const _Tp&                  reference;
};
```

看到了么？这就是模板偏特化，与模板特化的区别在于，**模板特化以后，实际上其本身已经不是templatized，而偏特化，仍然带有templatized**。我们来看一个实际的例子：

```cpp
#include <iostream>
using namespace std;
 
// 一般化设计
template <class T, class T1>
class TestClass
{
public:
     TestClass()
     {
          cout<<"T, T1"<<endl;
     }
};
 
// 针对普通指针的偏特化设计
template <class T, class T1>
class TestClass<T*, T1*>
{
public:
     TestClass()
     {
          cout<<"T*, T1*"<<endl;
     }
};
 
// 针对const指针的偏特化设计
template <class T, class T1>
class TestClass<const T*, T1*>
{
public:
     TestClass()
     {
          cout<<"const T*, T1*"<<endl;
     }
};
 
int main()
{
     TestClass<int, char> obj;
     TestClass<int *, char *> obj1;
     TestClass<const int *, char *> obj2;
 
     return 0;
}
```

从语法上看，偏特化的时候，template后面的尖括号里面的模板参数列表必须列出未特化的模板参数。同时在类后面要全部列出模板参数，同时指定特化的类型

## 特化与偏特化的调用顺序

对于模板、模板的特化和模板的偏特化都存在的情况下，编译器在编译阶段进行匹配时，是如何抉择的呢？从哲学的角度来说，应该先照顾最特殊的，然后才是次特殊的，最后才是最普通的。编译器进行抉择也是尊从的这个道理。从上面的例子中，我们也可以看的出来，这就就不再举例说明。