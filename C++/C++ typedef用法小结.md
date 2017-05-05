# C++ typedef 用法小结

# 四个用途

## 1. 用途一

定义一种类型的别名，而不是简单的宏替换。

可以用于同时声明指针型的多个对象，例如：

`char* pa, pb;` 

这多数不符合我们的意图，它只声明了一个指向字符变量的指针和一个字符变量。若采用以下方法：

`typedef char* PCHAR;`
`PCHAR pa, pb;`

虽然 `char *pa, *pb;` 也可行，但相对来说没有typedef的形式直观，尤其需要大量指针的地方。

## 2. 用途二

用在旧的C代码中，帮助struct的使用。以前的C代码中，声明struct新对象时，必须要带上struct，具体形式为： struct 结构名 对象名，如

```cpp
struct tag1{
	int x;
};
struct tag1 p1; //C
tag1 p1; //C++
```

而在C++中，则可以直接写： 结构名 对象名。

估计某人觉得经常多写一个struct太麻烦了，于是就发明了：

```cpp
typedef struct tag1 {
	int x;
} TAG;
TAG tag1;
```

这样就比原来的方式少写了一个struct，比较省事，尤其在大量使用的时候。或许，在C++中，typedef的这种用途不是很大，但是理解了它，对掌握以前的旧代码还是有帮助的，毕竟我们在项目中有可能会遇到较早些年代遗留下来的代码。

## 3. 用途三

用typedef来定义与平台无关的类型。

比如定义一个叫 REAL 的浮点类型，在目标平台一上，让它表示最高精度的类型为：

`typedef long double REAL;`

在不支持long double的平台上，改为：

`typedef double REAL;`

在连double都不支持的平台上，改为

`typedef float REAL;`

也就是说，当跨平台时，只要改下 typedef 本身就行，不用对其他源码做任何修改。标准库就广泛使用了这个技巧，比如size_t。

另外，因为typedef是定义了一种类型的新别名，不是简单的字符串替换，所以它比宏来得稳健（虽然用宏有时也可以完成以上的用途）。

## 4. 用途四

为复杂的声明定义一个新的简单的别名。

方法是：在原来的声明里逐步用别名替换一部分复杂声明，如此循环，把带变量名的部分留到最后替换，得到的就是原声明的最简化版。举例：

1. 原声明：`int *(*a[5])(int, char*);`

   变量名为a，直接用一个新别名pFun替换a就可以了：

   `typedef int *(*pFun)(int, char*);`

   原声明的最简化版：

   `pFun a[5];`

2. 原声明：`void (*b[10]) (void (*)());`

   变量名为b，先替换右边部分括号里的，pFunParam为别名一：

   `typedef void (*pFunParam)();`

   再替换左边的变量b，pFunx为别名二：

   `typedef void (*pFunx)(pFunParam);`

   原声明的最简化版：

   `pFunx b[10];`

3. 原声明：`doube(*)() (*e)[9];` 

   变量名为e，先替换左边部分，pFuny为别名一：

   `typedef double(*pFuny)();`

   再替换右边的变量e，pFunParamy为别名二

   `typedef pFuny (*pFunParamy)[9];`

   原声明的最简化版：

   `pFunParamy e;`

理解复杂声明可用的“右左法则”：

从变量名看起，先往右，再往左，碰到一个圆括号就调转阅读的方向；括号内分析完就跳出括号，还是按先右后左的顺序，如此循环，直到整个声明分析完。举例：

`int (*func)(int *p);`

首先找到变量名func，外面有一对圆括号，而且左边是一个\*号，这说明func是一个指针；然后跳出这个圆括号，先看右边，又遇到圆括号，这说明(\*func)是一个函数，所以func是一个指向这类函数的指针，即函数指针，这类函数具有int\*类型的形参，返回值类型是int。

`int (*func[5])(int *);`

