# 条款05：编译器暗中为class创建的函数

> 请记住：
>
> - 编译器可以暗自为class创建default构造函数、copy构造函数、copy assignment（赋值）操作符以及析构函数



如果定义了一个空类，例如

`class Empty {};`

编译器会为它声明（编译器版本的）一个default构造函数、copy构造函数、copy assignment（赋值）操作符以及析构函数。这些函数都是public且inline的。就好像你写下了这些代码

```cpp
class Empty {
public:
  	Empty() {...}
  	Empty(const Empty& rhs) {...}
  	~Empty() {...}
  
  	Empty& operator= (const Empty& rhs) {...}
};
```

default构造函数和析构函数主要给编译器一个地方用来放置幕后代码，像是调用基类和non-satatic成员变量的构造函数和析构函数。

至于copy构造函数和copy assignment操作符，编译器创建的版本只是单纯的将来源对象的每一个non-static成员变量拷贝到目标对象。

如果用户自己声明了一个构造函数，编译器就不会创建default构造函数。

编译器版本的copy assignment操作符，其行为基本上和copy构造函数相同，但一般只有当生出的代码合法且有意义时。如果在一个内含引用（reference）成员或者const成员的类中支持赋值操作，编译器会拒绝为class生成operator=。





# 条款06：若不想使用编译器自动生成的函数，就应该明确拒绝

> 请记住：
>
> - 为驳回编译器自动（暗自）提供的机能，可将相应的成员函数声明为private且不予实现。使用像Uncopyable这样的base class也是一种做法。





# 条款07：为多态基类声明virtual析构函数

> 请记住：
>
> - polymorphic（带多态性质的）基类应当声明一个virtual析构函数。如果class带有任何virtual函数，它就应该有一个virtual析构函数。
> - 如果class的设计目的不是为了作为基类使用，或者不是为了具备多态性，那么不要声明virtual析构函数。





# 条款08：别让异常逃离析构函数

> 请记住：
>
> - 析构函数绝对不要吐出异常。如果一个析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下它们或结束程序。
> - 如果客户要求对某个操作函数运行期间抛出的异常作出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作。



虽然C++并没有禁止析构函数抛出异常，但是通常不推荐这么做。举个例子：

```cpp
class Widget {
  ...
  ~Widget(){...}	//假设析构函数可能会抛出一个异常
}
void doSometing() {
  ...
  std::vector<Widget> v;
  ...
}	//v在这里会自动销毁
```

当返回时，函数创建的v中的每一个Widget将依次被销毁。假设析构第一个Widget元素时抛出了一个异常，剩余的元素还是要被销毁（否则会造成内存的泄露），因此还应该继续调用下一个元素的析构函数。最终会抛出无数的异常，则会导致程序结束或者其他不明确的行为。

假如你必须在析构函数中执行某个动作，而这个动作在可能失败的时候会抛出异常，那怎么办？比如下面的这个例子

```cpp
class DataBase {	//数据库类
public:
	...
	void connect();	//连接
  	void close();	//关闭连接
 	~DataBase() {
    	close();	//需要在销毁对象之前关闭连接，可能会失败
 	}
}
```

有两个方法可以避免这个问题：

1. close()抛出异常之前就结束程序

   ```cpp
   DataBase::~DataBase()
   {
   	try { close(); }
     	catch(...) { std::abort(); }	//直接终止程序
   }
   ```

2. 吞下调用close()产生的异常

   ```cpp
   DataBase::~DataBase()
   {
   	try { close(); }
     	catch(...) { ... }	//记录下错误日志，但不终止程序
   }
   ```

然而，上述两个方法只是解决了“发生了异常该如何解决”，而没有解决“如何避免在析构函数中发生异常”的问题。

最好的方法还是在程序设计接口上改进，即通知用户显式的调用close()来终止连接，而不能依赖于对象的自动实现。在析构函数中调用close()只能作为一种保险的手段。





# 条款09：绝不在构造和析构过程中调用virtual函数

> 请记住：
>
> - 在构造和析构期间不要调用virtual函数，因为这类调用从不会下降到当前类的派生类上



简单的解释是：在base class构造期间，virtual函数不是virtual函数。

由于base class的构造函数执行更早于derived class构造函数，当base class的构造函函数执行时derived class成员变量尚未初始化。如果此期间调用的virtual函数下降至derived class阶层，必然会使用derived class的local成员变量，而那些变量尚未初始化。

最根本的原因是：**在derived class对象的base class构造期间，对象类型是base class而不是derived class**。不只virtual函数会被解析至base class，若使用运行时类型信息（RTII，例如dynamic_cast和typeid），也会把对象视为base class类型。

相同道理也适用于析构函数。一旦derived class的析构函数开始执行，对象里的成员变量将呈现未定义值。进入base class析构函数后对象就成为一个base class对象。

由于你无法通过virtual函数从base class向下调用，在构造期间，你可以通过“令derived class将必要的构造信息向上传递至base class构造函数”加以弥补。



# 条款10：令operator=返回一个reference to *this



# 条款11：在operator=中处理”自赋值“

> 请记住：
>
> - 确保当前对象自我赋值时operator=有良好的行为。其中技术包括比较“来源对象”和“目标对象”的地址、精心周到的语句顺序、以及copy-and-swap。
> - 确定任何函数如果操作一个以上对象，而其中多个对象是同一个对象时，其行为仍然正确。





# 条款12：复制对象时勿忘其每一个部分

