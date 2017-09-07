# epoll内部实现

## 1. epoll内部数据结构

epoll之所以高效，是因为内部用了一个**红黑树**记录添加的socket，用了一个**双向链表**接收内核触发的事件，这些都是系统级别的支持的。

当某一进程调用epoll_create方法时，Linux内核会创建一个eventpoll结构体，这个结构体中有两个成员与epoll的使用方式密切相关，eventpoll结构体如下所示：

```cpp
struct eventpoll {
    ....
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
    ....
};
```

参考博文：[linux epoll机制](http://blog.csdn.net/u010657219/article/details/44061629)

每一个epoll对象都有一个独立的eventpoll结构体，用于存放通过epoll_ctl方法向epoll对象中添加进来的事件。这些事件都会挂载在红黑树中。因此，**重复添加的事件就可以通过红黑树而高效的识别出来**（红黑树的插入时间效率是logn）。

而所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系。也就是说，当相应的事件发生时会调用这个回调方法。这个回调方法在内核中叫ep_poll_callback，它会将发生的事件添加到rdlist双链表中。

在epoll中，对于每一个事件，都会建立一个epitem结构体，如下所示：

```cpp
struct epitem {
    struct rb_node rbn;  //红黑树节点
    struct list_head rdllink; //双向链表节点
    struct epoll_filefd ffd;  //事件句柄信息
    struct eventpoll *ep;     //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}
```

下面图的左上角文字写错了，应该是**双向链表的每个节点**都是基于epitem结构中的rdllink成员。 

![img](http://images2015.cnblogs.com/blog/899685/201701/899685-20170107223007628-405780492.png)

## 2. 为什么epoll很高效呢？

上面这一句更具体的解释是（为什么能支持百万句柄）：

1. **不用重复传递**。我们调用epoll_wait时就相当于以往调用select/poll，但是这时却不用传递socket句柄给内核，因为内核已经在epoll_ctl中拿到了要监控的句柄列表。
2. 在内核里，一切皆文件。所以，epoll向内核注册了一个文件系统，用于存储上述的被监控socket。当你调用epoll_create时，就会在这个虚拟的epoll文件系统里创建一个file结点。当然这个file不是普通文件，它只服务于epoll。
3. epoll在被内核初始化时（操作系统启动），同时会开辟出epoll自己的内核高速cache区，用于安置每一个我们想监控的socket，这些socket会以红黑树的形式保存在内核cache里，以支持快速的查找、插入、删除。这个内核高速cache区，就是建立连续的物理内存页，然后在之上建立slab层，简单的说，就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的对象。
4. **极其高效的原因**：这是由于我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。

这个准备就绪list链表是怎么维护的呢？当我们执行epoll_ctl时，除了把socket放到epoll文件系统里file对象对应的红黑树上之外，还会**给内核中断处理程序注册一个回调函数**，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，**当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后就来把socket插入到准备就绪链表里了**。

从上面这句可以看出，epoll的基础就是**回调**呀！ 

如此，**一颗红黑树，一张准备就绪句柄链表，少量的内核cache**，就帮我们解决了大并发下的socket处理问题。执行epoll_create时，创建了红黑树和就绪链表，执行epoll_ctl时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据。执行epoll_wait时立刻返回准备就绪链表里的数据即可。

![img](http://images2015.cnblogs.com/blog/899685/201701/899685-20170102144714222-318866749.png)

## 3. epoll的LT和ET模式

最后看看epoll独有的两种模式LT和ET。无论是LT和ET模式，都适用于以上所说的流程。区别是，LT模式下，只要一个句柄上的事件一次没有处理完，会在以后调用epoll_wait时次次返回这个句柄，而ET模式仅在第一次返回。

![img](http://images2015.cnblogs.com/blog/899685/201701/899685-20170102150324550-1306885424.png)

LT，ET这件事怎么做到的呢？

当一个socket句柄上有事件时，内核会把该句柄插入上面所说的准备就绪list链表，这时我们调用epoll_wait，会把准备就绪的socket拷贝到用户态内存，然后清空准备就绪list链表。最后，epoll_wait干了件事，就是检查这些socket，如果不是ET模式（就是LT模式的句柄了），**并且这些socket上确实有未处理的事件时，又把该句柄放回到刚刚清空的准备就绪链表了**。

所以，非ET的句柄，只要它上面还有事件，epoll_wait每次都会返回这个句柄。（从上面这段，可以看出，LT还有个回放的过程，低效了）