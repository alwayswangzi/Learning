# 条款01：视C++为一个语言联邦

C++由主要以下“邦”构成

C：

- 区块（Blocks）
- 语句（statements）
- 预处理器（preprocessor）
- 内置数据类型（build-in data types）
- 数组（arrays）
- 指针（pointers）
- 没有模板（templates）
- 没有异常（exceptions）
- 没有重载（overloading）

面向对象（Object-Oriented）：

- 类（classes）
- 封装（encapsulation）
- 继承（inheritance）
- 多态（polymorphism）
- 虚函数（virtual function）
- 动态绑定（dynamic bonding）

泛型编程（Template）：

- ~~是个坑（误）~~

STL：

- 容器（containers）
- 迭代器（iterators）
- 算法（algorithms）
- 函数对象（function objects）




# 条款02：尽量以const、enum、inline替换#define

> 请记住：
>
> - 对于单纯常量，最好以const对象或enum替换#define
> - 对于形似函数的宏（Macro），最好改用inline函数替换#define



尽量用编译器工作代替预处理器工作。

例如：

```cpp
#define ASPECT_RATIO 1.653	//大写名称常用于宏
const double AspectRatio = 1.653;
```

优点：

- AspectRatio作为一个语言常量，会被编译器看到，加入记号表中。在debug时会看到具体的符号信息，而不是1.653这样的数字带来理解上的困惑。
- 使用常量可能会产生较少的代码。因为#define单纯的替换可能会使生成的代码产生多份1.653拷贝，而常量永远只有一份拷贝。

使用常量替换#define有两种特殊情况：

- 定义常量指针

  由于常量定义式通常放在头文件内（以便被不同源文件include），因此有必要将指针（而不是指针指向的对象）声明为const，例如声明一个常量字符串：

  `const char* ptr const = "hello!";`	//const被写了两次

- class专属常量

  为了将常量的作用域限定于class内，必须让它成员class的一个成员，而为了确保常量最多只有一份拷贝，必须让它成为一个static成员。例如：

  ```cpp
  class test {
  private:
    	static const int N = 5;		//常量声明式
    	int array[N];		//使用该常量
  }；
  ```

  通常，编译器不允许在类内初始化static成员数据，需要在类外初始化：

  ```cpp
  class test {
  private:
    	static const int N;		//常量声明式
    	int array[N];		//使用该常量
  }；
  const int test::N = 5;		//初始化
  ```

  如果你不喜欢在类外搞事情，也可以用一个枚举（enum）类型的数值代替：

  ```cpp
  class test {
  private:
    	enum { N = 5 };		//声明一个枚举类型
    	int array[N];
  }；
  ```

使用enum和const还是有些区别的：

- enum的行为有点像#define而不像const
- 因此取一个const地址是合法的，而取一个enum地址就不合法，当然取一个#define地址也是不合法的

如果你不想让别人通过指针或引用指向你个某个整数常量，use enum can help!

\#define还有一个~~坑（雾，事实上要看具体问题）~~就是用它来实现宏，例如：

`#define MAX(a,b) ((a)>(b)?(a):(b))`

你必须为宏中所有实参加上括号，然而更坑的地方就是：即使你加了括号，还是有可能出现问题！，例如：

`MAX(++a,b++);`

`MAX(++a,++b);`

幸运的是，你可以通过template inline同时获得宏的效率和一般函数的所有可预料行为和类型安全性：

```cpp
template<typename T>
inline T& max(const T& a, const T& b)
{
    return a > b ? a : b;
}
```




# 条款03：尽量使用const

> 请记住：
>
> - 将某些东西声明为const可帮助编译器检测出错误的用法。const可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。
> - 当const和non-const版本的成员函数有着实质等价的实现时，令non-const版本调用const版本可以避免代码重复



const允许你指定一个语义约束，而编译器会强制执行这个约束。使用const等于让编译器帮你检查数据的安全性，避免你自己遇到很多坑，所以尽可能的使用const吧！

使用范围：

1.指针

- 指针指向的对象
- 指针本身
- 两者都是

如果const关键字出现在\*号左边，表示被指向的对象时常量。如果const关键字出现在\*号右边，表明指针自身是常量。

例如写法`int const * a;`与写法`const int * a`意思是相同的。

STL迭代器的作用就像一个T\*指针。但是声明迭代器为const则表明迭代器不能指向不同的东西。如果希望迭代器指向的东西不可被改动，你需要的是const_iterator

`vector<T>::iterator iter;`

`const iter`等价于`T * const`,

而`::const_iterator`等价于`const T *`



2.函数声明

- 返回值
- 函数自身（如果是成员函数）
- 参数

令函数返回一个常量值，可以防止使用函数错误造成的意外情况，保证安全性和高效性。例如，声明一个operator\*操作符：

```cpp
class object {...};
const object operator* (const object &lhs, const object &rhs);
```

函数使用者在某些情况下发生这样的错误

```cpp
object a,b,c;
...
if(a * b = c)	//它可能是想单纯比较a*b与c，而不是赋值操作
  {...}
```

将函数返回声明为const能够让编译器发现这个错误而产生警告。

将const用于成员函数的目的在于确认该成员函数可作用于const对象身上。这一类成员函数之所以重要基于两个理由：

- 第一，它们使class接口比较容易理解。可以知道那个成员函数可以改动对象内容，而哪些不可以。
- 第二，它们使“操作const对象”成为了可能。

const成员函数原则上是不能改变对象内数据的，如果一定要改变，请将需要修改的成员数据声明为mutable的。

一个成员函数的const版本和non-const版本可以被重载。

当类中存在函数的const和non-const版本的情况下：

- non-const对象可以调用const成员函数或non-const成员函数；
- const对象只能调用const成员函数，直接调用non-const函数时编译器会报错；

在实际编程实现过程中，为了避免代码重复，可以令non-const版本调用const版本（使用const_cast<>进行强制类型转换）。



# 条款04：确定对象使用前已先被初始化

> 请记住：
>
> - 对内置类型进行手工初始化，C++不保证初始化它们
> - 构造函数最好使用成员初始化列表，而不要在构造函数本体内使用赋值操作。成员初始化列表的顺序应该与声明的顺序一致。
> - 为免除“跨编译单元初始化次序”的问题，请以local static对象替换non-local static对象



使用未初始化的对象会导致不可预知的程序行为。

永远在使用对象之前将它初始化。对于非内置类型的对象而言，初始化责任落在构造函数身上。规则：确保每一个构造函数都将对象的每一个成员都初始化。

不要混淆了初始化（initialization）和赋值（assignment）的概念。C++规定：**对象的成员变量的初始化动作发生在进入构造函数体本体之前**，在构造函数类则是赋值操作。因此尽量使用成员初始化列表。

<u>注：具体细节可以参见另一篇文章《C++与类有关的注意事项总结》</u>