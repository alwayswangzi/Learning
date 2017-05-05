#c++中与类有关的注意事项总结

[TOC]

## 1. 数据成员的初始化

除了const static数据成员外，其他数据成员不能在类中显式的初始化，例如：

``` cpp
class First 
{
	int i = 0; //错误
	double j = 0.0; //错误
};
```

- const成员：只能在定义时初始化或者在构造函数里用成员初始化值来初始化，不能用赋值语句来初始化。
- static成员：只能在类外初始化，且初始化时变量名称不加static修饰。
- const static成员：类只有唯一一份拷贝，且数值不能改变。因此，可以在类中声明处初始化，也可以像static在类外初始化。

## 2. 可以声明一个类但不定义它

例如：

​	`class Screen; //Screen类的声明`

这个声明向程序引入了一个名字 Screen，指示 Screen为一个类类型。但是我们只能以有限的方式使用已经被声明但还没有被定义的类类型，如果没有定义类那么我们就不能定义这类类型的对象。因为类类型的大小不知道，编译器不知道为这种类类型的对象预留多少存储空间。

但是，我们可以声明指向该类类型的指针或引用，允许指针和引用是因为它们都有固定的大小， 这与它们指向的对象的大小无关，但是，因为该类的大小和类成员都是未知的，所以要等到完全定义了该类，我们才能将解引用操作符（ * ）应用在这样的指针上，或者使用指针或引用来指向某一个类成员。

只有已经看到了一个类的定义，我们才能把一个数据成员声明成该类的对象。在程序文本中还没有看到该类定义的地方，数据成员只能是该类类型的指针或引用。例如下面是StackScreen的定义，它有一个数据成员是指向 Screen类的指针，这里 Screen只有声明没有定义：  

``` cpp
class Screen; // 声明
class StackScreen { 
	int topStack; 
	Screen *stack; // ok: 指向一个 Screen 对象 
	void (*handler)(); 
};
```

因为只有当一个类的类体已经完整时，它才被视为已经被定义，所以**一个类不能有自身类型的数据成员**，但是，当一个类的类头被看到时，它就被视为已经被声明了，所以**一个类可以用指向自身类型的指针或引用作为数据成员**。例如 ：

``` cpp
class LinkScreen { 
	LinkScreen iLinkScreen； // 错误：一个类不能有自身类型的数据成员
	LinkScreen *next; 
	LinkScreen *prev; 
};
```

对于空类，编译器不会生成任何的成员函数，只会生成1个字节的占位符。大家可能会以为编译器会为空类生成默认构造函数等，事实上是不会的，编译器只会在需要的时候生成6个成员函数：一个缺省的构造函数、一个拷贝构造函数、一个析构函数、一个赋值运算符、一对（const、非const）取址运算符和一个this指针。

## 3. inline和非inline成员函数

在类体中定义的成员函数默认是inline函数，可以显示的加上inline；

通常，在类体外定义的成员函数不是 inline 的。但是，这样的函数也可以被声明为 inline函数。可以通过显式地在类体中出现的函数声明上使用关键字 inline 或者通过在类体外出现的函数定义上显式使用关键字 inline，或者两者都用。

## 4. const成员函数

通常，程序中任何试图修改const对象都会被标记为编译错误，例如：
​	`const char blank=' ';`
​	`blank='\n'; //错误`

但是，程序通常不直接修改类对象，有时在必须修改类的对象时，才调用公有成员函数集来完成，为尊重类对象的常量性，编译器必须区分不安全与安全的成员函数（即区分试图修改类对象与不试图修改类对象的函数 ），例如：

``` cpp
const Screen blankScreen;
blankScreen.display(); // 读类对象 
blankScreen.set( '*' ); // 错误: 修改类对象
```

类的设计者通过把成员函数声明为 const 以表明它们不修改类对象，例如 :

``` cpp
public: 
	char get() const { return _screen[_cursor]; } 
	// ... 
}
```

只有被声明为const的成员函数才能被一个const类对象调用，关键字const被放在成员函数的参数表和函数体之间。对于在类体之外定义的const成员函数，我们必须在它的声明和定义中同时指定关键字const。

**把一个修改类内数据成员的函数声明为const是非法的**。例如，在下面的简化Screen定义中：

``` cpp
public:
	int ok() const {return _cursor;}
	void error(int ival) const {_cursor=ival;} //错误
	//...
private:
	string::size_type _cursor;
	//...
}
```

ok() 的定义是一个有效的 const 成员函数定义，因为它没有改变 \_cursor 的值，但是 error() 的定义修改了 _cursor 的值，因此它不能被声明为一个 const 成员函数，这个函数定义将导致下面的编译器错误消息：
​	`error: cannot modify a data member within a const member function`

但是，把一个成员函数声明为const 并不能阻止程序员可能做到的所有修改动作，把一个成员函数声明为 const 可以保证这个成员函数不修改类的数据成员 。但是，**如果该类含有指针，那么在 const 成员函数中就能修改指针所指的对象，编译器不会把这种修改检测为错误**，例如 ：

``` cpp
#include <cstring> 
class Text { 
public: 
	void bad( const string &parm ) const; 
private: 
	char *_text; 
}; 
 
void Text::bad( const string &parm ) const 
{ 
	_text = parm.c_str(); // 错误: 不能修改 _text 
	for ( int ix = 0; ix < parm.size(); ++ix ) 
		_text[ix] = parm[ix]; // 不好的风格, 但不是错误的 
}
```

尽管 \_text 不能被修改，但是 \_text 的类型是 char* 在类 Text 的 const 成员函数中可以修改_text 指向的字符，成员函数 bad() 反映了一种不良的程序设计风格。
const 成员函数可以被相同参数表的非 const 成员函数重载，例如：

``` cpp
class Screen { 
public: 
	char get(int x, int y); 
	char get(int x, int y) const; 
	// ... 
};
```

在这种情况下，类对象的常量性决定了调用哪个函数：

``` cpp
int main() { 
	const Screen cs; 
	Screen s; 
	char ch = cs.get(0, 0); // 调用 const 成员 
	ch = s.get(0, 0); // 调用非 const 成员 
}
```

构造函数和析构函数是两个例外，即使构造函数和析构函数不是 const 成员函数，const 类对象也可以调用它们。当构造函数执行结束，类对象已经被初始化时，类对象的常量性就被建立起来了，析构函数一被调用，常量性就消失。所以一个 const 类对象从构造完成时刻到析构开始时刻，这段时间内被认为是const。

## 5. volatile关键字

可以将成员函数声明为volatile，如果一个类对象的值可能被修改的方式是编译器无法控制或检测的，例如：如果它是表示 I/O端口的数据结构，则把它声明为volatile。与 const 类对象类似，对于一个 volatile类对象，只有 volatile成员函数、构造函数和析构函数可以被调用。

``` cpp
class Screen { 
public: 
	char poll() volatile;  
	// ...
}; 
char Screen::poll() volatile { ... }
```

一个定义为volatile的变量是说这变量可能会被意想不到地改变。这样，编译器就不会去假设这个变量的值了。精确地说就是，优化器在用到这个变量时必须每次都小心地重新读取这个变量的值，而不是使用保存在寄存器里的备份。下面是volatile变量的几个例子：

1. 并行设备的硬件寄存器（如：状态寄存器）
2. 一个中断服务子程序中会访问到的非自动变量(Non-automatic variables)
3. 多线程应用中被几个任务共享的变量

回答不出这个问题的人是不会被雇佣的。我认为这是区分C程序员和嵌入式系统程序员的最基本的问题。搞嵌入式的家伙们经常同硬件、中断、RTOS等等打交道，所有这些都要求用到volatile变量。不懂得volatile的内容将会带来灾难。假设被面试者正确地回答了这是问题（嗯，怀疑是否会是这样），我将稍微深究一下，看一下这家伙是不是直正懂得volatile完全的重要性。

1. 一个参数既可以是const还可以是volatile吗？解释为什么。
2. 一个指针可以是volatile 吗？解释为什么。
3. 下面的函数有什么错误：

``` cpp
int square(volatile int *ptr)
{
	return *ptr * *ptr;
}
```

下面是答案：

