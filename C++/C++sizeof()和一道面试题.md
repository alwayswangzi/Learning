# C++ sizeof()和一道面试题

首先要明确sizeof不是函数，也不是一元运算符，他是个类似宏定义的特殊关键字，sizeof(); 括号内在编译过程中是不被编译的，而是被替代类型。

如`int a=8; sizeof(a); `在编译过程中，它不管a的值是什么，只是被替换成类型sizeof(int); 结果为4。

如果`sizeof(a=6);` 呢，也是一样的转换成a的类型，但是要注意：因为a=6是不被编译的，所以执行完`sizeof(a=6); `a的值还是8，是不变的！

**记住以下几个结论：**

1. unsigned影响的只是最高位bit的意义（正负），数据长度不会被改变的。所以 `sizeof(unsigned int) == sizeof(int)；`

2. 自定义类型的sizeof取值等同于它的类型原形。如 `typedef short WORD; sizeof(short) == sizeof(WORD);`

3. 对函数使用sizeof，在编译阶段会被函数返回值的类型取代。如:

   ```cpp
   int   f1() { return 0; }; 
   cout << sizeof(f1()) << endl;//f1()返回值为int，因此被认为是int 
   ```

4. 对于32位系统来说，只要是指针，大小就是4。如：`cout << sizeof(string*) << endl; // 4`

5. 数组的大小是各维数的乘积*数组元素所属类型的大小。如：

   ```cpp
    char a[] = "abcdef "; 
    int b[20] = {3,4}; 
    char c[2][3] = { "aa","bb"}; 
    cout << sizeof(a) << endl;   // 8 
    cout << sizeof(b) << endl;   // 20*4 
    cout << sizeof(c) << endl;   // 6
   ```

   数组a的大小在定义时未指定，编译时给它分配的空间是按照初始化的值确定的，也就是8，包括末尾不显示的‘\0’。

6. 字符串的sizeof和strlen的区别，见下列示例：

   ```cpp
   /****************************************************
   * Description: siseof 操作符的作用是返回一个对象或类型名的字节长度,它有以下三种形式:
   *	sizeof (type name); 
   *	sizeof (object); 
   *	sizeof object; 
   *
   * Author:charley
   * DateTime:2010-12-11 22:00
   * Compile Environment：win7 32 位 +vs2008
   ****************************************************/
    
   #include <iostream>
   #include <string> 
   #include <cstddef> 
    
   using namespace std;
   int main_test5() 
   { 
       size_t ia; //返回值的类型是 size_t 这是一种与机器相关的typedef定义,在cstddef头文件中
       ia = sizeof(ia); // ok 
       ia = sizeof ia; // ok 
    
       // ia = sizeof int; // 错误 
       ia = sizeof(int); // ok 
       int *pi = new int[12]; 
       cout << "pi: " << sizeof(pi)   //4个字节：指针大小
       << " *pi: " << sizeof(*pi)     //4个字节：数组的第一个元素为int型
       << endl; 
    
       // 一个 string 的大小与它所指的字符串的长度无关 
       string st1( "foobar" ); 
       string st2( "a mighty oak" ); 
       string *ps = &st1; 
       cout << "st1: " << sizeof(st1) //32个字节
       << " st2: " << sizeof(st2)     //32个字节
       << " ps: " << sizeof(ps)       //4个字节：指针大小
       << " *ps: " << sizeof(*ps)     //32个字节，*ps表示字符串内容，相当于sizeof(string)
       << endl; 
       cout << "short :\t" << sizeof(short) << endl;    //2个字节
       cout << "short* :\t"  << sizeof(short*) << endl; //4个字节
       cout << "short& :\t"  << sizeof(short&) << endl; //2个字节
       cout << "short[3] :\t" << sizeof(short[3]) << endl; // 6个字节=每个元素大小2 * 共3个元素
    
       cout << "short :\t" << sizeof(int) << endl;    //4个字节
       cout << "short* :\t"  << sizeof(unsigned int) << endl; //4个字节
    
       char a[] = "abcdef ";
       int b[20] = {3, 4}; 
       char c[2][3] = { "a", "b"}; 
       cout << sizeof(a) << endl;   //   8 = 8 * 1(char大小为1个字节), 8个字符包括一个空字符' '和隐藏的'\0'
       cout << sizeof(b) << endl;   //   80 = 20*4 int大小为4字节
       cout << sizeof(c) << endl;   //   6 = 2*3*1 char大小为1个字节
    
       //字符串的sizeof和strlen的区别
       char aa[] = "abcdef ";    //末尾是空格
       char bb[20] = "abcdef ";  //末尾是空格
       string s = "abcdef ";     //末尾是空格
       cout << strlen(aa) << endl;   //7，字符串长度,包括一个空字符' ',不包括隐藏的'\0'
       cout << sizeof(aa) << endl;   //8 = 8 * 1(char大小为1个字节), 8个字符包括一个空字符' '和隐藏的'\0'
       cout << strlen(bb) <<endl;   //7，字符串长度 ,包括一个空字符' ',不包括隐藏的'\0'
       cout << sizeof(bb) <<endl;   //20 = 20 * 1，字符串容量 
       cout << sizeof(s) <<endl;   //32,这里不代表字符串的长度，而是string类的大小 
        
       //错误: 不能将参数 1 从“std::string”转换为“const char *”
       //cout << strlen(s) << endl;   //   错误！s不是一个字符指针。
         
       aa[1] = '\0 '; 
       cout << strlen(aa) << endl;   // 
       cout << sizeof(aa) << endl;   //8，sizeof是恒定的  
    
       return 0;
   }
   ```

**一道面试题**

题目是要求输出：TrendMicroSoftUSCN ，然后要求修改程序，使程序能输出以上结果。代码如下：

```cpp
#include <iostream> 
#include <string> 
using namespace std; 
int main(int argc, char * argv[]) 
{ 
    string strArr1[] = {"Trend ", "Micro ", "soft "}; 
    string *p=new string[2]; 
    p[0]= "US "; 
    p[1]= "CN "; 
    cout << sizeof(strArr1) << endl; 
    cout << sizeof(p) << endl; 
    cout << sizeof(string) << endl; 
    for(int i=0; i < sizeof(strArr1)/sizeof(string); i++) 
    cout << strArr1[i]; 
    for(i=0;i < sizeof(p)/sizeof(string); i++) 
    cout << p[i]; 
    cout << endl; 
}
```

答案：

如果将

`for(i=0; i <sizeof(p)/sizeof(string);i++)`

改为

`for(i=0; i <sizeof(p)*2/sizeof(string);i++)`

答案也是不对的，sizeof(p)只是指针大小为4，要想求出数组p指向数组的成员个数，应该为

`for(i=0; i <sizeof(*p)*2/sizeof(string);i++)`

为什么？指针p指向数组，则\*p就是指向数组中的成员了，成员的类型是什么，string型，ok那么sizeof(*p)为32，乘以2才是整个数组的大小。