func右边是一个[]运算符，说明func是具有5个元素的数组；func的左边有一个\*，说明func的元素是指针（注意这里的\*不是修饰func，而是修饰func[5]的，原因是[]运算符优先级比\*高，func先跟[]结合）。跳出这个括号，看右边，又遇到圆括号，说明func数组的元素是函数类型的指针，它指向的函数具有int\*类型的形参，返回值类型为int。

也可以记住2个模式：

type (*)(....) 函数指针 

type (*)[] 数组指针

# 两大陷阱

## 1. 陷阱一

记住，typedef是定义了一种类型的新别名，不同于宏，它不是简单的字符串替换。比如：
先定义：

`typedef char* PSTR;`

然后：

`int mystrcmp(const PSTR, const PSTR);`

const PSTR实际上相当于const char\*吗？不是的，它实际上相当于char\* const。原因在于const给予了整个指针本身以常量性，也就是形成了常量指针char\* const。简单来说，记住当const和typedef一起出现时，typedef不会是简单的字符串替换就行。

## 2. 陷阱二

typedef在语法上是一个存储类的关键字（如auto、extern、mutable、static、register等一样），虽然它并不真正影响对象的存储特性，如：

`typedef static int INT2;` //不可行

编译将失败，会提示“指定了一个以上的存储类”。

