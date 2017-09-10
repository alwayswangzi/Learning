# C# 调用Win32 C++动态链接库



利用C#设计前端显示界面，C++完成后台算法和功能，是现在比较流行的一种桌面软件研发搭配。通常的做法就是C++封装成动态链接库接口，供C#来调用。这种做法最麻烦的是两者之间数据传递的问题，因为C#和C++之间的数据类型是不一样的，而且在实际应用中还存在一些未知的坑。下面就对C#调用C++动态链接库过程中我遇到的部分问题以及解决方案做下小结，分享给大家。

## 1. C++封装DLL

C++代码做字符串加密，然后返回加密后的字符串，代码结构如下：

![img](https://pic3.zhimg.com/v2-4a80ec218c2738a06cee7f35e0098756_b.png)

EncryptString.h

```cpp
#ifndef _EncryptString_h
#define _EncryptString_h

//计算md5
void __stdcall getMd5(const char* m_sourceStr,char* o_dstStr);
#endif
```

main.cpp

```cpp
#include "EncryptString.h"
#include "md5.h"

#include <iostream>
#include <cstdio>
#include <assert.h>

using namespace std;

//计算md5
void __stdcall getMd5(const char* m_sourceStr,char* o_dstStr)
{
	assert(m_sourceStr!=NULL&&o_dstStr!=NULL);
	MD5 md5(m_sourceStr);
	std::string strMd5=md5.md5();
	strcpy(o_dstStr,strMd5.c_str());
}
```

### 1.1 dllexport

定义导出函数有多种方式，比如__declspec(dllexport)声明导出函数，修改头文件如下：

EncryptString.h

```cpp
#ifndef _EncryptString_h
#define _EncryptString_h

#ifdef ENCRYPT_EXPORTS  
#define ENCRYPT_EXPORTS __declspec(dllexport)  
#else  
#define ENCRYPT_EXPORTS __declspec(dllimport)  
#endif

//计算md5
ENCRYPT_EXPORTS void __stdcall getMd5(const char* m_sourceStr,char* o_dstStr);
#endif

```

编译链接，生成DLL。利用dumpbin查看下导出函数的信息：

![img](https://pic1.zhimg.com/v2-8b8fe5f0f8e7e11e21b5e24e8c77d54c_b.png)

函数进行了重命名，主要是因为C++支持函数重载，因此编译器在编译代码的过程中会把函数的参数类型也加入到函数命名中，导致导出函数的名称发生了变化，这给代码调用带来了不便。为了避免出现重命名，比较常见的做法是告诉编译器按照C语言的风格来编译代码：

EncryptString.h

```cpp
#ifndef _EncryptString_h
#define _EncryptString_h

#ifdef ENCRYPT_EXPORTS  
#define ENCRYPT_EXPORTS __declspec(dllexport)  
#else  
#define ENCRYPT_EXPORTS __declspec(dllimport)  
#endif

#ifdef __cplusplus
extern "C" {
#endif
	ENCRYPT_EXPORTS void __stdcall getMd5(const char* m_sourceStr,char* o_dstStr); 
#ifdef __cplusplus
}
#endif

#endif

```

### 1.2 模块定义文件.def

除了上面通过dllexport方式定义导出函数外，也可以通过设置模块定义文件来实现。

main.def

```
LIBRARY
EXPORTS
	getMd5

```

指定DLL项目的模块定义文件：

![img](https://pic1.zhimg.com/v2-ae8ccc2bf72c173d218c197b0efd8d60_b.png)

EncryptString.h

```cpp
#ifndef _EncryptString_h
#define _EncryptString_h

void __stdcall getMd5(const char* m_sourceStr,char* o_dstStr); 

#endif

```

直接定义函数就可以了，不需要添加修饰。这种做法清爽很多。

### 1.3 __stdcall调用规则

前面导出函数用__stdcall进行了声明，指明了函数调用的规则，C#中默认采用的就是这种方式（CallingConvention=CallingConvention.StdCall），使用了P/Invoke调用方法，所以最好在DLL导出函数中显式声明，或者在C#中显式修改调用规则。

## 2. 数据类型对应关系

C#调用C++的动态链接库，最麻烦的就是数据类型对应关系的处理。下面是部分常见基本数据类型的对应关系。

| C++            | 描述      | C#      | 描述     | 字节数                      |
| -------------- | ------- | ------- | ------ | ------------------------ |
| char           | 字符      | sbyte   | 字节     | 1                        |
| usigned char   | 无符号字符   | byte    | 字节     | 1                        |
| wchar_t        | 无符号字符   | char    | 字符     | 2                        |
| bool           | 布尔值     | byte    | 字节     | 1                        |
| short          | 短整型     | short   | 短整型    | 2                        |
| unsigned short | 无符号短整型  | ushort  | 无符号短整型 | 2                        |
| int            | 整型      | int     | 整型     | 4                        |
| unsigned int   | 无符号整型   | uint    | 无符号整型  | 4                        |
| long           | 长整型     | int     | 整型     | 4                        |
| unsigned long  | 无符号长整型  | uint    | 无符号整型  | 4                        |
| float          | 单精度浮点数  | float   | 单精度浮点数 | 4                        |
| double         | 双精度浮点数  | double  | 双精度浮点数 | 8                        |
| long double    | 长双精度浮点数 | decimal | --     | long double-8 decimal-10 |

**C++ DLL导出的接口中不要存在STL类对象，这样很可能会导致程序崩溃，因为模块链接的C++库可能版本不一样。所以在封装DLL时，不要尝试提供std::string这种字符串的参数，应该提供C风格字符串的接口char \*，约定以\0结尾，或者另外传递字符串大小。**

## 3. C#调用DLL（一）

C#对导出函数进行封装：

![img](https://pic1.zhimg.com/v2-09635be30b48fc32141149c035ec47dc_b.png)

Program.cs

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Runtime.InteropServices;

namespace Md5Test
{
    class Program
    {
        [DllImport("EncryptString.dll", CharSet = CharSet.Ansi)]
        public static extern void getMd5(string m_sourceStr,StringBuilder m_DstStr);

        static void Main(string[] args)
        {
            string str = "123456";
            StringBuilder sb=new StringBuilder();
            getMd5(str,sb);
            Console.WriteLine(sb.ToString());
            Console.Read();
        }
    }
}
```

上面通过DllImport来导出我们生成的EncryptString.dll，指定编码方式为ansi（win32 C++中char*对应的编码方式是ansi）。C++中char*与C#中的string对应，但在使用时，可不是这么简单。有这么一条原则：**如果char\*参数在函数内部不发生变化，比如声明为const char*，那么可以对应为string，如果char*参数本身作为返回字符串使用，也就是说参数在函数内部会发生变化，那么可以对应StringBuilder。**

## 4. C#调用DLL（二）

当函数输入参数为字符串char*时，调用函数将其“退化”为一个指针，读取内容直到\0为止，那么C#封装时，可以考虑通过IntPtr来封装。修改如下：

```csharp
[DllImport("EncryptString.dll",EntryPoint="getMd5",CharSet = CharSet.Ansi)]
public static extern void getMd52(IntPtr m_sourceStr, IntPtr m_DstStr);

```

调用过程：

```csharp
//将托管区string复制到非托管区（ansi编码）
IntPtr pSourceStr = Marshal.StringToHGlobalAnsi(str);
//在非托管区动态分配内存
IntPtr pDstStr = Marshal.AllocHGlobal(128);
//写入0
Marshal.WriteByte(pDstStr,0);

getMd52(pSourceStr, pDstStr);
//获取字符串(将非托管区内存复制到托管区并赋值给string)
string strRes = Marshal.PtrToStringAnsi(pDstStr);
Console.WriteLine(strRes.ToString());

//释放非托管区内存
Marshal.FreeHGlobal(pSourceStr);
Marshal.FreeHGlobal(pDstStr);

```

上面利用IntPtr方式重新封装了DLL的导出函数，这种方式比利用string和StringBuilder更加灵活，在我们不知道DLL内部实现过程时，也显得更加安全。所以在实际封装过程中，推荐使用IntPtr。

## 5. 导出结构体

C++与C#中都支持结构体这种复杂的数据类型。前面讲到，在导出函数中，尽可能使用基本的数据类型，如果结构体是基本数据类型的一个集合的话，我们也可以封装到DLL导出函数中去。

### 5.1 DLL导出函数

![img](https://pic4.zhimg.com/v2-7983ae9eb4c7653c6d0fa6a7646d2023_b.png)

StructDLL.h

```cpp
#define DLL_API extern "C" __declspec(dllexport)

//设置结构体对齐方式
#pragma pack(1)
typedef struct{
    char name[64];
    int age;
    bool male;
    char address[128];
}PERSON;
#pragma pack() 

//获取姓名
DLL_API char* __stdcall getName(PERSON* pInfo);

//获取年龄
DLL_API int __stdcall getAge(PERSON* pInfo);

//获取性别
DLL_API bool __stdcall getMale(PERSON* pInfo);

//获取地址
DLL_API char* __stdcall getAddress(PERSON* pInfo);

//克隆结构体
DLL_API void __stdcall clonePerson(PERSON* pInfo,PERSON* outInfo);

```

main.cpp

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>
#include "StructDLL.h"

char* __stdcall getName(PERSON* pInfo)
{
	return pInfo->name;
};

int __stdcall getAge(PERSON* pInfo)
{
	return pInfo->age;
};

bool __stdcall getMale(PERSON* pInfo)
{
	return false;
};

char* __stdcall getAddress(PERSON* pInfo)
{
	return pInfo->address;
};

void __stdcall clonePerson(PERSON* pInfo,PERSON* outInfo)
{
	assert(pInfo!=NULL&&outInfo!=NULL);
	sprintf_s(outInfo->address,pInfo->address,128);
	sprintf_s(outInfo->name,pInfo->name,128);
	outInfo->age=pInfo->age;
	outInfo->male=pInfo->male;
};

```

### 5.2 结构体引用实现方式

![img](https://pic1.zhimg.com/v2-e76b2d352685d3f0ad1ced1d4b413528_b.png)

C#中封装结构体：

```csharp
[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Ansi,Pack=1)]
public struct PERSON
{
  [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 64)]
  public string name;
  public int age;
  //注意C++中的bool为1个字节，C#可以用byte来描述
  public byte male;
  [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 128)]
  public string address;
}

```

引入DLL中的导出函数。结构体是以传值方式传递，类才是以传地址方式传递，所以我们可以考虑加关键字ref来实现：

```csharp
[DllImport("StructDLL.dll", EntryPoint = "getName", CharSet = CharSet.Ansi)]
public static extern IntPtr getName(ref PERSON pInfo);

[DllImport("StructDLL.dll", EntryPoint = "getAge", CharSet = CharSet.Ansi)]
public static extern int getAge(ref PERSON pInfo);

[DllImport("StructDLL.dll", EntryPoint = "getMale", CharSet = CharSet.Ansi)]
public static extern byte getMale(ref PERSON pInfo);

[DllImport("StructDLL.dll", EntryPoint = "getAddress", CharSet = CharSet.Ansi)]
public static extern IntPtr getAddress(ref PERSON pInfo);

[DllImport("StructDLL.dll", EntryPoint = "clonePerson", CharSet = CharSet.Ansi)]
public static extern void clone(ref PERSON pInfo,ref PERSON outInfo);

```

测试代码：

```csharp
PERSON p1 = new PERSON { 
    name="kikay",
    age=18,
    male=0,
    address="china"
};
PERSON p2 = new PERSON();

//姓名
IntPtr pName = getName(ref p1);
string strName = Marshal.PtrToStringAnsi(pName);

//年龄
int iAge = getAge(ref p1);

//性别
byte blMale = getMale(ref p1);

//地址
IntPtr pAddress = getAddress(ref p1);
string strAddress = Marshal.PtrToStringAnsi(pAddress);

```

结果发现int和byte类型显示正常，但是字符串计算结果有时候显示不正常。为什么呢？其实原因也好分析。以getName为例，传入参数是托管区内存中的结构体对象p1，通过C++代码返回IntPtr，也就是说IntPtr现在指向的是托管区的内存地址，然后调用Marshal.PtrToStringAnsi来将指针转换为string。但是Marshal.PtrToStringAnsi是用来将非托管区内存内容复制到托管区并转换为string的函数，所以可能导致出现不稳定的问题。

可见，如果以ref方式来传递结构体的指针，对于字符串这样的字段，可能会出现乱码等异常。那么我们更彻底一点，直接全部传入IntPtr。

### 5.3 非托管区指针实现方式

重新封装：

```csharp
[DllImport("StructDLL.dll", EntryPoint = "getName", CharSet = CharSet.Ansi)]
public static extern IntPtr getName2(IntPtr pInfo);

[DllImport("StructDLL.dll", EntryPoint = "getAge", CharSet = CharSet.Ansi)]
public static extern int getAge2(IntPtr pInfo);

[DllImport("StructDLL.dll", EntryPoint = "getMale", CharSet = CharSet.Ansi)]
public static extern byte getMale2(IntPtr pInfo);

[DllImport("StructDLL.dll", EntryPoint = "getAddress", CharSet = CharSet.Ansi)]
public static extern IntPtr getAddress2(IntPtr pInfo);

[DllImport("StructDLL.dll", EntryPoint = "clonePerson", CharSet = CharSet.Ansi)]
public static extern void clone2(IntPtr pInfo, IntPtr outInfo);

```

传入参数全部换成了IntPtr。测试代码：

```csharp
PERSON p1 = new PERSON { 
    name="kikay",
    age=18,
    male=0,
    address="china"
};
PERSON p2 = new PERSON();
      
//在非托管区动态分配一片内存，并复制结构体给这片内存
IntPtr pP1 = Marshal.AllocHGlobal(Marshal.SizeOf(typeof(PERSON)));
Marshal.WriteByte(pP1, 0);
Marshal.StructureToPtr(p1, pP1, true);

IntPtr pP2 = Marshal.AllocHGlobal(Marshal.SizeOf(typeof(PERSON)));
Marshal.WriteByte(pP2, 0);
Marshal.StructureToPtr(p2, pP2, true);

IntPtr pName = getName2(pP1);
string strName = Marshal.PtrToStringAnsi(pName);

int iAge = getAge2(pP1);

byte blMale = getMale2(pP1);

IntPtr pAddress = getAddress2(pP1);
string strAddress = Marshal.PtrToStringAnsi(pAddress);


clone2(pP1, pP2);

//将非托管区的内存复制给托管区，并装换为结构体
PERSON p3 = (PERSON)Marshal.PtrToStructure(pP2, typeof(PERSON));

//释放动态分配的内存
Marshal.FreeHGlobal(pP1);
Marshal.FreeHGlobal(pP2);

```

现在一切都正常了，记得释放动态分配的非托管区内存。**建议：当调用结构体类型的变量时，采用IntPtr的方式来处理。**

## 6. 导出C++类的一种技巧

上面讲了如何导出结构体，那么C++中的类呢？相比于结构体，类就要复杂一些。网上讲了一些解决方法，但是总觉得有点繁琐，这里就介绍下自己在实战中总结的一种屡试不爽的小技巧。

DLL中的导出函数主要包括：

1. MyClass* init()：主要用来动态声明一个类对象的指针；
2. doSomething1(MyClass* pClass,void arg1,…)：导出函数1，利用类中的成员函数完成特定的功能1；
3. doSomething2(MyClass* pClass,void arg1,…)：导出函数2，利用类中的成员函数完成特定的功能2；
4. close(MyClass* pClass)：最后关闭函数调用，就是释放动态分配的内存。

将类中相互调用的过程全部在C++的DLL中完成，对外提供的导出函数接口避免了调用类对象的参数，有效回避了类对象参数需要在C#中进行数据类型转换的问题。

![img](https://pic2.zhimg.com/v2-646718b4ff0d88e7a4e0a1b4536ecde5_b.png)

### 6.1 导出函数封装

**1. 需要用到的运算类**

为了更加复杂一点，这里特意设计为模板类。

MyMath.h

```cpp
#ifndef MYMATH_H_
#define MYMATH_H_

template <class T>
class MyMath
{
public:
	//构造函数
	MyMath();
	//析构函数
	~MyMath();
	//加法运算
	int add(const T& a,const T& b);
	//减法运算
	int substract(const T& a,const T& b);
	//数组升序排列
	void sort(T* arr,const int&size);
	//输出运算结果
	char* toString();
private:
	char m_info[32];
};

#define TEMPLATE_DLL
#include "MyMath.cpp"

#endif

```

MyMath.cpp

```cpp
#ifdef TEMPLATE_DLL

#include <iostream>
#include <vector>
#include <algorithm>
#include <cstdio>
#include <cstring>

using namespace std;

//升序排列算法
template<class T>
class myAscCompare
{
public:
    bool operator()(T& t1,T& t2)
    {
        return (t1<t2);
    }
};

template <class T>
MyMath<T>::MyMath()
{
	memset(this->m_info,0,sizeof(m_info));
}

template <class T>
MyMath<T>::~MyMath()
{
}

template <class T>
int MyMath<T>::add(const T& a,const T& b)
{
	memset(this->m_info,0,sizeof(m_info));
	sprintf_s(m_info,"加法运算",32);
	return a+b;
}

template <class T>
int MyMath<T>::substract(const T& a,const T& b)
{
	memset(this->m_info,0,sizeof(m_info));
	sprintf_s(m_info,"减法运算",32);
	return a-b;
}

template <class T>
void MyMath<T>::sort(T* arr,const int&size)
{
	typename std::vector<T>v;
	for(int i=0;i<size;i++)
	{
		v.push_back(arr[i]);
	}
	std::sort(v.begin(),v.end(),myAscCompare<T>());
	for(int i=0;i<size;i++)
	{
		arr[i]=v[i];
	}
	memset(this->m_info,0,sizeof(m_info));
	sprintf_s(m_info,"数组升序排列",32);
}

template <class T>
char* MyMath<T>::toString()
{
	return m_info;
}
#endif

```

**2. 封装导出函数**

ClassDLL.h

```cpp
#ifndef CLASSDLL_H_
#define CLASSDLL_H_

#include "MyMath.h"

#define DLL_API extern "C" __declspec(dllexport) 

//初始化
DLL_API  MyMath<int>* __stdcall InitMyMath();
//加法运算
DLL_API int __stdcall add(MyMath<int>* pMath,const int& a,const int& b);
//减法运算
DLL_API int __stdcall substract(MyMath<int>* pMath,const int& a,const int& b);
//数组升序排列
DLL_API void __stdcall sortArray(MyMath<int>* pMath,int* arr,const int& size);
//输出结果
DLL_API char* __stdcall toString(MyMath<int>* pMath);
//释放
DLL_API void __stdcall CloseMyMath(MyMath<int>* pMath);

#endif

```

main.cpp

```cpp
#include "ClassDLL.h"
#include <limits.h>

MyMath<int>* __stdcall InitMyMath()
{
	MyMath<int>* myMath=new MyMath<int>;
	return myMath;
}

int __stdcall add(MyMath<int>* pMath,const int& a,const int& b)
{
	if(pMath!=NULL)
	{
		return pMath->add(a,b);
	}
	return INT_MIN;
}

int __stdcall substract(MyMath<int>* pMath,const int& a,const int& b)
{
	if(pMath!=NULL)
	{
		return pMath->substract(a,b);
	}
	return INT_MIN;
}

void __stdcall sortArray(MyMath<int>* pMath,int* arr,const int& size)
{
	if(pMath!=NULL)
	{
		pMath->sort(arr,size);
	}
}

char* __stdcall toString(MyMath<int>* pMath)
{
	if(pMath!=NULL)
	{
		return pMath->toString();
	}
	return NULL;
}

void __stdcall CloseMyMath(MyMath<int>* pMath)
{
	if(pMath!=NULL)
	{
		delete pMath;
		pMath=NULL;
	}
}

```

### 6.2 C#调用DLL实现方式

C#封装的调用接口：

```csharp
[DllImport("ClassDLL.dll", EntryPoint = "InitMyMath", CharSet = CharSet.Ansi)]
public static extern IntPtr InitMyMath();

[DllImport("ClassDLL.dll", EntryPoint = "add", CharSet = CharSet.Ansi)]
public static extern int add(IntPtr ptr,ref int a,ref int b);

[DllImport("ClassDLL.dll", EntryPoint = "substract", CharSet = CharSet.Ansi)]
public static extern int substract(IntPtr ptr, ref int a, ref int b);

[DllImport("ClassDLL.dll", EntryPoint = "sortArray", CharSet = CharSet.Ansi)]
public static extern void sortArray(IntPtr ptr, IntPtr arr, ref int size);

[DllImport("ClassDLL.dll", EntryPoint = "toString", CharSet = CharSet.Ansi)]
public static extern IntPtr toString(IntPtr ptr);

[DllImport("ClassDLL.dll", EntryPoint = "CloseMyMath", CharSet = CharSet.Ansi)]
public static extern void CloseMyMath(IntPtr ptr);

```

测试代码：

```csharp
static void Main(string[] args)
{
    //初始化
    IntPtr ptr = InitMyMath();

    int a = 1;
    int b = 101;
    IntPtr pRes;

    //加法运算
    int iAdd = add(ptr, ref a, ref b);
    pRes = toString(ptr);
    Console.WriteLine(Marshal.PtrToStringAnsi(pRes) + "，结果（" + iAdd.ToString() + "）");

    //减法运算
    int iSubsract = substract(ptr,ref a,ref b);
    pRes = toString(ptr);
    Console.WriteLine(Marshal.PtrToStringAnsi(pRes)+"结果（"+iSubsract.ToString()+"）");

    //排序
    int[] iArr={1,2,5,4,6,33,22,1,1,15};
    int iLen = iArr.Length;
    int iSize = Marshal.SizeOf(iArr[0]) * iLen;
    IntPtr pArr = Marshal.AllocHGlobal(iSize);
    Marshal.Copy(iArr, 0, pArr, iLen);
    sortArray(ptr, pArr, ref iLen);
    //还原成数组
    int[] iSorted = new int[iLen];
    Marshal.Copy(pArr, iSorted, 0, iLen);
    StringBuilder strSorted = new StringBuilder();
    for (int i = 0; i < iLen; i++)
    {

        strSorted.Append(iSorted[i].ToString());
        if (i != iLen - 1)
        {
            strSorted.Append(",");
        }
    }
    pRes = toString(ptr);
    Console.WriteLine(Marshal.PtrToStringAnsi(pRes) + "结果（" + strSorted.ToString() + "）");
//释放非托管区分配的内存
    Marshal.FreeHGlobal(pArr);


    //释放
    CloseMyMath(ptr);

    Console.Read();
}
```

运行结果：

![img](https://pic2.zhimg.com/v2-e804e5cb87a51fe8c131d52c1f4602e9_b.png)

## 7 后记

C#调用C++动态链接库过程中遇到的问题肯定远远不止上面的这些，比如处理回调函数的问题等（可以参见我以前转载的一篇博客：[C++通过Callback向C#传递数据**](http://link.zhihu.com/?target=http%3A//blog.csdn.net/kikaylee/article/details/72818632)）。以后遇到新的坑，再和大家一起探讨。