1. 是的。一个例子是只读的状态寄存器。它是volatile因为它可能被意想不到地改变。它是const因为程序不应该试图去修改它。
2. 是的。尽管这并不很常见。一个例子是当一个中断服务子程序修该一个指向一个buffer的指针时。
3. 这段代码有点变态。这段代码的目的是用来返指针\*ptr指向值的平方，但是，由于*ptr指向一个volatile型参数，编译器将产生类似下面的代码：

``` cpp
int square(volatile int *ptr)
{
	int a,b;
	a = *ptr;
	b = *ptr;
	return a * b;
}
```

volatile 相当于告诉编译器，由它声明的东西的易变的，不确定的，可能由外部程序 (如中断程序) 改变的，禁止编译器对其读写操作进行优化。如果定义`int i`，则编译器可能会将其优化，而放到CPU寄存器中，这在多数情况下是好的。然而在有些情况下，我们会要求一些变量必须在内存中（如驱动程序、中断服务程序等等），这时编译器这个优化就是引起问题。为了避免这种情况，应该讲其定义`volatile int i`。

PS：**volatile 通常也用来阻止编译器具优化操作**，如你有一个非精确延时函数：

``` cpp
void delay(unsigned int timeout)
{
	unsigned int i;
	for (i = 0; i < timeout; i++);
}
```

有些编译会足够聪明地注意到这个函数本质上是什么也没干，会将针对这个函数的调用优化掉，但这样是不对的， 所以你应该这么声明：

``` cpp
volatile void delay(...)
{
	// 同上
}
```

## 6. mutable数据成员

当我们把一个 Screen类对象声明为 const 时出现了一些问题，我们期望的行为是 一旦 const Screen对象被初始化，它的内容就不能被修改，但是我们应该能够监视到 Screen对象的内容。例如，给出下面的 Screen对象 cs：`const Screen cs(5, 5);`

我们想监视在位置(3, 4)的内容，我们这样做 ：

``` cpp
// 读位置(3, 4)的内容 
// 喔! 不能工作 
cs.move( 3, 4 ); 
char ch = cs.get();
```

但是，这不能工作，为什么？move()不是const成员函数，而且不能很容易的变成const成员函数。move()的定义如下：

``` cpp
inline void Screen::move(int r, int c) 
{ 
	if ( checkRange( r, c ) ) 
	{ 
  	int row = (r-1) * _width; 
  	_cursor = row + c - 1;  // 修改 _cursor 
	} 
}
```

我们注意到move()修改了数据成员\_cursor，因此若不加改动，它就不能被声明为const，但是，对一个 Screen类的 const 对象不能修改\_cursor 这看起来可能很奇怪，因为\_cursor只是一个索引，修改\_cursor 不会修改 Screen本身的内容，我们只是想记住要被监视的 Screen位置，即使 Screen对象是 const 也应该允许修改\_cursor，因为这么做对于监视 Screen对象内容是必需的，而且又不会修改 Screen本身的内容。

**为了允许修改一个类的数据成员，即使它是一个const对象的数据成员，我们也可以把该数据成员声明为mutable（易变的）， mutable 数据成员永远不会是 const 成员，即使它是一个const 对象的数据成员，mutable 成员总可以被更新，即使是在一个 const 成员函数中**。为把一个成员声明为mutable 数据成员，我们必须把关键字 mutable 放在类成员表中的数据成员声明之前：

``` cpp
class Screen { 
public: 
	// 成员函数 
private: 
	string _screen; 
	mutable string::size_type _cursor; // mutable 成员 
	short _height; 
	short _width; 
};
```

现在任何 const 成员函数都可以修改\_cursor，我们可以把成员函数 move()声明为 const ，即使 move()修改了数据成员\_cursor，也不会有编译错误产生：

``` cpp
// move() 是一个 const 成员函数 
inline void Screen::move( int r, int c ) const 
{ 
	// ... 
	// ok: const 成员函数可以修改 mutable 成员 
	_cursor = row + c - 1; 
	// ... 
}
```

现在我们可以执行开始时给出的操作来监视Screen对象 cs 而不会有错误发生。我们注意到只有\_cursor被声明为mutable 数据成员，而\_screen\_height 和\_width都没有，因为这些数据成员的值在const的Screen类对象中是不应该被改变的。

## 7. 隐含的this指针

先要理解class的意思：class应该理解为一种类型，像int、char一样，是用户自定义的类型（虽然比int char这样build-in类型复杂的多，但首先要理解它们一样是类型)。用这个类型可以来声明一个变量，比如int x, myclass my等等。这样就像变量x具有int类型一样，变量my具有myclass类型。
理解了这个，就好解释this了，my里的this就是指向my的指针。如果还有一个变量myclass mz，mz的this就是指向mz的指针。这样就很容易理解this的类型应该是myclass \*，而对其的解引用\*this就应该是一个myclass类型的变量。

通常在class定义时要用到类型变量自身时，因为这时候还不知道变量名（为了通用也不可能固定实际的变量名），就用this这样的指针来使用变量自身。

### 7.1 this指针的用处

一个对象的this指针并不是对象本身的一部分，不会影响sizeof(对象)的结果。this作用域是在类内部，当在类的非静态成员函数中访问类的非静态成员的时候，编译器会自动将对象本身的地址作为一个隐含参数传递给函数。也就是说，**即使你没有写上this指针，编译器在编译的时候也是加上this的，它作为非静态成员函数的隐含形参，对各成员的访问均通过this进行**。例如，调用date.SetMonth(9) <===> SetMonth(&date, 9)，this帮助完成了这一转换。

### 7.2 this指针的使用

- 第一种情况，在类的非静态成员函数中返回类对象本身的时候，直接使用 return *this；
- 第二种情况，当参数与成员变量名相同时，如this->n = n （不能写成n = n）；
- 第三种情况， this指针含有被调用的类对象的地址。例如：

``` cpp
void Screen::copy(const Screen& sobj) 
{ 
 	// 如果 Screen 对象与 sobj 是同一个对象则无需拷贝 
    if(this != &sobj) 
	{ 
  	// 把 sobj 的值拷贝到 *this 中 
	} 
```

如果sobj指向的对象的地址与this指针值相等，则sobj和this指向同一对象，拷贝动作是不必要的。

### 7.3 this指针程序示例

this指针是存在与类的成员函数中,指向被调用函数所在的类实例的地址。根据以下程序来说明this指针：

``` cpp
#include <iostream>
class Point
{ 
	int x, y;
public:
  	Point(int a, int b) { x=a; y=b;}
  	void MovePoint(int a, int b) { x+=a; y+=b;}
  	void print() {cout<<"x="<<x<<"y="<<y<<endl;}
};
 
void main( )
{
   Point point1(10,10);
   point1.MovePoint(2,2);
   point1.print();
}
```

当对象point1调用MovePoint(2,2)函数时，即将point1对象的地址传递给了this指针。
MovePoint函数的原型应该是`void MovePoint(Point *this, int a, int b);`第一个参数是指向该类对象的一个指针，我们在定义成员函数时没看见是因为这个参数在类中是隐含的。

这样point1的地址传递给了this，所以在MovePoint函数中便显式的写成：

​	`void MovePoint(int a, int b) { this->x +=a; this-> y+= b;}`

即可以知道，point1调用该函数后，也就是point1的数据成员被调用并更新了值。即该函数过程可写成

 	`point1.x+= a; point1. y + = b;`

### 7.4 关于this指针的一个经典回答

当你进入一个房子后，
你可以看见桌子、椅子、地板等，
但是房子你是看不到全貌了。

对于一个类的实例来说，
你可以看到它的成员函数、成员变量，
但是实例本身呢？
this是一个指针，它时时刻刻指向你这个实例本身。

## 8. 静态成员和静态成员函数

静态类成员包括静态数据成员和静态函数成员两部分。

### 8.1 静态数据成员

类体中的数据成员的声明前加上static关键字，该数据成员就成为了该类的静态数据成员。和其他数据成员一样，静态数据成员也遵守public/protected/private访问规则。同时，静态数据成员还具有以下特点：

#### 8.1.1 静态数据成员的定义

静态数据成员实际上是类域中的全局变量。因此，静态数据成员的定义（初始化）不应该放在头文件中，其定义方式与全局变量相同。举例如下：

``` cpp
// .h文件
class base{ 
private: 
	static const int _i; //声明，标准c++支持有序类型在类体中初始化,但vc6不支持。 
};

// .cpp文件
const int base::_i=10; //定义(初始化)时不受private和protected访问限制.

```

