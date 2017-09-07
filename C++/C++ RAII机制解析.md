# C++ RAII机制解析

## 什么是RAII？

RAII是Resource Acquisition Is Initialization（资源获取即初始化）的简称，是C++语言的一种管理资源、避免内存泄漏的惯用手法。

RAII利用的就是C++中构造的对象最终会被销毁的原则，做法是使用一个对象，在其构造时获取对应的资源，在对象生命期内控制对资源的访问，使之始终保持有效，最后在对象析构的时候，释放构造时获取的资源。

## 为什么要使用RAII？

众所周知，一个进程所能拥有的资源是有限的，因此在申请资源使用完之后，一定要及时释放资源，避免造成泄漏。所以在编程的时候new/delete和malloc/free总是匹配操作的。然而在实际的操作之中，由于程序过于庞大或者嵌套过深等原因，程序员经常会忘记释放操作，导致发生错误。并且，即使你将每一个new/delete和malloc/free都匹配了，也会造成程序代码的极度臃肿，效率下降，更可怕的是，程序的可理解性和可维护性明显降低了，当操作增多时，处理资源释放的代码就会越来越多，越来越乱。如果某一个操作发生了异常而导致释放资源的语句没有被调用，怎么办？这个时候，RAII机制就可以派上用场了。

## 如何使用RAII?

当我们在一个函数内部使用局部变量，当退出了这个局部变量的作用域时，这个变量也就被销毁了；当这个变量是类对象时，这个时候，就会自动调用这个类的析构函数，而这一切都是编译器帮你完成的，不要程序员显式的去调用。这个也太好了，RAII就是这样去完成的。

如果把资源用类进行封装起来，对资源操作都封装在类的内部，在析构函数中进行释放资源。当定义的局部变量的生命结束时，它的析构函数就会自动的被调用，如此，就不用程序员显式的去调用释放资源的操作了。

## 一个例子

```cpp
CRITICAL_SECTION cs;	//临界区对象
int gGlobal = 0;	//全局变量

class MyLock	//使用RAII的锁对象
{
public:
    MyLock()
    {
        EnterCriticalSection(&cs);	//申请
    }

    ~MyLock()
    {
        LeaveCriticalSection(&cs);	//释放
    }

private:
    //禁止拷贝构造和赋值运算符
  	MyLock(const MyLock &);	
    MyLock operator =(const MyLock &);
};

void DoComplex(MyLock &lock)
{
  	//TODO...
}

unsigned int __stdcall ThreadFun(void* pv) 
{
    MyLock lock;
    int *para = (int *) pv;

    //使用RAII锁住临界区
    DoComplex(lock);

	//TODO...
    return 0;
}

int main()
{
    InitializeCriticalSection(&cs);

    int thread1, thread2;	
  	//TODO...
    return 0;
}
```

上面的例子中，当多个进程访问临界变量时，为了不出现错误的情况，需要对临界变量进行加锁。但是很多时候，忘记及时解锁导致发生死锁的现象。当我将对CRITICAL_SECTION的访问封装到MyLock类中时，之后，我只需要定义一个MyLock变量，而不必手动的去显示调用LeaveCriticalSection函数。

## 使用RAII可能会遇到的一些陷阱

如果对上述的代码进行一些修改：

```cpp
class MyLock
{
    //...
private:
  	//MyLock(const MyLock &);	
    //MyLock operator =(const MyLock &);
};

void DoComplex(MyLock lock)
{
  	//TODO...
}
```

由于DoComplex函数的参数使用的传值，此时就会发生值的复制，会调用类的复制构造函数，生成一个临时的对象，由于MyLock没有实现复制构造函数，所以就是使用的默认复制构造函数，然后在DoComplex中使用这个临时变量。当调用完成以后，这个临时变量的析构函数就会被调用，由于在析构函数中调用了LeaveCriticalSection，导致了提前离开了CRITICAL_SECTION，从而造成对gGlobal变量访问冲突问题。

为了避免掉进了这个陷阱，同时考虑到封装的是资源，由于资源很多时候是不具备拷贝语义的，所以，在实际实现过程中，MyLock类应该将拷贝构造函数和赋值运算符设置为私有的，这样就防止了背后的资源复制过程，让资源的一切操作都在自己的控制当中。

## 总结

说了这么多，RAII的本质内容是用对象代表资源，把管理资源的任务转化为管理对象的任务，将资源的获取和释放与对象的构造和析构对应起来，从而确保在对象的生存期内资源始终有效，对象销毁时资源一定会被释放。说白了，就是拥有了对象，就拥有了资源，对象在，资源则在。所以，RAII机制是进行资源管理的有力武器，C++程序员依靠RAII写出的代码不仅简洁优雅，而且做到了异常安全。