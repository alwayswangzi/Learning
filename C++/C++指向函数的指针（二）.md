# C++指向函数的指针（二）

# 1. 函数指针数组

我们可以声明一个函数指针的数组，例如

`int (*testCases[10]) ();`

将testCases声明为一个拥有10个元素的数组，每个元素都是一个指向函数的指针，该函数没有参数，返回类型为int。

像testCases这样的声明非常难读，因为很难分析出函数类型与声明哪部分有关，在这种情况下使用typedef名字可以使声明更为易读。

`typedef int (*PFV) ();`
`PFV testCases[10];`

由testCases的一个元素引用的函数调用如下：

```cpp
const int size = 10; 
PFV testCases[size]; 
int testResults[size]; 
 
void runtests() 
{ 
	for (int i = 0; i < size; ++i) 
	// 调用一个数组元素
		testResults[i] = testCases[i](); 
}
```

函数指针的数组可以用一个初始化列表来初始化，该表中每个初始值都代表了一个与数组元素类型相同的函数，例如：

```cpp
int lexicoCompare(const string &, const string &); 
int sizeCompare(const string &, const string &); 
 
//PFI2S 表示一个函数指针类型
typedef int (*PFI2S)(const string &, const string & ); 

//声明并初始化一个2个元素的函数指针数组
PFI2S compareFuncs[2] = {lexicoCompare, sizeCompare};
```

**难点**

我们也可以声明指向 compareFuncs 的指针 这种指针的类型是 指向函数指针数组的指针，声明如下：

`PFI2S (*pfCompare)[2] = &compareFuncs;` 

声明可以分解为：  

`(*pfCompare)`解引用操作符 \* 把 pfCompare 声明为指针，后面的[2]表示 pfCompare 是指向两个元素数组的指针：`(\*pfCompare)[2]`

`typedef PFI2S` 表示数组元素的类型，它是指向函数的指针，该函数返回 int 有两个 const string& 型的参数，数组元素的类型与表达式&lexicoCompare的类型相同，也与 compareFuncs 的第一个元素的类型相同，此外，它还可以通过下列语句之一获得:

`compareFuncs[0]; `
`(*pfCompare)[0];`

要通过 pfCompare 调用 lexicoCompare 程序员可用下列语句之一：

`pfCompare[0](string1, string2); // 编写` 
`((*pfCompare)[0])(string1, string2); // 显式`

# 2. 参数和返回类型

现在我们回头看一下开始提出的问题，在那里给出的任务要求我们写一个排序函数，怎样用函数指针写这个函数呢? 因为函数参数可以是函数指针，所以我们把表示所用比较操作的函数指针作为参数传递给排序函数:

`int sort(string *, string *, int (*)(const string &, const string &));`

我们再次用 typedef名字使 sort()的声明更易读，

`typedef int (*PFI2S)(const string &, const string &);`
`int sort(string*, string*, PFI2S);`

因为在多数情况下使用的函数是 lexicoCompare(),所以我们让它成为缺省的函数指针参数，

`int lexicoCompare(const string &, const string &);` 
`int sort(string*, string*, PFI2S = lexicoCompare);`

最终，sort()函数的定义可能像这样：

```cpp
void sort(string *s1, string *s2, PFI2S compare = lexicoCompare) 
{
	//…实现代码
｝
```

**难点**

注意，**除了用作参数类型之外，函数指针也可以被用作函数返回值的类型**，例如：

`int (*ff(int))(int*, int);`

该声明将 ff()声明为一个函数，它有一个 int 型的参数，返回一个指向函数的指针，类型为：

`int (*) (int*, int);`

同样，使用typedef名字可以使声明更容易读懂。例如，下面的typedef PF使得我们能更容易地分解出ff()的返回类型是函数指针，

`typedef int (*PF)(int*, int);` 
`PF ff(int);`

函数不能声明返回一个函数类型，如果是，则产生编译错误。例如，函数ff()不能如下声明：

`typedef int func(int*, int);` 
`func ff(int); //错误: ff()的返同类型为函数类型`