注：**不要试图在头文件中定义（初始化）静态数据成员。在大多数的情况下，这样做会引起重复定义这样的错误。即使加上 #ifndef #define #endif 或者 #pragma once 也不行。**

#### 8.1.2 静态数据成员被类的所有对象所共享，包括派生类的对象

即派生类对象与基类对象共享基类的静态数据成员。举例如下：

``` cpp
class base { 
public : 
	static int _num;//声明 
}; 
int base::_num=0;//静态数据成员的真正定义 
class derived:public base { 
};

int main() { 
	base a; 
	derived b; 
	a._num++; 
	cout<<"base class static data number _num is"<<a._num<<endl; 
	b._num++; 
	cout<<"derived class static data number _num is"<<b._num<<endl; 
}
// 结果为1,2，可见派生类与基类共用一个静态数据成员
```

#### 8.1.3 静态数据成员可以成为成员函数的可选参数，而普通数据成员则不可以

举例如下：

``` cpp
class base { 
public : 
	static int _staticVar; 
	int _var; 
	void foo1(int i=_staticVar);//正确，_staticVar为静态数据成员 
	void foo2(int i=_var);//错误，_var为普通数据成员 
};
```

#### 8.1.4 ★静态数据成员的类型可以是所属类的类型，而普通数据成员则不可以

普通数据成员的只能声明为所属类类型的指针或引用。举例如下：

```cpp
class base { 
public : 
	static base _object1;//正确，静态数据成员 
	base _object2;//错误 
	base *pObject;//正确，指针 
	base &mObject;//正确，引用 
};
```

#### 8.1.5 ★静态数据成员的值在const成员函数中可以被合法的改变

~~这个特性，我不知道是属于标准c++中的特性，还是vc6自己的特性~~。举例如下：

``` cpp
class base { 
public: 
	base(){_i=0; _val=0;} 
	mutable int _i; 
	static int _staticVal; 
	int _val; 
	void test() const { //const成员函数 
		_i++; //正确，mutable数据成员 
		_staticVal++; //正确，static数据成员 
		_val++; //错误 
	} 
}; 
int base::_staticVal = 0;
```

### 8.2 静态成员函数

#### 8.2.1 静态成员函数的地址可用普通函数指针保存，而普通成员函数的地址需要用类成员函数指针来保存

举例如下：

``` cpp
class base { 
	static int func1(); 
	int func2(); 
}; 
int (*pf1)() = &base::func1; //普通的函数指针 
int (base::*pf2)() = &base::func2; //成员函数指针
```

#### 8.2.2 静态成员函数不可以调用类的非静态成员，因为静态成员函数不含this指针

#### 8.2.3 静态成员函数不可以同时声明为virtual、const、volatile函数

举例如下： 
``` cpp
class base { 
	virtual static void func1();//错误 
	static void func2() const;//错误 
	static void func3() volatile;//错误 
};
```

**最后要说的一点是，静态成员是可以独立访问的，也就是说，无须创建任何对象实例就可以访问。**

## 9. 指向类成员的指针

假定 Screen类定义了四个新成员函数：forward()、back()、up()和down()，它们分别向右、向左、向上和向下移动光标。首先，我们在类体中声明这些新的成员函数：

``` cpp
class Screen { 
public: 
    inline Screen& forward(); 
    inline Screen& back(); 
    inline Screen& end(); 
    inline Screen& up(); 
    inline Screen& down(); 
    // 其他成员函数同前 
private: 
    inline int row(); 
    // 其他私有成员同前 
};
```

Screen类的用户要求一个函数 repeat()，它执行用户指定的操作 n 次，它的非通用的实现如下： 

``` cpp
Screen &repeat(char op, int times) { 
	switch(op) { 
	case DOWN: // 调用 Screen::down() n 次 
 		break; 
	case UP: // 调用 Screen::up() n 次 
  		break; 
	// ... 
	} 
}
```

虽然这种实现能够工作，但是它有许多缺点。一个问题是，它依赖于 Screen的成员函数保持不变，每次增加或删减一个成员函数，都必须更新 repeat()；第二个问题是它的大小，由于必须测试每个可能的成员函数，所以repeat()的完整列表非常大而且不必要地复杂。 

替换的办法是一种更通用的实现：**用Screen成员函数的指针类型的参数取代OP**，repeat() 不需要再确定用户期望什么样的操作，整个switch 语句也可以被去掉。

### 9.1 类成员的类型

**函数指针不能被赋值为成员函数的地址，即使返回类型和参数表完全匹配**。例如 ，下面的 pfi是一个函数指针，该函数没有参数，返回类型为int：
​	`int (*pfi) ();`
给出两个全局函数HeightIs()和WidthIs()：
​	`int HeightIs();`
​	`int WidthIs();`
把HeightIs()和WidthIs()中的任何一个或两个赋值给指针pfi都是合法且正确的：
​	`pfi = HeightIs;`
​	`pfi = WidthIS;`
类 Screen也定义了两个访问函数height()和width()，它们也没有参数，返回类型也 是int：
​	`inline int Screen::height() {return _height;}`
​	`inline int Screen::width() {return _width;}`
但是把 height()或width()任何一个赋给指针pfi都是类型违例，都将导致编译时刻错误：	
​	`pfi = &Screen::height; //非法赋值: 类型违例` 

为什么会出现类型违例呢？**成员函数有一个非成员函数不具有的属性——它的类（it's class），指向成员函数的指针必须与向其赋值的函数类型匹配，不是两个而是三个方面都要匹配 ：(1) 参数的类型和个数 (2) 返回类型 (3) 它所属的类类型**。

成员函数指针首先必须被绑定在一个对象或者一个指针上，才能得到被调用对象的 this指针，然后才调用指针所指的成员函数。虽然普通函数指针和成员函数指针都被称作指针，但是它们是不同的事物。

成员函数指针的声明要求扩展的语法，它要考虑类的类型，对指向类数据成员的指针也是这样。考虑 Screen类的成员 height 的类型，它的完整类型是“short 型的 Screen类的成 员”，指向_height 的指针的完整类型是“指向short 型的 Screen类的成员”的指针，这可以写为：
​	`short Screen::*`
指向 short 型的 Screen类的成员的指针的定义如下：
​	`short Screen::*ps_Screen;`
ps_Screen可以用_height 的地址初始化，如下：
​	`short Screen::*ps_Screen = &Screen::_height; `
类似地，它也可以用_width的地址赋值，如下：
​	`ps_Screen = &Screen::_width;`

**定义一个成员函数指针需要指定函数返回类型、参数表和类**，例如，指向Screen成员函数并且能够引用成员函数 height()和 width()的指针类型如下：
​	`int (Screen::*) ()`
这种类型指定了一个指向类 Screen的成员函数的指针，它没有参数，返回值类型为int 。指向成员函数的指针可被声明、初始化及赋值如下：

``` cpp
// 所有指向类成员的指针都可以用 0 赋值 
int (Screen::*pmf1)() = 0; 
int (Screen::*pmf2)() = &Screen::height; 

pmf1 = pmf2; 
pmf2 = &Screen::width;
```

使用 typedef可以使成员指针的语法更易读。例如，下面的类型定义：
​	`Screen& ( Screen::* ) ()`
也就是指向 Screen类成员函数的一个指针，该函数没有参数，返回Screen类对象的引用。它可以被下列 typedef定义Action所取代：
​	`typedef Screen& (Screen::*Action)();` 
​	`Action default = &Screen::home;` 
​	`Action next = &Screen::forward;`

指向成员函数类型的指针可以被用来声明函数参数和函数返回类型 ，我们也可以为成员函数类型的参数指定缺省实参，例如：
​	`Screen& action( Screen&, Action );`
action()被声明为有两个参数，一个 Screen类对象的引用和一个指向类 Screen的成员函数的指针，该函数没有参数，返回Screen类对象的引用。action()可以以下列任意方式被调用：
``` cpp
Screen myScreen; 
typedef Screen& (Screen::*Action) (); 
Action default = &Screen::home; 
extern Screen& action(Screen&, Action = &Screen::display); 
void ff() 
{
	action(myScreen); 
	action(myScreen, default); 
	action(myScreen, &Screen::end); 
}
```

