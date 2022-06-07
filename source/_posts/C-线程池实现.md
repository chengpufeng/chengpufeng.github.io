---
title: C++线程池实现
tags: C++
date: 2022-06-05 22:00:27
categories: CS笔记
---


# C++线程池实现

<!--more-->

```c++
#include <list>
#include <cstdio>
#include <exception>
#include <pthread.h>
#include "locker.h"
```



## threadpool类定义

```c++
template<typename T>
class threadpool
{
public:
    threadpool(int thread_number = 9, int max_requests = 10000);
    ~threadpool();
    bool append(T* requests);
private:
    //工作线程运行的函数，不断从工作队列中取出任务并执行
    static void* worker(void* arg);
    void run();
private:
    int m_thread_number;
    int m_max_requests;//请求队列中允许的最大请求数
    pthread_t* m_threads;
    std::list<T*> m_workqueue;//请求队列
    locker m_queuelocker;//保护请求队列的互斥锁
    sem m_queuestat;//是否有任务需要处理
    bool m_stop;
};
```



## 构造函数&析构函数

```c++
template<typename T>
threadpool<T>::threadpool(int thread_number, int max_requests):
            m_thread_number(thread_number),m_max_requests(max_requests),m_stop(false),m_threads(NULL)
{
    if((thread_number <= 0) || (max_requests <= 0))
    {
        throw std::exception();
    }
    m_threads = new pthread_t[m_thread_number];
    if(!m_threads)
    {
        throw std::exception();
    }

    for(int i = 0 ;i<thread_number ; i++)
    {
        printf("create the %dth thread\n",i);
        //pthread_create第三个参数必须是静态函数
        //在静态函数中使用类的动态成员：1、通过类的静态对象调用（单例模式）；2、将类的对象作为参数传入静态函数（this）
        if(pthread_create(m_threads + i, NULL, worker, this) != 0)
        {
            delete [] m_threads;
            throw std::exception();
        }
        if(pthread_detach(m_threads[i]))// 线程分离，线程主动与主线程断开关系，线程结束后不会产生僵尸线程，资源自动释放，成功0失败错误码
        {
            delete [] m_threads;
            throw std::exception();
        }
    }
}
template<typename T>
threadpool<T>::~threadpool()
{
    delete [] m_threads;
    m_stop = true;
}
```

`pthread_create`第三个参数必须是静态函数，在静态函数中使用类的动态成员：1、通过类的静态对象调用（单例模式）；2、将类的对象作为参数传入静态函数（this指针）。在创建每个线程后，要进行`pthread_detach`线程分离，线程主动与主线程断开关系，线程结束后不会产生僵尸进程，资源自动释放。

## Append成员函数

```c++
template<typename T>
bool threadpool<T>::append(T* request)
{
    m_queuelocker.lock();
    
    if(m_workqueue.size() > m_max_requests)
    {
        m_queuelocker.unlock();
        return false;
    }
    m_workqueue.push_back(request);

    m_queuelocker.unlock();
    m_queuestat.post();
    return true;
}
```

如果当前工作队列的size已经超过最大请求数，就会直接将请求丢弃。否则请求进工作队列，信号量`m_queuestat`进行post操作。

## worker 成员函数

```c++
template<typename T>
void* threadpool<T>::worker(void* arg)
{
    //arg->this
    threadpool* pool = (threadpool*) arg;
    pool->run();
    return pool;
}
```

## run成员函数

```c++
template<typename T>
void threadpool<T>::run()
{
    while (!m_stop)
    {
        m_queuestat.wait();
        m_queuelocker.lock();

        if(m_workqueue.empty())
        {
            m_queuelocker.unlock();
            continue;
        }
        T* request = m_workqueue.front();
        m_workqueue.pop_front();
        m_queuelocker.unlock();
        if(!request)
        {
            continue;
        }
        //这里可以分别实现ETorLT模式，Procator模式或Reactor模式
        request->process();///模板T
    }
    
}
```

在构造函数中就已经将worker函数绑定到线程池中的各个线程，调用链`threadpool->worker->run`，只要参数`m_stop`是`true`，信号量`m_queuestat`就会一直wait（线程睡眠），当有`request`传入`append`函数，即加入工作队列后，`m_queuestat.post`，`run`就会继续运行，首先对工作队列加锁，判断工作队列是否是空，如果是空直接`continue`进入下一个循环等待任务（可以看成线程看到队列里没任务又回去睡眠了），如果非空从队列中取任务，然后解锁，还需要检查一下`request`是否是空指针，如果空也会回去睡眠。对于能加入工作队列的`T* request`必须实现`process()`。

# locker.h封装

```c++
#include <exception>
#include <pthread.h>
#include <semaphore.h>
```

## sem类定义（构造、析构、wait、post）

```c++
class sem
{
public:
    sem()
    {
        if(sem_init(&m_sem, 0 ,0 ) != 0)
        {
            throw std::exception();
        }
    }
    ~sem()
    {
        sem_destroy(&m_sem);
    }

    bool wait()
    {
        return sem_wait(&m_sem) == 0;
    }
    bool post()
    {
        return sem_post(&m_sem) == 0;
    }
private:
    sem_t m_sem;
};
```


## locker类
```c++
class locker
{
public:
    locker()
    {
        if(pthread_mutex_init(&m_mutex, NULL) != 0)
        {
            throw std::exception();
        }
    }
    ~locker()
    {
        pthread_mutex_destroy(&m_mutex);
    }
    bool lock()
    {
        return pthread_mutex_lock(&m_mutex) == 0;
    }
    bool unlock()
    {
        return pthread_mutex_unlock(&m_mutex) == 0;
    }
private:
    pthread_mutex_t m_mutex;
};
```

## cond 类

```c++
class cond
{
public:
    cond()
    {
        if(pthread_mutex_init(&m_mutex,NULL) != 0)
        {
            throw std::exception();
        }
        if(pthread_cond_init(&m_cond, NULL) != 0)
        {
            pthread_mutex_destroy(&m_mutex);
            throw std::exception();
        }
    }
    ~cond()
    {
        pthread_mutex_destroy(&m_mutex);
        pthread_cond_destroy(&m_cond);
    }
    bool wait()
    {
        int ret = 0;
        pthread_mutex_lock(&m_mutex);
        ret = pthread_cond_wait(&m_cond, &m_mutex);
        pthread_mutex_unlock(&m_mutex);
        return ret == 0;
    }
    bool signal()
    {
        return pthread_cond_signal(&m_cond) == 0;
    }
private:
    pthread_mutex_t m_mutex;
    pthread_cond_t m_cond;
};
```

