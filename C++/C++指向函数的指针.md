# 指向函数的指针

假定我们被要求提供一个如下形式的排序函数 :

`sort( start, end, compare );`

start和end是指向字符串数组中元素的指针，函数 sort()对于start和end之间的数组元素进行排序，compare定义了比较数组中两个字符串的比较操作（提供一个比较的策略的比较函数）。

为简化sort()的用法而又不限制它的灵活性，我们可能希望指定一个缺省的比较函数。

如何来实现呢？compare可能指向不同的排序方法，解决这些需求的一种策略是将第三个参数compare设为函数指针，并由它指定要使用的比较函数。

# 1. 指向函数的指针的类型

怎样声明指向函数的指针呢？用函数指针作为实参的参数会是什么样呢？下面是函数lexicoCompare()的定义，它按字典序比较两个字符串：

```cpp
#include <string>  
int lexicoCompare( const string &s1, const string &s2 ) 
{ 
	return s1.compare(s2); 
} 
```

**函数名不是其类型的一部分，函数的类型只由它的返回值和参数表决定**，指向lexicoCompare()的指针必须指向与lexicoCompare()相同类型的函数：带有相同的返回类型和相同的参数表，让我们试一下：

`int *pf( const string &, const string & ); //喔! 差一点` 

这几乎是正确的，问题是编译器把该语句解释成名为pf的函数的声明，它有两个参数并且返回一个int\*型的指针，参数表是正确的，但是返回值不是我们所希望的解引用操作符(*)应与返回类型关联，所以在这种情况下是与类型名 int 关联，而不是 pf ，要想让解引用操作符与pf关联，括号是必需的：

`int (*pf)( const string &, const string & ); // ok: 正确 `

这个语句声明了pf是一个指向函数的指针，该函数有两个参数和 int 型的返回值，即指向函数的指针，它与lexicoCompare()的类型相同，下列函数与lexicoCompare()类型相同，都可以用pf来指向：

`int sizeCompare( const string &, const string & );` 

但是 calc()和gcd()与前面两个函数的类型不同，不能用Pf来指向： 

`int calc( int , int );` 
`int gcd( int , int );` 

可以如下定义 pfi 它能够指向这两个函数  

`int (*pfi)( int, int );`

**省略号是函数类型的一部分，如果两个函数具有相同的参数表，但是一个函数在参数表末尾有省略号，则它们被视为不同的函数类型，指向这两个函数的指针类型也不同**。例如：

`int printf( const char*, ... );` 
`int strlen( const char* );`

`int (*pfce)( const char*, ... ); // 可以指向 printf()` 
`int (*pfc)( const char* ); // 可以指向 strlen()`

函数返回类型和参数表的不同组合代表了各不相同的函数类型。

# 2. 初始化和赋值

我们知道，**不带下标操作符的数组名会被解释成指向首元素的指针，当一个函数名没被调用操作符修饰时，会被解释成指向该类型函数的指针**，例如，表达式：

`lexicoCompare;`

会被解释成类型

`int (*) ( const string &, const string & );`

的指针。

**将取地址操作符用在函数名字上也可以产生指向该函数类型的指针，因此`lexcioCompare`和`&lexcioCompare`类型相同**，指向函数的指针可以如下被初始化：

`int (*pfi) ( const string &, const string &) = lexicoCompare;`
`int (*pfi2) ( const string &, const string & ) = &lexicoCompare;`

指向函数的指针可以如下被赋值：

`pfi = lexicoCompare;`
`pfi2 = pfi;`

只有当赋值操作符左边指针的参数表和返回类型与右边函数或指针的参数表和返回类型完全匹配时，初始化和赋值才是正确的。如果不匹配，则将产生编译错误消息，**在指向函数类型的指针之间不存在隐式转换**，例如：

```cpp
int calc(int, int);
int (*pfi2s) (const string &, const string &) = 0;
int (*pfi2i) (int, int) = 0;

int main()
{
	pfi2i = calc; //ok
	pfi2s = calc; //错误：类型不匹配
	pfi2s = pfi2i; //错误：类型不匹配
	
	return 0;
}
```

函数指针可以用0来初始化或赋值，以表示指针不指向任何函数。

# 3. 调用

指向函数的指针可以被用来调用它所指向的函数，调用函数时，不需要解引用操作符，无论是函数名直接调用函数，还是用指针间接调用函数，两者的写法都是一样的，例如：

```cpp
#include <iostream>

int min(int *, int);
int (*pf)(int *, int) = min;
const int iaSize = 5;
int ia[iaSize] = {7,4,9,2,5};

int main()
{
	cout << "Direct call: min: " << min(ia, iaSize) << endl;
	cout << "Indirect call: min: " << pf(ia, iaSize) << endl; 
  	return 0;
}

int min(int* ia, int sz) 
{ 
	int minVal = ia[ 0 ];
	for ( int ix = 1; ix < sz; ++ix ) 
		if ( minVal > ia[ ix ] ) 
			minVal = ia[ ix ]; 
  	return minVal; 
}
```

调用

`pf(ia, iaSize);`

也可以用显式的指针符号写出

`(*pf)(ia, iaSize);`

这两种形式产生相同的结果，但是第二种形式让读者更清楚该调用是通过函数指针执行的。

当然，如果函数指针的值为0，则两个调用都导致运行时刻错误。只有已经被初始化或者赋值的指针引用到一个函数，才可以被安全的用来调用另一个函数。