### 9.2 使用指向类成员的指针

**类成员的指针必须总是通过特定的对象或指向该类类型的对象的指针来访问 ，我们通过使用两个指向成员操作符的指针（针对类对象和引用的.* 以及针对指向类对象的指针的->*）**来做到这一点，例如，如下所示， 我们通过成员函数的指针调用成员函数：

``` cpp
int (Screen::*pmfi)() = &Screen::height; 
Screen& (Screen::*pmfS)( const Screen& ) = &Screen::copy; 
Screen myScreen, *bufScreen;  

// 直接调用成员函数 
if ( myScreen.height() == bufScreen->height() ) 
bufScreen->copy( myScreen ); 
 
// 通过成员指针的等价调用 
if ( (myScreen.*pmfi)() == (bufScreen->*pmfi)() ) 
(bufScreen->*pmfS)( myScreen );
```

如下调用：
`(myScreen.*pmfi)() `
`(bufScreen->*pmfi)()`
要求有括号 因为调用操作符“()”的优先级高于成员操作符指针的优先级，没有括号`myScreen.*pmfi()`将会被解释为`myScreen.*(pmfi())`，它会先调用函数 pmfi() 把它的返回值与成员对象操作符的指针 .* 绑定，当然 pmfi的类型不支持这种用法，会产生一个编译时刻错误。 

 类似地，指向数据成员的指针可以按下列方式被访问:

``` cpp
typedef short Screen::*ps_Screen; 
 
Screen myScreen, *tmpScreen = new Screen(10, 10); 
 
ps_Screen pH = &Screen::_height; 
ps_Screen pW = &Screen::_width; 
 
tmpScreen->*pH = myScreen.*pH; 
tmpScreen->*pW = myScreen.*pW;
```

下面是在开始时讨论的成员函数repeat()的实现，它被修改为用一个成员函数的指针作为参数：

``` cpp
typedef Screen& (Screen::*Action)(); 
 
Screen& Screen::repeat(Action op, int time s) 
{ 
	for(int i = 0; i < times; ++i ) 
  		(this->*op)(); 
return *this; 
}
```

参数 op 是一个成员函数的指针 它指向要被调用times次的成员函数，若要为repeat() 的参数提供缺省实参，则声明如下：

``` cpp
class Screen { 
public: 
Screen &repeat( Action = &Screen::forward, int = 1 ); 
// ... 
};
```

repeat()的调用如下：

``` cpp
Screen myScreen; 
myScreen.repeat(); // repeat(&Screen::forward, 1); 
myScreen.repeat( &Screen::down, 20);
```

### 9.3 静态类成员的指针

**在非静态类成员的指针和静态类成员的指针之间有一个区别，指向类成员的指针语法不能被用来引用类的静态成员，静态类成员是属于该类的全局对象和函数，它们的指针是普通指针**。（请记住静态成员函数没有 this 指针）。

**指向静态类成员的指针的声明看起来与非类成员的指针相同，解引用该指针不需要类对象**，例如 我们来看一下类 Account：

``` cpp
class Account { 
public: 
	static void raiseInterest( double incr ); 
	static double interest() { return _interestRate; } 
	double amount() { return _amount; } 
private: 
	static double _interestRate; 
	double _amount; 
	string _owner; 
}; 
 
inline void Account::raiseInterest( double incr ) 
{ 
	_interestRate += incr; 
}
```

&\_interestRate的类型是double\* 而不是 double Account::\*，指向\_interestRate的指针定义如下： `double *pd = &Account::_interestRate;` **它被解引用的方式与普通指针一样，不需要相关的类对象**。

但是，因为\_interestRate和\_amount 都是私有成员，所以我们需要使用静态成员函数interest()和非静态成员函数amount()。指向interest()的指针的类型是一个普通函数指针
​	`double (*) () // 正确 `
而不是类 Account 的成员函数的指针
​	`double (Account::*) () // 不正确 `
这个指针的定义和对 interest()的间接调用处理方式与非类的指针相同：
​	`// ok: douple(*pf) () 不是 double(Account::*pf)() `
​	`double (*pf)() = &Account::interest;`
​	`double daily = pf () / 365 * unit.amount();`

## 10. 嵌套类与局部类

### 10.1 嵌套类

在一个类的内部定义另一个类，我们称之为嵌套类（nested class），或者嵌套类型。之所以引入这样一个嵌套类，往往是因为外围类需要使用嵌套类对象作为底层实现，并且该嵌套类只用于外围类的实现，且同时可以对用户隐藏该底层实现。

虽然嵌套类在外围类内部定义，但它是一个独立的类，基本上与外围类不相关。它的成员不属于外围类，同样，外围类的成员也不属于该嵌套类。嵌套类的出现只是告诉外围类有一个这样的类型成员供外围类使用。并且，**外围类对嵌套类成员的访问没有任何特权，嵌套类对外围类成员的访问也同样如此，它们都遵循普通类所具有的标号访问控制**。

*若不在嵌套类内部定义其成员，则其定义只能写到与外围类相同的作用域中，且要用外围类进行限定，不能把定义写在外围类中*。例如，嵌套类的静态成员就是这样的一个例子。

前面说过，之所以使用嵌套类的另一个原因是达到底层实现隐藏的目的。为了实现这种目的，*我们需要在另一个头文件中定义该嵌套类，而只在外围类中前向声明这个嵌套类即可*。当然，*在外围类外面定义这个嵌套类时，应该使用外围类进行限定*。使用时，只需要在外围类的实现文件中包含这个头文件即可。

**另外，嵌套类可以直接引用外围类的静态成员、类型名和枚举成员（假定这些成员是公有的）**。类型名是一个typedef名字、枚举类型名或是一个类名。实例如下：

``` cpp
#ifndef NESTCLASS_H_ 
#define NESTCLASS_H_ 
class A {
public:
     A();
     ~A();
     void operate();
private:
     class B;
     B* m_b;
}; 
#endif
```

``` cpp
#include "nestclass.h" 
#include <iostream> 
using namespace std;
class A::B {
public:
     B() {}
     ~B() {}
     void operate() {
         cout<<"B operate!"<<endl;
     }
};
 
A::A() {}
A::~A() {}
void A::operate() {
    m_b = new B;
    cout<<"A operate!"<<endl;
    m_b->operate();
}
```

``` cpp
#include "nestclass.h" 
void main() {
     A a;
     a.operate();
}
```

在嵌套类的定义被看到之前我们只能声明嵌套类的指针和引用，如上面在A中定义为B m_b而不是B* m_b将会引发一个编译错误。

关于C++嵌套类的详细用法请参考《C++ Primer 第三版》P551。

### 10.2 局部类

类也可以定义在函数体内 这样的类被称为局部类(local class)， 局部类只在定义它的局部域内可见，与嵌套类不同的是，在定义该类的局部域外没有语法能够引用局部类的成员，因此，**局部类的成员函数必须被定义在类定义中**，在实际中，这就把局部类的成员函数的复杂性限制在几行代码中，否则对读者来说，代码将变得很难理解。 

因为没有语法能够在名字空间域内定义局部类的成员，所以也**不允许局部类声明静态数据成员**。

在局部类中嵌套的类可以在其类定义之外被定义，但是，该定义必须出现在包含外围局部类定义的局部域内。在局部域定义中的嵌套类的名字必须由其外围类名限定修饰，在外围类中，该嵌套类的声明不能被省略。例如：

```cpp
void foo( int val ) { 
	class Bar { 
	public: 
  		int barVal; 
  		class nested; // 嵌套类的声明是必需的 
	}; 
 
	// 嵌套类定义 
	class Bar::nested { 
  		// ... 
	}; 
}
```

外围函数没有特权访问局部类的私有成员，当然，这可以通过使外围函数成为局部类的友元来实现。

同嵌套类一样，局部类可以访问的外围域中的名字也是有限的，局部类只能访问在外围局部域中定义的类型名、静态变量以及枚举值，例如：

```cpp
int a, val;  
void foo(int val) { 
	static int si; 
	enum Loc {a = 1024, b};  
	class Bar { 
	public: 
    	Loc locVal; // ok; 
  		int barVal; 
  		void fooBar(Loc l = a) { // ok: Loc::a 
   			barVal = val; // 错误: 局部对象 
   			barVal = ::val; // OK: 全局对象 
   			barVal = si; // ok: 静态局部对象 
   			locVal = b; // ok: 枚举值 
  		} 
	}; 
// ... 
}
```

