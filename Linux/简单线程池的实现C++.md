# 基于Linux/C++简单线程池的实现

我们知道Java语言对于多线程的支持十分丰富，JDK本身提供了很多性能优良的库，包括ThreadPoolExecutor和ScheduleThreadPoolExecutor等。C++11中的STL也提供了std:thread（然而我还没有看，这里先占个坑）还有很多第三方库的实现。这里我重复“造轮子”的目的还是为了深入理解C++和Linux线程基础概念，主要以学习的目的。

首先，为什么要使用线程池。因为线程的创建、和清理都是需要耗费系统资源的。我们知道Linux中线程实际上是由轻量级进程实现的，相对于纯理论上的线程这个开销还是有的。假设某个线程的创建、运行和销毁的时间分别为T1、T2、T3，当T1+T3的时间相对于T2不可忽略时，线程池的就有必要引入了，尤其是处理数百万级的高并发处理时。线程池提升了多线程程序的性能，因为线程池里面的线程都是现成的而且能够重复使用，我们不需要临时创建大量线程，然后在任务结束时又销毁大量线程。一个理想的线程池能够合理地动态调节池内线程数量，既不会因为线程过少而导致大量任务堆积，也不会因为线程过多了而增加额外的系统开销。

其实线程池的原理非常简单，它就是一个非常典型的生产者消费者同步问题。根据刚才描述的线程池的功能，可以看出线程池至少有两个主要动作，一个是主程序不定时地向线程池添加任务，另一个是线程池里的线程领取任务去执行。且不论任务和执行任务是个什么概念，但是一个任务肯定只能分配给一个线程执行。这样就可以简单猜想线程池的一种可能的架构了：主程序执行入队操作，把任务添加到一个队列里面；池子里的多个工作线程共同对这个队列试图执行出队操作，这里要保证同一时刻只有一个线程出队成功，抢夺到这个任务，其他线程继续共同试图出队抢夺下一个任务。所以在实现线程池之前，我们需要一个队列。这里的生产者就是主程序，生产任务（增加任务），消费者就是工作线程，消费任务（执行、减少任务）。因为这里涉及到多个线程同时访问一个队列的问题，所以我们需要互斥锁来保护队列，同时还需要条件变量来处理主线程通知任务到达、工作线程抢夺任务的问题。

一般来说实现一个线程池主要包括以下4个组成部分：

1.  线程管理器：用于创建并管理线程池。
2.  工作线程：线程池中实际执行任务的线程。在初始化线程时会预先创建好固定数目的线程在池中，这些初始化的线程一般处于空闲状态。
3.  任务接口：每个任务必须实现的接口。当线程池的任务队列中有可执行任务时，被空间的工作线程调去执行（线程的闲与忙的状态是通过互斥量实现的），把任务抽象出来形成一个接口，可以做到线程池与具体的任务无关。
4.  任务队列：用来存放没有处理的任务。提供一种缓冲机制。实现这种结构有很多方法，常用的有队列和链表结构。

代码：

1 thread_pool.h

```cpp
#ifndef __THREAD_POOL_H
#define __THREAD_POOL_H

#include <vector>
#include <string>
#include <pthread.h>

using namespace std;

/*执行任务的类：设置任务数据并执行*/
class CTask {
protected:
	string m_strTaskName;	//任务的名称
  	void* m_ptrData;	//要执行的任务的具体数据

public:
  	CTask() = default;
  	CTask(string &taskName): m_strTaskName(taskName), m_ptrData(NULL) {}
  	virtual int Run() = 0;
  	void setData(void* data);	//设置任务数据
  
  	virtual ~CTask() {}
  	
};

/*线程池管理类*/
class CThreadPool {
private:
  	static vector<CTask*> m_vecTaskList;	//任务列表
  	static bool shutdown;	//线程退出标志
  	int m_iThreadNum;	//线程池中启动的线程数
  	pthread_t *pthread_id;
  
  	static pthread_mutex_t m_pthreadMutex;	//线程同步锁
  	static pthread_cond_t m_pthreadCond;	//线程同步条件变量
  
protected:
  	static void* ThreadFunc(void *threadData);	//新线程的线程回调函数
  	static int MoveToIdle(pthread_t tid);	//线程执行结束后，把自己放入空闲线程中
  	static int MoveToBusy(pthread_t tid);	//移入到忙碌线程中去
  	int Create();	//创建线程池中的线程
  
public:
  	CThreadPool(int threadNum);
  	int AddTask(CTask *task);	//把任务添加到任务队列中
  	int StopAll();	//使线程池中的所有线程退出
  	int getTaskSize();	//获取当前任务队列中的任务数
};

#endif
```

2 thread_pool.cpp