以上资料出自：[http://blog.sina.com.cn/s/blog_4826f7970100074k.html](http://blog.sina.com.cn/s/blog_4826f7970100074k.html) 作者：赤龙

# typedef 与 #define的区别

## 1. 案例一

通常讲，typedef要比#define要好，特别是在有指针的场合。请看例子：
```cpp
typedef char *pStr1;
#define pStr2 char *;
pStr1 s1, s2;
pStr2 s3, s4;
```
在上述的变量定义中，s1、s2、s3都被定义为char *，而s4则定义成了char，不是我们所预期的指针变量，根本原因就在于#define只是简单的字符串替换而typedef则是为一个类型起新名字。

## 2. 案例二

下面的代码中编译器会报一个错误，你知道是哪个语句错了吗？

```cpp
typedef char * pStr;
char string[4] = "abc";
const char *p1 = string;
const pStr p2 = string;
p1++;
p2++;
```

是p2++出错了。这个问题再一次提醒我们：typedef和#define不同，它不是简单的文本替换。上述代码中const pStr p2并不等于const char * p2。const pStr p2和const long x本质上没有区别，都是对变量进行只读限制，只不过此处变量p2的数据类型是我们自己定义的而不是系统固有类型而已。因此，const pStr p2的含义是：限定数据类型为char *的变量p2为只读，因此p2++错误。

# 资料：使用typedef抑制劣质代码

作者：Danny Kalev
编译：MTT 工作室

原文出处：Using typedef to Curb Miscreant Code

**摘要：**Typedef 声明有助于创建平台无关类型，甚至能隐藏复杂和难以理解的语法。不管怎样，使用 typedef 能为代码带来意想不到的好处，通过本文你可以学习用 typedef 避免缺欠，从而使代码更健壮。

typedef 声明，简称 typedef，为现有类型创建一个新的名字。比如人们常常使用 typedef 来编写更美观和可读的代码。所谓美观，意指 typedef 能隐藏笨拙的语法构造以及平台相关的数据类型，从而增强可移植性和以及未来的可维护性。本文下面将竭尽全力来揭示 typedef 强大功能以及如何避免一些常见的陷阱。

Q：如何创建平台无关的数据类型，隐藏笨拙且难以理解的语法?

A： 使用 typedefs 为现有类型创建同义字。

**定义易于记忆的类型名**

typedef 使用最多的地方是创建易于记忆的类型名，用它来归档程序员的意图。类型出现在所声明的变量名字中，位于 ''typedef'' 关键字右边。例如：

`typedef int size;`

此声明定义了一个 int 的同义字，名字为 size。注意 typedef 并不创建新的类型。它仅仅为现有类型添加一个同义字。你可以在任何需要 int 的上下文中使用 size：

`void measure(size * psz); `
`size array[4];`
`size len = file.getlength();std::vector <size> vs;` 

typedef 还可以掩饰符合类型，如指针和数组。例如，你不用象下面这样重复定义有 81 个字符元素的数组：

`char line[81];char text[81];`

定义一个 typedef，每当要用到相同类型和大小的数组时，可以这样：

`typedef char Line[81]; Line text, secondline;getline(text);`

同样，可以象下面这样隐藏指针语法：

`typedef char * pstr;int mystrcmp(pstr, pstr);`

这里将带我们到达第一个 typedef 陷阱。标准函数 strcmp()有两个‘const char *’类型的参数。因此，它可能会误导人们象下面这样声明 mystrcmp()：

`int mystrcmp(const pstr, const pstr);` 

这是错误的，按照顺序，‘const pstr’被解释为‘char * const’（一个指向 char 的常量指针），而不是‘const char *’（指向常量 char 的指针）。这个问题很容易解决：

`typedef const char * cpstr; int mystrcmp(cpstr, cpstr);` // 现在是正确的

**记住：**不管什么时候，只要为指针声明 typedef，那么都要在最终的 typedef 名称中加一个 const，以使得该指针本身是常量，而不是对象。

**代码简化**

上面讨论的 typedef 行为有点像 #define 宏，用其实际类型替代同义字。不同点是 typedef 在编译时被解释，因此让编译器来应付超越预处理器能力的文本替换。例如：

`typedef int (*PF) (const char *, const char *);`

这个声明引入了 PF 类型作为函数指针的同义字，该函数有两个 const char * 类型的参数以及一个 int 类型的返回值。如果要使用下列形式的函数声明，那么上述这个 typedef 是不可或缺的：

`PF Register(PF pf);`

Register() 的参数是一个 PF 类型的回调函数，返回某个函数的地址，其署名与先前注册的名字相同。做一次深呼吸。下面我展示一下如果不用 typedef，我们是如何实现这个声明的：

`int (*Register (int (*pf)(const char *, const char *))) (const char *, const char *);` 

很少有程序员理解它是什么意思，更不用说这种费解的代码所带来的出错风险了。显然，这里使用 typedef 不是一种特权，而是一种必需。持怀疑态度的人可能会问：“OK，有人还会写这样的代码吗？”，快速浏览一下揭示** **signal()函数的头文件 <csinal>，一个有同样接口的函数。

**typedef 和存储类关键字（storage class specifier）**

这种说法是不是有点令人惊讶，typedef 就像 auto，extern，mutable，static，和 register 一样，是一个存储类关键字。这并是说 typedef 会真正影响对象的存储特性；它只是说在语句构成上，typedef 声明看起来象 static，extern 等类型的变量声明。下面将带到第二个陷阱：

`typedef register int FAST_COUNTER;` // 错误

编译通不过。问题出在你不能在声明中有多个存储类关键字。因为符号 typedef 已经占据了存储类关键字的位置，在 typedef 声明中不能用 register（或任何其它存储类关键字）。

**促进跨平台开发**

typedef 有另外一个重要的用途，那就是定义机器无关的类型，例如，你可以定义一个叫 REAL 的浮点类型，在目标机器上它可以i获得最高的精度：

`typedef long double REAL;` 

在不支持 long double 的机器上，该 typedef 看起来会是下面这样：

`typedef double REAL; `

并且，在连 double 都不支持的机器上，该 typedef 看起来会是这样：

`typedef float REAL; `

你不用对源代码做任何修改，便可以在每一种平台上编译这个使用 REAL 类型的应用程序。唯一要改的是 typedef 本身。在大多数情况下，甚至这个微小的变动完全都可以通过奇妙的条件编译来自动实现。不是吗? 标准库广泛地使用 typedef 来创建这样的平台无关类型：size_t，ptrdiff 和 fpos_t 就是其中的例子。此外，象 std::string 和 std::ofstream 这样的 typedef 还隐藏了长长的，难以理解的模板特化语法，例如：basic_string<char, char_traits<char>，allocator<char>> 和 basic_ofstream<char, char_traits<char>>。