在局部类体内，不包括成员函数定义中的的名字解析过程是，在外围域中查找出现在局部类定义之前的声明。在局部类的成员函数体内的名字的解析过程是：在查找外围域之前，首先直找该类的完整域 。

还是一样，如果先找到的声明使该名字的用法无效，则不考虑其他声明，即使在fooBar()中使用val 是错的，编译器也不会找到全局变量val，除非用全局域解析操作符限定修饰val，如::val。

## 11. 构造函数

### 11.1 我们可以给构造函数提供一个缺省的实参

```cpp
class Account { 
public: 
	// 缺省构造函数 
	Account();  
	// 声明中的参数名不是必需的 
	Account(const char*, double=0.0); 
	const char* name() {return _name;} 
	// ...
private: 
	// ... 
};
```

下面是两个合法的 Account 类对象定义,它们向构造函数传递了一个或两个实参: 

```cpp
int main() { 
	// ok: 都调用双参数构造函数 
	Account acct( "Ethan Stern" ); 
	Account *pact = new Account( "Michael Lieberman", 5000 );  
	if ( strcmp( acct.name(), pact->name() )) 
  // ... 
}
```

### 11.2 构造函数形式

C++语言新手常犯的错误是：按如下方式声明一个用缺省构造函数初始化的对象

`Account newAccount();`

它能通过编译 但是 当我们试图使用它时 `if (!newAccount.name()) ...`，编译器会抱怨我们不能把成员访问符应用到函数上，定义`Account newAccount();` 被编译器解释为定义了一个没有参数，返回一个 Account 类型对象的函数——完全不是我们的意图，用缺省构造函数初始化类对象的正确声明是去掉尾部的小括号。

只有当没有构造函数或声明了缺省构造函数时，我们才能不指定实参集来定义类对象。如果定义了带参数的构造函数，最好也定义一个默认的构造函数。

### 11.3 成员初始化列表

类的初始化有一种可替换的语法，成员初始化表（member initialization list ），是由逗号分开的成员名及其初值的列表。成员初始化表只能在构造函数定义中被指定，而不是在其声明中。该初始化表被放在参数表和构造函数体之间，由冒号开始，下面是构造函数体。

### 11.4 防止构造函数隐式地被用作转换操作符（explicit）

考虑如下程序段：

```cpp
// 在某个头文件中 
extern void print(const Account &acct); 
// ... 
 
int main() {
	// 把 "oops" 转换成一个 Account 对象 
	// 用 Account::Account( "oops", 0.0 ) 
	print( "oops" );  
	// ... 
}
```

缺省情况下，单参数构造函数或者构造函数有多个参数（除了第一个参数外， 其他都有缺省实参）被用作转换操作符，在上面的程序段中的print()的调用里 Account 构造函数被编译器隐式地应用，以便把一个文字字符串转换成一个 Account 对象，尽管这种转换在这种情况下并不合适。 

无意的隐式类转换，如把 oops 转换成一个 Account 对象，已经被证明是很难跟踪的错误源，关键字 explicit 被引入到标准 C++中以帮助我们抑制这种不受欢迎的编译器辅助行为，explicit 修饰符通知编译器不要提供隐式转换： 
```cpp
class Account { 
public: 
	explicit Account( const char*, double=0.0 ); 
	// ... 
};
```
**explicit 只能被应用在构造函数上**。

### 11.5 拷贝构造函数

用一个类对象初始化该类的另一个对象被称为缺省按成员初始化（default memberwise initialization），在概念上，*一个类对象向该类的另一个对象作拷贝是通过依次拷贝每个非静态数据成员来实现的*。

缺省按成员的初始化对于类的正确行为常常是不合适的，通过定义拷贝构造函数的显式实例，我们可以改变缺省的行为，我们的Account类要求我们这样做，否则两个Account对象会有相同的帐号，这在该类的规范中显然是不允许的。

拷贝构造函数有一个指向类对象的引用作为形式参数，传统上被声明为 const ，下面是它的实现：

```cpp
inline Account::Account (const Accout &rhs) : _balance(rhs._balance) 
{ 
	_name = new char[ strlen(rhs._name)+1 ]; 
	strcpy( _name, rhs._name ); 
	// 不能拷贝 rhs._acct_nmbr 
	_acct_nmbr = get_unique_acct_nmbr(); 
} 
```

当我们写 `Account acct2( acct1 );`，编译器判断是否为 Account 类声明了一个显式的拷贝构造函数，如果声明了拷贝构造函数，并且是可以访问的，则调用它；如果声明了拷贝构造函数但是不可访问，则acct2的定义就是一个编译时刻错误；如果没有声明拷贝构造函数的实例，则执行缺省的按成员初始化。如果我们后来引入或去掉了一个拷贝构造函数的声明，则用户程序无需改变，但是需要重新编译它们。

## 12. 析构函数

### 12.1 析构函数的定义

析构函数是一个特殊的由用户定义的成员函数，当该类的对象离开了它的域或者delete表达式应用到一个该类的对象的指针上时，析构函数会自动被调用。析构函数的名字是在类名前加上波浪线 ~ ，它不返回任何值也没有任何参数，因为它不能指定任何参数，所以它也不能被重载。尽管我们可以为一个类定义多个构造函数，但是我们只能提供一个析构函数，它将被应用在类的所有对象上。下面是Account 类的析构函数：

```cpp
class Account { 
public: 
	Account(); 
	explicit Account(const char*, double=0.0); 
	Account(const Account&); 
	~Account(); 
	// ... 
private: 
	char *_name; 
	unsigned int _acct_nmbr; 
	double _balance; 
};

inline Account::~Account() { 
	delete [] _name; 
	return_acct_nmbr( _acct_nmbr ); 
}
```

一般地，如果一个类的数据成员是按值存储的，则无需析构函数。并不是每一个类都要求有析构函数，即使我们为该类定义了一个或多个构造函数，析构函数主要被用来放弃在类对象的构造函数或生命期中获得的资源，如释放互斥锁或删除由操作符new分配的内存。

当类对象的指针或引用离开域时，被引用的对象还没有结束生命期，析构函数不会被调用。  

**C++语言在内部保证，不会用 delete 操作符删除不指向任何对象的指针**，所以我们无需再编写代码来保证这一点：
​	`if (pact != 0) delete pact; // 没有必要——由编译器隐式执行`

无论何时，当在一个函数内删除一个独立的堆对象时，最好是用 auto_ptr 类对象而不是一个实际的指针。对于堆上的类对象尤其应该这样做，否则的话，如果应用 delete 表达式失败（比如一个异常被抛出的情况下），不仅会导致内存泄漏，而且析构函数也不会被调用。例如，下面是我们用auto_ptr 重写之后的程序示例，它被稍做修改因为 auto_ptr 对象不支持被显式地重置以指向第二个对象，除非赋值第二个auto_ptr。

```cpp
#include <memory> 
#include "Account.h" 
 
Account global("James Joyce"); 
int main() { 
	Account local("Anna Livia Plurabelle", 10000); 
	Account &loc_ref = global; 
	auto_ptr<Account> pact(new Account("Stephen Dedalus"));

	{ 
  		Account local_too("Stephen Hero"); 
	} 
	// auto_ptr 对象在此被销毁 
}
```

### 12.2 显式的析构调用

在某些程序情况下，有必要显示地对一个特殊类对象调用析构函数。这常常发生在和定位 new操作符结合的时候。让我们看一个例子，当写：
​	`char *arena = new char [ sizeof Image];`
时，实际上我们已经分配了一个大小等于 Image 型对象的新的堆存储区。相关联的内存区没有被初始化，里面是上次使用之后的一段随机位序列。当我们写：
​	`Image *ptr = new (arena) Image("Quasimodo")`
时，没有新的内存被分配。相反，ptr 被赋值为与 arena 相关联的地址，通过 ptr，内存被解释为一个 Image 类对象。然而，虽然没有分配内存，但是构造函数被应用在现有的存储区上。实际上，定位 new 操作符允许我们在一个特定的、预分配的内存地址上构造一个类对象。 