```cpp
#include "thread_pool.h"
#include <cstdio>

void CTask::setData(void* data) {
  	m_ptrData = data;
}

//静态成员初始化
vector<CTask*> CThreadPool::m_vecTaskList;
bool CThreadPool::shutdown = false;
pthread_mutex_t CThreadPool::m_pthreadMutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t CThreadPool::m_pthreadCond = PTHREAD_COND_INITIALIZER;

//线程管理类构造函数
CThreadPool::CThreadPool(int threadNum) {
  	this->m_iThreadNum = threadNum;
  	printf("I will create %d threads.\n", threadNum);
	Create();
}

//线程回调函数
void* CThreadPool::ThreadFunc(void* threadData) {
  	pthread_t tid = pthread_self();
  	while(1)
    {
    	pthread_mutex_lock(&m_pthreadMutex);
      	//如果队列为空，等待新任务进入任务队列
      	while(m_vecTaskList.size() == 0 && !shutdown)
          	pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex);
		
		//关闭线程
		if(shutdown)
		{
			pthread_mutex_unlock(&m_pthreadMutex);
			printf("[tid: %lu]\texit\n", pthread_self());
			pthread_exit(NULL);
		}
		
		printf("[tid: %lu]\trun: ", tid);
		vector<CTask*>::iterator iter = m_vecTaskList.begin();
		//取出一个任务并处理之
		CTask* task = *iter;
		if(iter != m_vecTaskList.end())
		{
			task = *iter;
			m_vecTaskList.erase(iter);
		}
		
		pthread_mutex_unlock(&m_pthreadMutex);
		
		task->Run();	//执行任务
		printf("[tid: %lu]\tidle\n", tid);
		
    }
	
	return (void*)0;
}

//往任务队列里添加任务并发出线程同步信号
int CThreadPool::AddTask(CTask *task) {	
	pthread_mutex_lock(&m_pthreadMutex);	
	m_vecTaskList.push_back(task);	
	pthread_mutex_unlock(&m_pthreadMutex);	
	pthread_cond_signal(&m_pthreadCond);	
	
  	return 0;
}

//创建线程
int CThreadPool::Create() {	
	pthread_id = new pthread_t[m_iThreadNum];
		for(int i = 0; i < m_iThreadNum; i++)
			pthread_create(&pthread_id[i], NULL, ThreadFunc, NULL);
		
	return 0;
}

//停止所有线程
int CThreadPool::StopAll() {	
	//避免重复调用
	if(shutdown)
		return -1;
	printf("Now I will end all threads!\n\n");
	
	//唤醒所有等待进程，线程池也要销毁了
	shutdown = true;
	pthread_cond_broadcast(&m_pthreadCond);
	
	//清楚僵尸
	for(int i = 0; i < m_iThreadNum; i++)
		pthread_join(pthread_id[i], NULL);
	
	delete[] pthread_id;
	pthread_id = NULL;
	
	//销毁互斥量和条件变量
	pthread_mutex_destroy(&m_pthreadMutex);
	pthread_cond_destroy(&m_pthreadCond);
	
	return 0;
}

//获取当前队列中的任务数
int CThreadPool::getTaskSize() {	
	return m_vecTaskList.size();
}
```

3 main.cpp

```cpp
#include "thread_pool.h"
#include <cstdio>
#include <stdlib.h>
#include <unistd.h>

class CMyTask: public CTask {	
public:
	CMyTask() = default;	
	int Run() {
		printf("%s\n", (char*)m_ptrData);
		int x = rand()%4 + 1;
		sleep(x);	
		return 0;
	}
	~CMyTask() {}
};

int main() {
	CMyTask taskObj;
	char szTmp[] = "hello!";
	taskObj.setData((void*)szTmp);
	CThreadPool threadpool(5);	//线程池大小为5
	
	for(int i = 0; i < 10; i++)
		threadpool.AddTask(&taskObj);
	
	while(1) {
		printf("There are still %d tasks need to handle\n", threadpool.getTaskSize());
		//任务队列已没有任务了
		if(threadpool.getTaskSize()==0) {
			//清除线程池
			if(threadpool.StopAll() == -1) {
				printf("Thread pool clear, exit.\n");
				exit(0);
			}
		}
		sleep(2);
      	printf("2 seconds later...\n");
	}	
	return 0;
}
```

4 Makefile

```cmake
CC:= g++
TARGET:= threadpool
INCLUDE:= -I./
LIBS:= -lpthread
# C++语言编译参数  
CXXFLAGS:= -std=c++11 -g -Wall -D_REENTRANT
# C预处理参数
# CPPFLAGS:=
OBJECTS :=thread_pool.o main.o
  
$(TARGET): $(OBJECTS)
	$(CC) -o $(TARGET) $(OBJECTS) $(LIBS)
  
# $@表示所有目标集  
%.o:%.cpp   
	$(CC) -c $(CXXFLAGS) $(INCLUDE) $< -o $@
  
.PHONY : clean
clean:   
	-rm -f $(OBJECTS) $(TARGET)
```

5 输出结果

```sh
I will create 5 threads.
There are still 10 tasks need to handle
[tid: 140056759576320]	run: hello!
[tid: 140056751183616]	run: hello!
[tid: 140056742790912]	run: hello!
[tid: 140056734398208]	run: hello!
[tid: 140056767969024]	run: hello!
2 seconds later...
There are still 5 tasks need to handle
[tid: 140056742790912]	idle
[tid: 140056742790912]	run: hello!
[tid: 140056767969024]	idle
[tid: 140056767969024]	run: hello!
[tid: 140056751183616]	idle
[tid: 140056751183616]	run: hello!
[tid: 140056759576320]	idle
[tid: 140056759576320]	run: hello!
[tid: 140056751183616]	idle
[tid: 140056751183616]	run: hello!
[tid: 140056734398208]	idle
2 seconds later...
There are still 0 tasks need to handle
Now I will end all threads!
2 seconds later...
[tid: 140056734398208]	exit
[tid: 140056767969024]	idle
[tid: 140056767969024]	exit
[tid: 140056759576320]	idle
[tid: 140056759576320]	exit
[tid: 140056751183616]	idle
[tid: 140056751183616]	exit
[tid: 140056742790912]	idle
[tid: 140056742790912]	exit
2 seconds later...
There are still 0 tasks need to handle
Thread pool clear, exit.

```

流程图如下：

[线程池](./线程池.jpg)