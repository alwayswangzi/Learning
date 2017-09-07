# C++ RTTI机制解析

## 1. 概念

RTTI（Run Time Type Identification）即运行时类型识别，程序能够使用基类的指针或者引用来查看该指针或引用实际指向的派生类对象类型。

## 2. RTTI机制出现的原因

为什么会出现RTTI这一机制，这和C++语言本身有关系。

### 虚函数机制的局限性

和很多其他语言一样，C++是一种静态类型语言。其数据类型是在编译期就确定的，不能在运行时更改。然而由于面向对象程序设计中多态性的要求，C++中的指针或引用(Reference)本身的类型，可能与它实际代表(指向或引用)的类型并不一致。有时我们需要将一个多态指针转换为其实际指向对象的类型，就需要知道运行时的类型信息，这就产生了运行时类型识别的要求。

和[Java](http://lib.csdn.net/base/java)相比，C++要想获得运行时类型信息，只能通过RTTI机制，并且C++最终生成的代码是直接与机器相关的。

Java中任何一个类都可以通过反射机制来获取类的基本信息（接口、父类、方法、属性、Annotation等），而且Java中还提供了一个关键字，可以在运行时判断一个类是不是另一个类的子类或者是该类的对象，Java可以生成字节码文件，再由JVM（Java虚拟机）加载运行，字节码文件中可以含有类的信息。

## 3. typeid操作符

RTTI提供了两个非常有用的操作符：typeid和dynamic_cast。

typeid操作符返回指针和引用所指的实际类型。

我们知道C++的多态性（运行时）是由虚函数实现的，对于多态性的对象，无法在程序编译阶段确定对象的类型。当类中含有虚函数时，其基类的指针就可以指向任何派生类的对象，这时就有可能不知道基类指针到底指向的是哪个对象的情况，类型的确定要在运行时利用运行时类型标识做出。

>RTTI和虚函数并非一回事，实际上虚函数的动态绑定并没有使用对象的type_info信息。

typeid函数以一个对象或者类型名作为参数，返回一个对type_info类对象的引用。

要使用typeid必须使用头文件\<typeinfo\>，type_info类简单的源代码结构为：

```cpp
class type_info {  
public:  
    //析构函数  
    _CRTIMP virtual ~type_info();  
    //重载的==操作符  
    _CRTIMP int operator==(const type_info& rhs) const;  
    //重载的!=操作符  
    _CRTIMP int operator!=(const type_info& rhs) const;
  	//用于type_info对象之间的排序算法
    _CRTIMP int before(const type_info& rhs) const;  
    //返回类的名字  
    _CRTIMP const char* name() const;
  	//返回类名称的编码字符串
    _CRTIMP const char* raw_name() const;  
private:  
    //各种存储数据成员  
    void *_m_data;  
    char _m_d_name[1];  
    //将拷贝构造函数与赋值构造函数设为了私有  
    type_info(const type_info& rhs);  
    type_info& operator=(const type_info& rhs);  
};  
```

因为type_info类的复制构造函数和赋值运算符都是私有的，所以不允许用户自已创建type_info的类。唯一要使用type_info类的方法就是使用typeid函数。

typeid函数的主要作用就是让用户知道当前的变量是什么类型的，比如使用typeid(a).name()就能知道变量a是什么类型的。typeid()函数的返回类型为typeinfo类型的引用。

typeid函数是type_info类的一个引用对象，可以访问type_info类的成员。但因为不能创建type_info类的对象，而typeid又必须反回一个类型为type_info类型的对象的引用，所以怎样在typeid函数中创建一个type_info类的对象以便让函数反回type_info类对象的引用就成了问题。这可能是把typid函数声明为了type_info类的友元函数来实现的，默认构造函数并不能阻止该类的友元函数创建该类的对象。所以typeid函数如果是友元的话就可以访问type_info类的私有成员，从而可以创建type_info类的对象，从而可以创建返回类型为type_info类的引用。

例如：

```cpp
class A{
private:
	A(){}
	A(const A&){}
	A& operator = (const A&){}
	friend A& f(); 
};
```

函数f()是类A的友元，所以在函数f()中可以创建类A的对象。同时为了实现函数f()反回的对象类型是A的引用，就必须在函数f中创建一个类A的对象以作为函数f的反回值，比如函数f可以这样定义:

```cpp
A &f()
{
	A m_a;
	return m_a;
}
```

## 4. dynamic_cast运算符

可以看出typeid()不具备可扩展性，因为它只能返回一个对象的确切类型而不能获得其基类的类型。

因此RTTI还引入了另一个运算符：dynamic_cast，该转换符用于将一个指向派生类的基类指针或引用安全的转换为派生类的指针或引用。其表达式为dynamic_cast\<dest_type\> (src)，其中的类型是指把表达式要转换成的目标类型。

比如含有虚函数的基类B和从基类B派生出的派生类D，则

```cpp
B *pb; 
D *pd, md; 
pb = &md; 
pd = dynamic<D*>(pb);
```

最后一条语句表示把指向派生类D的基类指针pb转换为派生类D的指针，然后将这个指针赋给派生类D的指针pd，有人可能会觉得这样做没有意义，既然指针pd要指向派生类为什么不`pd =& md;`样做更直接呢？

因为有些时候我们需要强制转换，比如如果指向派生类的基类指针B想访问派生类D中的除虚函数之外的成员时就需要把该指针转换为指向派生类D的指针，以达到访问派生类D中特有的成员的目的，比如派生类D中含有特有的成员函数g()，这时可以这样来访问该成员`dynamic_cast<D*>(pb)->g();`因为dynamic_cast转换后的结果是一个指向派生类的指针，所以可以这样访问派生类中特有的成员。但是该语句不影响原来的指针的类型，即基类指针pb仍然是指向基类B的。

> dynamic_cast转换符只能用于含有虚函数的类，dynamic_cast可以用于指针和引用，但不能转换对象。

dynamic_cast转换操作符在执行类型转换时首先将检查能否成功转换:如果能成功转换则转换之，如果转换失败:

1. 如果是指针则反回一个null值
2. 如果是转换的是引用，则抛出一个std::bad_cast异常

所以在使用dynamic_cast转换之间应使用if语句对其转换成功与否进行测试。

dynamic_cast可以实现两种方向的转换：upcast和downcast：

1. upcast把派生类的指针或引用转换为基类的指针或引用（系统可以隐式转换）
2. downcast把基类的指针或引用转换为派生类的指针或引用。如果这个基类的指针或引用确实指向一个这种派生类的对象，那么转换才会成功。

> 为了支持dynamic_cast运算符，RTTI机制必须维护一个继承树，通过遍历继承树来确定一个待转换的对象和目标类型之间是否存在is-a的关系。