当完成了 Quasimodo 的图象（image）时，我们或许希望在由 arena 指向的同一个内存位置上操作一个 Esmerelda 的图象（image）。一方面，我们知道怎样做：
​	`Image *ptr = new (arena) Image( “Esmerelda” );`
问题是，这样做覆盖了 Quasimodo 的图像，我们已经修改了 Quasimodo 的图像并希望把它存储在磁盘上。一般我们通过 Image 类的析构函数来做到这一点，但是如果应用操作符delete 
​	`delete ptr; //不好：调用析构函数的同时也删除了存储区` 
则除了调用析构函数，我们还删除了底层的堆存储区，这不是我们希望的。我们可以显式地调用 Image 的析构函数：
​	`ptr -> ~ Image();`
底层的存储区可以被后面的定位 new操作符调用继续使用。
尽管 ptr和arena 指向同一个堆存储区没有任何意义，但是，在 arena 上应用 delete 操作符:
​	`delete arena; //没有调用析构函数` 
不会导致调用 Image 的析构函数，因为 arena 的类型是 char*。记住，只有当 delete 表达式中的指针指向一个带有析构函数的类类型时，编译器才会调用析构函数。 

### 12.3 可能出现的程序代码膨胀

毫无疑问，内联析构函数可能时程序代码膨胀的一个源泉，因为它被插入到函数中的每个退出点，以析构每一个活动的局部类对象。例如，在如下代码段中：

```cpp
Account acct ( “Tina Lee” ); 
int swt;
// ... 
switch( swt ) { 
	case 0: return; 
	case 1: // 进行操作 
			return; 
	case 2: // 进行其他操作 
			return; 
	// 等等 
}
```

在每个 return语句之前，析构函数都必须被内联地展开，在 Account 类的析构函数的情况下，由于它的长度较小，所以多次展开的相关开销也较小。但是如果已经发现它确实是一个问题，则解决方案是：或者把析构函数声明为非内联的，或者重新改写程序代码。一种可能的重写方案是在每个case标签中用break语句代替return语句，然后在 switch 语句后面引入一个return语句：

```cpp
// 重写来提供一个返回点 
switch(swt) { 
case 0: break; 
case 1: // 进行操作 
		break; 
case 2: // 进行另一些操作 
		break; 
// 等等 
}  
// 单个返回点 
return;
```

## 13. 确定基类有虚析构函数

设想在一个军事应用程序里，有一个表示敌人目标的类：

```cpp
class enemytarget {
public:
	enemytarget() { ++numtargets; }
	enemytarget(const enemytarget&) { ++numtargets; }
	~enemytarget() { --numtargets; }
	static size_t numberoftargets() { return numtargets; }
	virtual bool destroy(); //摧毁enemytarget对象后返回成功
private:
	static size_t numtargets; //对象计数器
};

//类的静态成员要在类外定义;
//缺省初始化为0
size_t enemytarget::numtargets;
```

敌人的坦克是一种特殊的敌人目标，所以会很自然地想到将它抽象为一个以公有继承方式从enemytarget派生出来的类。因为不但要关心敌人目标的总数，也要关心敌人坦克的总数，所以和基类一样，在派生类里也采用了上面提到的同样的技巧。

```cpp
class enemytank: public enemytarget {
public:
	enemytank() { ++numtanks; }
	enemytank(const enemytank& rhs) : enemytarget(rhs) { ++numtanks; }
	~enemytank() { --numtanks; }
	static size_t numberoftanks() { return numtanks; }
	virtual bool destroy();
private:
	static size_t numtanks; //坦克对象计数器
};
```

最后，假设程序的其他某处用new动态创建了一个enemytank对象，然后用delete删除掉：

```cpp
enemytarget *targetptr = new enemytank;
//...
delete targetptr;
```

这样会发生严重问题，因为c++语言标准关于这个问题的阐述非常清楚：**当通过基类的指针去删除派生类的对象，而基类又没有虚析构函数时，结果将是不可确定的**。实际运行时经常发生的是，派生类的析构函数永远不会被调用。在本例中，这意味着当targetptr 删除时，enemytank的数量值不会改变，那么，敌人坦克的数量就是错的。

如果某个类不包含虚函数，那一般是表示它将不作为一个基类来使用。当一个类不准备作为基类使用时，使析构函数为虚一般是个坏主意。因为它会为类增加一个虚函数表，使得对象的体积翻倍，还有可能降低其可移植性。

所以基本的一条是：**无故的声明虚析构函数和永远不去声明一样是错误的**。实际上，很多人这样总结：**当且仅当类里包含至少一个虚函数的时候才去声明虚析构函数**。

抽象类是准备被用做基类的，基类必须要有一个虚析构函数，纯虚函数会产生抽象类，所以方法很简单：在想要成为抽象类的类里声明一个纯虚析构函数。这里是一个例子：

```cpp
class awov { // awov = "abstract w/o virtuals"
public:
	virtual ~awov() = 0; // 声明一个纯虚析构函数
};
```

这个类有一个纯虚函数，所以它是抽象的，而且它有一个虚析构函数，所以不会产生析构函数问题。但这里还有一件事：必须提供纯虚析构函数的定义：

​	`awov::~awov() {} // 纯虚析构函数的定义`

这个定义是必需的，因为**虚析构函数工作的方式是：最底层的派生类的析构函数最先被调用，然后各个基类的析构函数被调用**。这就是说，即使是抽象类，编译器也要产生对~awov的调用，所以要保证为它提供函数体。如果不这么做，链接器就会检测出来，最后还是得回去把它添上。

注意：**如果声明虚析构函数为inline，将会避免调用它们时产生的开销，但编译器还是必然会在什么地方产生一个此函数的拷贝**。

## 14. 类对象数组的初始化

有一个如下类：

```cpp
class EquipmentPiece {
private:
	int IDNumber;
public:
	EquipmentPiece(int IDNumber) : IDNumber(IDNumber) {};
};
```

以下列出几种初始化的方法：

### 14.1 对象数组

```cpp
int ID1, ID2, ID3;
EquipmentPiece bestPieces[] = { EquipmentPiece(ID1), EquipmentPiece(ID2), EquipmentPiece(ID3) };
```

注意，该类没有定义默认构造函数，所以下面的方法是不允许的：

```cpp
EquipmentPiece bestPieces[10]; //error
EquipmentPiece *bestPieces = new EquipmentPiece[10]; //error
//如果想要这样，类中构造函数可以修改如下
EquipmentPiece(int IDNumber = 0) : IDNumber(IDNumber) {};
```

### 14.2 指针数组

```cpp
typedef EquipmentPiece* PEP; //PEP是个指向EquipmentPiece的指针
PEP bestPieces[10]; //等同于 PEP *bestPieces = new PEP[10], 然后初始化
for(int i = 0; i < 10; i++) {
	bestPieces[i] = new EquipmentPiece( IDNumber );
}
```

注意：要记得将此数组所指的所有对象删除。如果忘了会产生资源泄露。还有就是该方法与对象数组相比需要额外内存用于存放指针(过度使用内存这一问题可以避免，见第三种方法)。

### 14.3 使用 placement new（定位new操作符）

方法是：先为此数组分配raw memory，然后使用 "placement new" 在这块内存上构造 EquipmentPiece objects;

```cpp
//分配足够的raw memory，给一个预备容纳10个EquipmentPiece objects的数组使用
void *rawMemory = operator new(10*sizeof(EquipmentPiece));
//让bestPieces指向此内存，使这块内存被视为一个EquipmentPiece数组
EquipmentPiece *bestPieces = reinterpret_cast<EquipmentPiece*>(rawMemory);
//利用"placement new"构造这块内存中的EquipmentPiece objects。
int IDNumber = 0;
for(int i = 0; i < 10; i++) {
	new (&bestPieces[i]) EquipmentPiece(IDNumber);
}
```

注意：该方法维护比较困难。在数组内对象结束生命时，要以手动方式调用destructors，最后还得调用operator delete释放raw memory。

```cpp
//将bestPieces中对象以构造次序的反序析构掉
for(i = 0; i < 10; i++) {
	bestPieces[i].~EquipmentPiece();
}
//释放raw memory
operator delete (rawMemory);
```

## 15. 成员初始化列表

类对象的构造顺序是这样的：

