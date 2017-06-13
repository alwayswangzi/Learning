# Linux C/C++ gdb详解教程

GDB是GNU开源组织发布的一个强大的UNIX下的程序调试工具。或许，各位比较喜欢那种图形界面方式的，像VC、BCB等IDE的调试，但如果你是在UNIX平台下做软件，你会发现GDB这个调试工具有比VC、BCB的图形化调试器更强大的功能。所谓“寸有所长，尺有所短”就是这个道理。

一般来说，GDB主要帮忙你完成下面四个方面的功能：

1. 启动你的程序，可以按照你的自定义的要求随心所欲的运行程序。
2. 可让被调试的程序在你所指定的调置的断点处停住。（断点可以是条件表达式）
3. 当程序被停住时，可以检查此时你的程序中所发生的事。
4. 动态的改变你程序的执行环境。

从上面看来，GDB和一般的调试工具没有什么两样，基本上也是完成这些功能，不过在细节上，你会发现GDB这个调试工具的强大，大家可能比较习惯了图形化的调试工具，但有时候，命令行的调试工具却有着图形化工具所不能完成的功能。让我们一一看来。

----------------------------------------------------------------

## 1. 一个调试示例：

```cpp
// test.c
#include <stdio.h>

int func(int n)
{
    int sum=0,i;
    for(i=0; i<n; i++)
    {
    	sum+=i;
    }
    return sum;
}

int main()
{
	int i;
    long result = 0;
    for(i=1; i<=100; i++)
    {
    	result += i;
    }

    printf("result[1-100] = %d /n", result );
    printf("result[1-250] = %d /n", func(250) );

  	return 0;
}
```

编译生成执行文件

> gcc -g test.c -o test

使用gdb调试

>$ gdb test	<------启动gdb  
>GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.04) 7.11.1  
>Copyright (C) 2016 Free Software Foundation, Inc.  
>License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>  
>This is free software: you are free to change and redistribute it.  
>There is NO WARRANTY, to the extent permitted by law.  Type "show copying"  
>and "show warranty" for details.  
>This GDB was configured as "x86_64-linux-gnu".

>Type "show configuration" for configuration details.

>For bug reporting instructions, please see:

><http://www.gnu.org/software/gdb/bugs/>.

>Find the GDB manual and other documentation resources online at:

><http://www.gnu.org/software/gdb/documentation/>.

>For help, type "help".

>Type "apropos word" to search for commands related to "word"...

>Reading symbols from test...done.

>(gdb) l		<------列出源代码 

>2	

>3	int func(int n)

>4	{

>5	    int sum=0,i;

>6	    for(i=0; i<n; i++)

>7	    {

>8	    	sum+=i;

>9	    }

>10	    return sum;

>11	}

>(gdb)		<------直接回车表示重复执行上一次的命令 

>12	

>13	int main()

>14	{

>15	    int i;

>16	    long result = 0;

>17	    for(i=1; i<=100; i++)

>18	    {

>19	    	result += i;

>20	    }

>21	

>(gdb) break 16		<------设置断点，在源代码第16行处

>Breakpoint 1 at 0x40055c: file test.c, line 16.

>(gdb) break func	<------设置断点，在函数func入口处

>Breakpoint 2 at 0x40052d: file test.c, line 5.

>(gdb) info break	<------查看断点信息

>Num     Type           Disp Enb Address            What

>1       breakpoint     keep y   0x000000000040055c in main at test.c:16

>2       breakpoint     keep y   0x000000000040052d in func at test.c:5

>(gdb) r		<------运行程序（run）

>Starting program: /home/wzh/test/test 

>Breakpoint 1, main () at test.c:16

>16	    long result = 0;

>(gdb) n		<------单步执行（next）

>17	    for(i=1; i<=100; i++)

>(gdb) n

>19	    	result += i;

>(gdb) n

>17	    for(i=1; i<=100; i++)

>(gdb) n

>19	    	result += i;

>(gdb) c		<------继续执行（continue）

>Continuing.

>Breakpoint 2, func (n=250) at test.c:5

>5	    int sum=0,i;

>(gdb) n

>6	    for(i=0; i<n; i++)

>(gdb) p i	<------打印（print）变量i的值

>$1 = 32767

>(gdb) p sum

>$2 = 0

>(gdb) bt	<------查看函数堆栈

>\#0  func (n=250) at test.c:6

>\#1  0x00000000004005a0 in main () at test.c:23

>(gdb) finish	<------退出函数

>Run till exit from #0  func (n=250) at test.c:6

>0x00000000004005a0 in main () at test.c:23

>23	    printf("result[1-250] = %d /n", func(250) );

>Value returned is $6 = 31125

>(gdb) c

>Continuing.

>result[1-100] = 5050 /nresult[1-250] = 31125 /n[Inferior 1 (process 110450) exited normally]	<------程序退出，调试结束

>(gdb) q		<------退出gdb

## 2. 使用gdb

一般来说gdb主要调试C/C++程序。在编译时，我们必须要把调试信息加入到可执行文件中，使用编译器的-g参数。

> gcc -g test.c

如果没有-g，你将看不见程序的函数名，变量名，所代替的全是运行时的内存地址。

启动gdb的方法有如下几种：

1. gdb <program> ：program就是你的可执行文件
2. gdb <program> core ：用gdb同时调试一个可执行文件和core文件。core文件是程序非法执行后的core dump后产生的文件
3. gdb <program> <PID> ：果你的程序是一个服务程序，那么你可以指定这个服务程序运行时的进程ID。gdb会自动attach上去，并调试他。program应该在PATH环境变量中搜索得到。

gdb启动时，可以加上一些启动开关，详细的开关可以用gdb -help查看。我列出了一些比较常用的参数：

> -symbols/-s <file>

从指定文件中读取符号表。

> -se <file>

从指定文件中读取符号表信息，并把他用在可执行文件中。

> -core/-c <file>

调试时core dump的core文件。

> -directory/-d <directory>

加入一个源文件的搜索路径。默认搜索路径是环境变量中PATH所定义的路径。

