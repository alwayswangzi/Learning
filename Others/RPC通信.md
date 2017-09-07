#RPC通信
##1. 概念
###RPC是什么？
RPC 的全称是 Remote Procedure Call 是一种进程间通信方式。它允许程序调用另一个地址空间（通常是共享网络的另一台机器上）的过程或函数，而不用程序员显式编码这个远程调用的细节。即程序员无论是调用本地的还是远程的，本质上编写的调用代码基本相同。
###RPC结构
实现 RPC 的程序包括 5 个部分：

1. User
2. User-stub
3. RPCRuntime
4. Server-stub
5. Server

这 5 个部分的关系如下图所示：

![](http://img.blog.csdn.net/20150108170924203?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbWluZGZsb2F0aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这里user就是client端，当user想发起一个远程调用时，它实际是通过本地调用user-stub。user-stub 负责将调用的接口、方法和参数通过约定的协议规范进行编码并通过本地的RPC Runtime实例传输到远端的实例。远端RPC Runtime实例收到请求后交给server-stub进行解码后发起本地端调用，调用结果再返回给user端。
###RPC的评价标准
- 标准1：真的像本地函数一样调用

RPC的本质是为了屏蔽网络的细节和复杂性，提供易用的api，让用户就像调用本地函数一样实现远程调用，所以RPC最重要的就是“像调用本地函数一样”实现远程调用，完全不让用户感知到底层的网络。真正好用的RPC接口，他的调用形式是和本地函数无差别的，但是本地函数调用是灵活多变的。服务器如果提供和客户端完全一致的调用形式将是非常好用的，这也是RPC框架的一个巨大挑战。

- 标准2：使用简单，用户只需要关注业务即可

RPC的使用简单直接，非常自然，就是和调用本地函数一样，不需要写一大堆额外代码，用户只用写业务逻辑代码，而不用关注框架的细节，其他的事情都由RPC框架完成。

- 标准3：灵活，RPC调用的序列化方式可以自由定制

RPC调用的数据格式支持多种编解码方式，比如一些通用的json格式、msgpack格式或者boost.serialization等格式，甚至支持用户自己定义的格式，这样使用起来才会更灵活。
###RPC开源实现
####谷歌 gRPC
gRPC是谷歌开源的一款跨平台的RPC框架，基于C++实现，支持protobuf和HTTP2协议。
- 协议定义

```
rpc GetFeature(Point) returns (Feature) {}
    message Point {
      int32 latitude = 1;
      int32 longitude = 2;
    }
// Obtains the feature at a given position.
```
####百度 sofa-RPC