1. 分配内存，调用构造函数时，隐式／显式的初始化各数据成员；
2. 进入构造函数后在构造函数中执行一般计算。

使用初始化列表有两个原因：

1. **必须这样做**

   如果我们有一个类成员，它本身是一个类或者是一个结构，而且这个成员它只有一个带参数的构造函数，而没有默认构造函数，这时要对这个类成员进行初始化，就必须调用这个类成员的带参数的构造函数，如果没有初始化列表，那么他将无法完成第一步，就会报错。

   ```cpp
   class ABC {
   public:
            ABC(int x,int y,int z);
   private:
            int a;
            int b;
            int c;
   };
   class MyClass {
   public:
           MyClass():abc(1,2,3)...{}
   private:
           ABC abc;
   };
   ```

   因为ABC有了显示的带参数的构造函数，那么他是无法依靠编译器生成无参构造函数的，所以没有三个int型数据，就无法创建ABC的对象。
   ABC类对象是MyClass的成员，想要初始化这个对象abc，那就只能用成员初始化列表，没有其他办法将参数传递给ABC类构造函数。

   另一种情况是这样的：**当类成员中含有一个const对象时，或者是一个引用时，他们也必须要通过成员初始化列表进行初始化，因为这两种对象要在声明后马上初始化，而在构造函数中，做的是对他们的赋值，这样是不被允许的**。

2. **效率要求这样做**

   类对象的构造顺序显示，进入构造函数体后，进行的是计算，是对他们的**赋值操作**，显然，赋值和初始化是不同的，这样就体现出了效率差异，*如果不用成员初始化类表，那么类对自己的类成员分别进行的是一次隐式的默认构造函数的调用，和一次复制操作符的调用*，如果是类对象，这样做效率就得不到保障。

   注意：**构造函数需要初始化的数据成员，不论是否显式的出现在构造函数的成员初始化列表中，都会在该处完成初始化**，并且**初始化的顺序和其在声明时的顺序是一致的，与列表的先后顺序无关**，所以要特别注意，保证两者顺序一致才能真正保证其效率。

   为了说明清楚，假设有这样一个类：

   ```cpp
   class foo {
   private :
   	int a, b;
   };
   ```

   1. `foo() {}` 和 `foo(int i = 0) {}` 都被认为是默认构造函数，因为后者是默认参数。两者不能同时出现。
   2. 构造函数列表的初始化方式不是按照列表的的顺序，而是按照变量声明的顺序。比如foo里面，a在b之前，那么会先构造a再构造b。所以无论 `foo() : a(b + 1),  b(2) {}` 还是 `foo() : b(2), a(b+1) {}` 都不会让a得到期望的值。如果先声明b再声明a则会更好。
   3. 构造函数列表能够对const成员初始化。比如foo里面有一个`int const c;`则`foo(int x) : c(x) {}` 可以让c值赋成x。不过需要注意的是，c必须在每个构造函数（如果有多个）都有值。
   4. 在继承里面，只有初始化列表可以构造父类的private成员。比如说：`class child : public foo{}`，foo里面的构造函数是这样写的：`foo (int x) { a = x; }`
      而在child里面写 `child(int x) { foo(x); }` 是通过不了编译的。只有把父类初始化改为`foo(int x) : a(x) {}` 而子类构造写作 `child (int x) : foo(x){}` 才可以。

**另一篇关于初始化列表的文章：**

C++初始化类的成员，不但可以用构造函数（constructor）完成，而且可以用初始化类成员列表来完成。MFC大量用到此方法。例如有些初学者可能不大理解如下代码：

```cpp
class A {
public:
	int member_var; //成员变量
	A(); //构造函数 
} 
A::A():member_var(0) {}
```

他们觉得这个构造函数的定义应该只能这样写：

```cpp
A::A() {
	member_var=1; 
}
```

其实两种方法都可。但是有些情况下，只能用第一种，而且通常情况下用第一种也会效率高些。

其实，**第一种方法是真正的初始化(initialization)，而在构造函数内实现的“=”操作其实是赋值(assign)**。这两种方法的一切区别从这儿开始。区别大概如下：

1. 我们知道普通变量编译器都会默认的替你初始化。他们既能初始化，也能被赋值的，而常量(const)按照其意思只能被初始化，不能赋值。否则与变量就无区别了。所以常量成员(const member)只能用成员初始化列表来完成他们的“初始化”，而不能在构造函数内为他们“赋值”。
2. 我们知道类的对象的初始化其实就是调用他的构造函数完成，如果没有写构造函数，编译器会为你默认生成一个。如果你自定义了带参数的构造函数，那么编译器将不生成默认构造函数。这样这个类的对象的初始化必须有参数。如果这样的类的对象来做另外某个类的成员，那么为了初始化这个成员，你必须为这个类的对象的构造函数传递一个参数。同样，如果你在包含它的这个类的构造函数里用“=”，其实是为这个对象“赋值”而非“初始化”它。所以一个类里的所有构造函数都是有参数的，那么这样的类如果做为别的类的成员变量，你必须显式的初始化它，你也是只能通过成员初始化列表来完成初始化。

## 16. 按成员初始化与按成员赋值

### 16.1 按成员初始化（与构造函数和拷贝构造函数有关）

用一个类对象初始化另一个类对象，比如：

```cpp
Account oldAcct("Anna Livia Plurabelle"); 
Account newAcct(oldAcct);
```

被称为缺省的按成员初始化（default memberwise initialization），缺省是因为它自动发生，无论我们是否提供显式构造函数，按成员是因为初始化的单元是单个非静态数据成员，而不是对整个类对象的按位拷贝。 例如，Account 类的第一个定义： 

```cpp
class Account { 
public: 
// ...  
private: 
	char *_name; 
	unsigned int _acct_nmbr; 
	double _balance; 
};
```

我们可以认为缺省的 Account 拷贝构造函数被定义如下：

```cpp
inline Account::Account( const Account &rhs ) { 
	_name = rhs._name; 
	_acct_nmbr = rhs._acct_nmbr; 
	_balance = rhs._balance; 
}
```

用一个类对象初始化该类另一个对象 发生在下列程序情况下：

1. 用一个类对象显式地初始化另一个类对象，例如： `Account newAcct(oldAcct);`

2. 把一个类对象作为实参传递给一个函数，例如：

   `extern bool cash_on_hand(Account acct);` 
   `if (cash_on_hand(oldAcct)) ...`

