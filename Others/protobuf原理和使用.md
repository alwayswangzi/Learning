# protobuf原理和使用

## 1. 简介

protobuf全称Google Protocol Buffers，是google开发的的一套用于数据存储，网络通信时用于协议编解码的工具库。它和XML或者JSON差不多，也就是把某种数据结构的信息，以某种格式（XML，JSON）保存起来，protobuf与XML和JSON不同在于，protobuf是基于二进制的。主要用于数据存储、传输协议格式等场合。那既然有了XML等工具，为什么还要开发protobuf呢？主要是因为性能，包括时间开销和空间开销：

1. 时间开销。XML格式化（序列化）和XML解析（反序列化）的时间开销是很大的，在很多时间性能上要求很高的场合，你能做的就是看着XML干瞪眼了。
2. 空间开销。熟悉XML语法的同学应该知道，XML格式为了有较好的可读性，引入了一些冗余的文本信息。所以空间开销也不是太好(应该说是很差，通常需要实际内容好几倍的空间)。据实验，一条消息数据，用protobuf序列化后的大小是json格式的十分之一，xml格式的二十分之一。

除了性能好，代码生成机制也是protobuf吸引人的地方。举个例子，比如有个电子商务的系统（假设用C++实现），其中的模块A需要发送大量的订单信息给模块B，通讯的方式使用socket。

假设订单包括如下属性：

> 时间：			time（用整数表示）
> 客户id：		userid（用整数表示）
> 交易金额：		price（用浮点数表示）
> 交易的描述：	desc（用字符串表示）

如果使用protobuf实现，首先要写一个proto文件（不妨叫Order.proto），在该文件中添加一个名为"Order"的message结构，用来描述通讯协议中的结构化数据。该文件的内容大致如下：

```
message Order
{
  required int32 time = 1;
  required int32 userid = 2;
  required float price = 3;
  optional string desc = 4;
}
```

然后，使用protobuf内置的编译器编译 该proto。由于本例子的模块是C++，你可以通过protobuf编译器的命令行参数，让它生成C++语言的“订单包装类”。（一般来说，一个message结构会生成一个包装类）,然后你使用类似下面的代码来序列化/解析该订单包装类：

```cpp
// 发送方
Order order;
order.set_time(XXXX);
order.set_userid(123);
order.set_price(100.0f);
order.set_desc("a test order");
string sOrder;
order.SerailzeToString(&sOrder);
// 然后调用某种socket的通讯库把序列化之后的字符串发送出去
// ......
// －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
// 接收方
string sOrder;
// 先通过网络通讯库接收到数据，存放到某字符串sOrder
// ......
Order order;
if(order.ParseFromString(sOrder))  // 解析该字符串
{
  cout << "userid:" << order.userid() << endl
       << "desc:" << order.desc() << endl;
}
else
{
  cerr << "parse error!" << endl;
}
```

有了这种代码生成机制，开发人员再也不用吭哧吭哧地编写那些协议解析的代码了（干这种活是典型的吃力不讨好）。

万一将来需求发生变更，要求给订单再增加一个“状态”的属性，那只需要在Order.proto文件中增加一行代码。对于发送方（模块A），只要增加一行设置状态的代码；对于接收方（模块B）只要增加一行读取状态的代码。哇塞，简直太轻松了！

另外，如果通讯双方使用不同的编程语言来实现，使用这种机制可以有效确保两边的模块对于协议的处理是一致的。

## 2. 安装配置

1. 下载protobuf源码

2. 安装protobuf

   ```shell
   tar -xvf ./protobuf-x.x.x.tar.gz
   cd protobuf-x.x.x
   ./configure --prefix=/usr/local/protobuf
   make
   make check
   make install
   ```

3. sudo vim /etc/profile，添加

   ```shell
   export PATH=$PATH:/usr/local/protobuf/bin/
   export PKG_CONFIG_PATH=/usr/local/protobuf/lib/pkgconfig/
   # 保存执行
   source /etc/profile
   ```

   同时 在~/.profile中添加上面两行代码，否则会出现登录用户找不到protoc命令

4. 配置动态链接库路径

   ```shell
   sudo vim /etc/ld.so.conf
   # 插入
   /usr/local/protobuf/lib
   ```

5. 进入root账户，执行ldconfig

6. 写消息文件：msg.proto

   ```
   package lm;   
   message helloworld   
   {   
       required int32     id = 1;  // ID     
       required string    str = 2;  // str    
       optional int32     opt = 3;  //optional field   
   }  
   ```

   将消息文件msg.proto映射成cpp文件

   protoc -I=. --cpp_out=. msg.proto

   可以看到生成了

   msg.pb.h 和msg.pb.cc

7. 序列化例程：write.cpp

   ```cpp
   #include "msg.pb.h"  
   #include <fstream>  
   #include <iostream>  
   using namespace std;  
     
   int main(void)   
   {   
     
       lm::helloworld msg1;   
       msg1.set_id(101);   
       msg1.set_str("hello");   
       fstream output("./log", ios::out | ios::trunc | ios::binary);   
     
       if (!msg1.SerializeToOstream(&output)) {   
           cerr << "Failed to write msg." << endl;   
           return -1;   
       }          
       return 0;   
   }  
   ```

8. 反序列化例程：read.cpp

   ```cpp
   #include "msg.pb.h"  
   #include <fstream>  
   #include <iostream>  
   using namespace std;  
     
   void ListMsg(const lm::helloworld & msg) {    
       cout << msg.id() << endl;   
       cout << msg.str() << endl;   
   }   
     
   int main(int argc, char* argv[]) {   
     
       lm::helloworld msg1;   
     
       {   
           fstream input("./log", ios::in | ios::binary);   
           if (!msg1.ParseFromIstream(&input)) {   
               cerr << "Failed to parse address book." << endl;   
               return -1;   
           }         
       }   
     
       ListMsg(msg1);   
   }  
   ```

9. Makefile文件

   ```cmake
   all: write read  
     
   clean:  
       rm -f write read msg.*.cc msg.*.h *.o log  
     
   proto_msg:  
       protoc --cpp_out=. msg.proto  
       
   write: msg.pb.cc write.cpp  
       g++  msg.pb.cc write.cpp -o write `pkg-config --cflags --libs protobuf`  
     
   read: msg.pb.cc read.cpp  
       g++  msg.pb.cc read.cpp -o read `pkg-config --cflags --libs protobuf` 
   ```

   ​



