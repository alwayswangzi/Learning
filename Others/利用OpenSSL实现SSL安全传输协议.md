OpenSSL是一个开放源代码的SSL协议的安全[算法](http://lib.csdn.net/base/datastructure)库，它采用[C语言](http://lib.csdn.net/base/c)作为开发语言，具备了跨系统的性能。调用OpenSSL的函数就可以很方便地实现一个SSL加密的安全数据传输通道，从而保护客户端和服务器之间数据的安全。

头文件：
\#include <openssl/ssl.h>
\#include <openssl/err.h>

基于OpenSSL的程序都要遵循以下几个步骤：

### 1. OpenSSL初始化

在使用OpenSSL之前，必须进行相应的协议初始化工作，这可以通过下面的函数实现：

void SSLeay_add_ssl_algorithms();

该函数实际是int SSL_library_init(void)，在ssl.h中时行了宏定义。

### 2. 选择会话协议

在利用OpenSSL开始SSL会话之前，需要为客户端和服务器制定本次会话采用的协议，目前能够使用的协议包括TLSv1.0、SSLv2、SSLv3、SSLv2/v3。

SSL_METHOD \*SSLv23_client_method();

需要注意的是，客户端和服务器必须使用相互兼容的协议，否则SSL会话将无法正常进行。

### 3. 创建会话环境

在OpenSSL中创建的SSL会话环境称为CTX，使用不同的协议会话，其环境也不一样的。

申请SSL会话环境的OpenSSL函数是：

SSL_CTX \*SSL_CTX_new(SSL_METHOD \* method);

当SSL会话环境申请成功后，还要根据实际的需要设置CTX的属性，通常的设置是指定SSL握手阶段证书的验证方式和加载自己的证书。

制定证书验证方式的函数是：

int SSL_CTX_set_verify(SSL_CTX \*ctx, int mode, int(\*verify_callback), int(X509_STORE_CTX \*));

为SSL会话环境加载CA证书的函数是：

SSL_CTX_load_verify_location(SSL_CTX \*ctx, const char \*Cafile, const char \*Capath);

为SSL会话加载用户证书的函数是：

SSL_CTX_use_certificate_file(SSL_CTX \*ctx, const char \*file, int type);

为SSL会话加载用户私钥的函数是：

SSL_CTX_use_PrivateKey_file(SSL_CTX \*ctx, const char\* file, int type);

在将证书和私钥加载到SSL会话环境之后，就可以调用下面的函数来验证私钥和证书是否相符：

int SSL_CTX_check_private_key(SSL_CTX \*ctx);

### 4. 建立SSL套接字

SSL套接字是建立在普通的TCP套接字基础之上，在建立SSL套接字时可以使用下面的一些函数：

// 申请一个SSL套接字
SSL \*SSl_new(SSL_CTX \*ctx);

// 绑定读写套接字
int SSL_set_fd(SSL \*ssl, int fd);)

// 绑定只读套接字
int SSL_set_rfd(SSL \*ssl, int fd);

// 绑定只写套接字
int SSL_set_wfd(SSL \*ssl, int fd);

### 5. 完成SSL握手

在成功创建SSL套接字后，客户端应使用函数SSL_connect( )替代传统的函数connect( )来完成握手过程:

int SSL_connect(SSL \*ssl);

而对服务器来讲，则应使用函数SSL_ accept ( )替代传统的函数accept ( )来完成握手过程:

int SSL_accept(SSL \*ssl);

握手过程完成之后，通常需要询问通信双方的证书信息，以便进行相应的验证，这可以借助于下面的函数来实现:

X509 \*SSL_get_peer_certificate(SSL \*ssl);

该函数可以从SSL套接字中提取对方的证书信息，这些信息已经被SSL验证过了。

X509_NAME \*X509_get_subject_name(X509 \*a);

该函数得到证书所用者的名字。

### 6. 进行数据传输

当SSL握手完成之后，就可以进行安全的数据传输了，在数据传输阶段，需要使用SSL_read( )和SSL_write( )来替代传统的read( )和write( )函数，来完成对套接字的读写操作：

int SSL_read(SSL \*ssl, void \*buf, int num);

int SSL_write(SSL \*ssl, const void \*buf, int num);

### 7. 结束SSL通信

当客户端和服务器之间的数据通信完成之后，调用下面的函数来释放已经申请的SSL资源：

//关闭SSL套接字
int SSL_shutdown(SSL \*ssl);

//释放SSL套接字
void SSl_free(SSL \*ssl);

//释放SSL会话环境
void SSL_CTX_free(SSL_CTX \*ctx); 

### OpenSSL编程框架

1.服务器编程框架

```c
meth = SSLv23_server_method();  
ctx = SSL_CTX_new (meth);  
ssl = SSL_new(ctx);  
fd = socket();  
bind();  
listen();  
accept();  
SSL_set_fd(ssl,fd);  
SSL_connect(ssl);  
SSL_read(ssl, buf, sizeof(buf));
```

2.客户端编程框架

```c
meth = SSLv23_client_method();  
ctx = SSL_CTX_new (meth);  
ssl = SSL_new(ctx);  
fd = socket();  
connect();  
SSL_set_fd(ssl,fd);  
SSL_connect(ssl);  
SSL_write(ssl,"Hello world",strlen("Hello World!"));  
```

对程序来说，openssl将整个握手过程用一对函数体现,即客户端的SSL_connect和服务端的SSL_accept。而后的应用层数据交换则用SSL_read和 SSL_write来完成。