3. 把一个类对象作为一个函数的返回值传递回来，例如：

   `extern Account consolidate_accts( const vector< Account >&)` { 
   ​	`Account final_acct; `
   ​	`...` 
   ​	`return final_acct;` 
   `}`

4. 非空顺序容器类型的定义，例如： 

   `vector<string> svec(5);  //五个 string 拷贝构造函数被调用`

   （在本例中，用 string 缺省构造函数创建一个临时对象，然后通过 string 拷贝构造函数，该临时对象被依次拷贝到vector的五个元素中。）

5. 非空顺序容器类型的定义，例如：

   `svec.push_back( string( "pooh" ));`


对于大多数实际的类定义， 由于考虑到类的安全性以及用法正确性，所以说缺省的按成员初始化是不够的，**最经常出现的情况是一个类的数据成员是一个指向堆内存的指针，并且这块内存将由该类的析构函数删除**，就如Account 类中的_name 成员一样 。

在缺省按成员初始化之后，newAcct.\_name 和 oldAcct.\_name 指向同一个C风格字符串，如果 oldAcct 离开了域，并且 Account 的析构函数被应用在其上，则 newAcct.\_name 现在指向一个被删除了的内存区；另一种情况是 如果newAcct 修改了由\_name 指向的字符串 则 oldAcct也会受到影响，这种指向错误很难跟踪 。

指针”别名 (aliasing) 问题”的一种解决方案是，分配该字符串的第二个拷贝，并初始化 newAcct._name 以指向这份新的拷贝，为实现这一点，我们必须改变 Account 类的缺省按成员初始化，我们通过提供一个显式的拷贝构造函数来做到这一点。   

类的内部语义也可能使缺省的按成员初始化无效，比如前面所解释的，不能有两个Account 类的对象持有同一个帐号，为了保证这一点，我们必须改变 Account 类的缺省按成员初始化，下面是解决这两个问题的拷贝构造函数：

```cpp
inline Account::Account(const Account &rhs) { 
	// 处理指针别名问题 
	_name = new char[strlen(rhs._name)+1]; 
	strcpy( _name, rhs._name ); 
	// 处理帐号惟一性问题 
	_acct_nmbr = get_unique_acct_nmbr(); 
	// ok: 现在可以按成员拷贝 
	_balance = rhs._balance; 
}
```

除了提供拷贝构造函数，另一种替代的方案是完全不允许按成员初始化，这可以通过下列两个步骤实现： 

1. **把拷贝构造函数声明为私有的**，这可以防止按成员初始化发生在程序的任何一个地方（除了类的成员函数和友元之外）。  

2. **通过有意不提供一个定义**，但是，我们仍然需要第 1 步中的声明，可以防止在类的成员函数和友元中出现按成员初始化。C++语言不会允许我们阻止类的成员函数和友元访问任何私有类成员，但是通过不提供定义，任何试图调用拷贝构造函数的动作虽然在编译系统中是合法的，但是会产生链接错误， 因为无法为它找到可解析的定义。  例如，为了不允许 Account 类的按成员初始化 我们必须如下声明该类：

   ```cpp
   class Account { 
   public: 
   	Account(); 
   	Account(const char*, double=0.0); 
   // ... 
   private: 
   	Account(const Account&); 
   // ... 
   };
   ```

### 16.2 成员类对象的初始化

把 C风格字符串的\_name 声明，替换成 string 类类型的\_name 声明，会发生什么变化？

缺省的按成员初始化依次检查每个成员，如果成员是内置或复合数据类型，则直接执行从成员到成员的初始化。例如，在我们原来的Account 类定义中，因为_name 是一个指针，所以它直接被初始化：
​	`newAcct._name = oldAcct._name` 
但是成员类对象的处理则不同，当我们写以下语句时： 
​	`Account newAcct( oldAcct );`
这两个对象就被识别为 Account 类对象，如果 Account 类提供了一个显式的拷贝构造函数则调用它以完成初始化，否则应用缺省的按成员初始化；类似地，当一个成员类对象被识别出来时，则递归应用相同的过程。

在我们的例子中， string 类提供了显式拷贝构造函数，通过调用该拷贝构造函数，_name被初始化。现在我们可以认为缺省Account 拷贝构造函数被定义如下：

```cpp
inline Account::Account( const Account &rhs ) 
{ 
	_acct_nmbr = rhs._acct_nmbr; 
	_balance = rhs._balance; 
	// C++伪代码 
	// 说明调用了一个类成员 
	// 对象的拷贝构造函数 
	_name.string::string(rhs._name); 
}
```

Account 类的缺省按成员初始化过程现在可以正确地处理_name 的分配和释放，但是 拷贝帐号仍然不正确；因此，我们仍然必须提供一个显式的拷贝构造函数，下面的代码不是十分正确。你能看出为什么吗？

```cpp
// 不太对 
inline Account:: Account( const Account &rhs ) 
{ 
	_name = rhs._name; 
	_balance = rhs._balance; 
	_acct_nmbr = get_unique_acct_nmbr(); 
}
```

该实现不完全正确是因为我们没有区分开初始化和赋值，结果，调用的不是string 拷贝构造函数，而是在隐式初始化阶段调用了缺省的 string 构造函数，并且在构造函数体内调用了string 拷贝赋值操作符。修正很简单：

```cpp
inline Account:: Account( const Account &rhs ) : _name( rhs._name ) 
{ 
	_balance = rhs._balance; 
	_acct_nmbr = get_unique_acct_nmbr(); 
}
```

再次强调，真正的工作是在一开始就意识到我们需要提供一个修正 两个实现的结果都是\_name 持有 rhs.\_name 的值， 只不过第一个实现要求做两次重复工作，**一个一般性的规则是：在成员初始化表中初始化所有的成员类对象**。

### 16.3 按成员赋值（与拷贝赋值操作符有关）

缺省的按成员赋值（default memberwise assignment）所处理的是用一个类对象向该类的另一个对象的赋值操作，其机制基本上与缺省的按成员初始化相同；但是它利用了一个隐式的拷贝赋值操作符来取代拷贝构造函数，例如：

​	`newAcct = oldAcct;`

在缺省情况下，用 oldAcct 的相应成员的值依次向 newAcct 的每个非静态成员赋值，在概念上就好像编译器已经生成下列拷贝赋值操作符：

```cpp
inline Account& Account::operator=(const Account &rhs) 
{ 
  _name = rhs._name; 
  _balance = rhs._balance;
  _acct_nmbr = rhs._acct_nmbr; 
}
```

一般来说，如果缺省的按成员初始化对于一个类不合适，则缺省的按成员赋值也不合。例如，对于原来的 Account 类的定义来说，其中\_name 被声明为 char*类型 \_name 和\_acct_nmbr 的按成员赋值就都不合适了。

通过提供一个显式的拷贝赋值操作符的实例，可以改变缺省的按成员赋值，我们在这操作符实例中实现了正确的类拷贝语义，拷贝赋值操作符的一般形式如下：

```cpp
// 拷贝赋值操作符的一般形式 
className& className::operator=(const className &rhs) 
{ 
	// 保证不会自我拷贝 
	if ( this != &rhs ) // 类拷贝语义在这里 
	{...}  
	// 返回被赋值的对象 
	return *this; 
}
```

*应该防止一个类对象向自己赋值， 因为对于（先释放与该对象当前相关的资源，以便分配与被拷贝对象相关的资源）这样的拷贝赋值操作符 拷贝自身尤其不合适*。例如 ，考虑Account拷贝赋值操作符：

```cpp
Account& Account:: operator=( const Account &rhs ) 
{ 
	// 避免向自身赋值 
	if ( this != &rhs ) 
	{ 
  		delete [] _name; 
  		_name = new char[strlen(rhs._name)+1]; 
  		strcpy( _name,rhs._name ); 
  		_balance = rhs._balance; 
  		_acct_nmbr = rhs._acct_nmbr; 
	} 
	return *this; 
}
```

当一个类对象被赋值给该类的另一个对象时，如`newAcct = oldAcct;`，下面几个步骤就会发生： 

1. 检查该类，判断它是否提供了一个显式的拷贝赋值操作符；
2. 如果是， 则检查访问权限，判断是否在这个程序部分它可以被调用；
3. 如果它不能被调用，则会产生一个编译时刻错误，否则，调用它执行赋值操作；
4. 如果该类没有提供显式的拷贝赋值操作符，则执行缺省按成员赋值； 
5. 在缺省按成员赋值下，每个内置类型或复合类型的数据成员被赋值给相应的成员；  
6. 对于每个类成员对象，递归执行1到6步，直到所有内置或复合类型的数据成员都被赋值。

例如，如果我们再次修改 Account 类的定义，使\_name 为一个string 类型的成员类对象，则：`newAcct = oldAcct;`会调用缺省的按成员赋值，就好像编译器为我们生成了下面的拷贝赋值操作符：

```cpp
inline Account& Account:: operator=( const Account &rhs ) 
{ 
	_balance = rhs._balance; 
	_acct_nmbr = rhs._acct_nmbr; 
	// 即使在程序员这个层次上, 
	// 这个调用也是正确的 
	// 等同于简短形式: _name = rhs._name 
	_name.string::operator=( rhs._name ); 
}
```

但是 Account 类对象的缺省按成员赋值仍然不合适，同为_acct_nmbr 成员也被按成员拷贝了，我们仍然必须提供一个显式的拷贝赋值操作符， 但是它以成员类 string 对象的方式来处理 name ：

```cpp
Account& Account:: operator=( const Account &rhs ) 
{ 
	// 避免类对象向自身赋值 
	if(this != &rhs) 
	{ 
  		// 调用 string::operator=(const string& ) 
  		_name = rhs._name; 
  		_balance = rhs._balance; 
	} 
	return *this; 
}
```

*如果希望完全禁止按成员拷贝的行为，那么就需要像禁止按成员初始化一样，将操作符声明为 private，并且不提供实际的定义*。

*一般来说，应该将拷贝构造函数和拷贝赋值操作符视为一个个体单元，因为在我们需要其中一个的时候，往往也需要另外一个；而试图禁止一个的时候，也很可能需要禁止另一个*。

[TOC]