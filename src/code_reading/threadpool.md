# 【源码剖析】threadpool —— 基于 pthread 实现的简单线程池
## 线程池介绍
线程池可以说是项目中经常会用到的组件，在这里假设读者都有一定的多线程基础，如果没有的话不妨在这里进行了解：[POSIX 多线程基础](https://github.com/AngryHacker/ocean/blob/master/multithreaded%20programming/README.md)。

线程池是什么？我的简单理解是有一组预先派生的线程，然后有一个管理员来管理和调度这些线程，你只需不断把需要完成的任务交给他，他就会调度线程的资源来帮你完成。

那么管理员是怎么做的呢？一种简单的方式就是，管理员管理一个任务的队列，如果收到新的任务，就把任务加到队列尾。每个线程盯着队列，如果队列非空，就去队列头拿一个任务来处理（每个任务只能被一个线程拿到），处理完了就继续去队列取任务。如果没有任务了，线程就休眠，直到任务队列不为空。如果这个管理员更聪明一点，他可能会在没有任务或任务少的时候减少线程的数量，任务处理不过来的时候增加线程的数量，这样就实现了资源的动态管理。

那么任务是什么呢？以后台服务器为例，每一个用户的请求就是一个任务，线程不断的在请求队列里取出请求，完成后继续处理下一个请求。

简单图示为：
![threadpool](https://raw.githubusercontent.com/AngryHacker/articles/master/img/20151224173731480.png)

线程池有一个好处就是减少线程创建和销毁的时间，在任务处理时间比较短的时候这个好处非常显著，可以提升任务处理的效率。

## 线程池实现
这里介绍的是线程池的一个简单实现，在创建的时候预先派生指定数量的线程，然后去任务队列取添加进来的任务进行处理就好。

作者说之后会添加更多特性，我们作为学习之后就以这个版本为准就好了。

项目主页：[threadpool](https://github.com/mbrossard/threadpool)
### 数据结构
主要有两个自定义的数据结构
#### `threadpool_task_t`
用于保存一个等待执行的任务。一个任务需要指明：要运行的对应函数及函数的参数。所以这里的 struct 里有函数指针和 void 指针。

```c
typedef struct {
    void (*function)(void *);
    void *argument;
} threadpool_task_t;
```

#### `thread_pool_t`
一个线程池的结构。因为是 C 语言，所以这里任务队列是用数组，并维护队列头和队列尾来实现。
```c
struct threadpool_t {
  pthread_mutex_t lock;     /* 互斥锁 */
  pthread_cond_t notify;    /* 条件变量 */
  pthread_t *threads;       /* 线程数组的起始指针 */
  threadpool_task_t *queue; /* 任务队列数组的起始指针 */
  int thread_count;         /* 线程数量 */
  int queue_size;           /* 任务队列长度 */
  int head;                 /* 当前任务队列头 */
  int tail;                 /* 当前任务队列尾 */
  int count;                /* 当前待运行的任务数 */
  int shutdown;             /* 线程池当前状态是否关闭 */
  int started;              /* 正在运行的线程数 */
};
```

### 函数
#### 对外接口

* `threadpool_t *threadpool_create(int thread_count, int queue_size, int flags);` 创建线程池，用 thread_count 指定派生线程数，queue_size 指定任务队列长度，flags 为保留参数，未使用。
* `int threadpool_add(threadpool_t *pool, void (*routine)(void *),void *arg, int flags);` 添加需要执行的任务。第二个参数为对应函数指针，第三个为对应函数参数。flags 未使用。
* `int threadpool_destroy(threadpool_t *pool, int flags);` 销毁存在的线程池。flags 可以指定是立刻结束还是平和结束。立刻结束指不管任务队列是否为空，立刻结束。平和结束指等待任务队列的任务全部执行完后再结束，在这个过程中不可以添加新的任务。

#### 内部辅助函数

* `static void *threadpool_thread(void *threadpool);` 线程池每个线程所执行的函数。
* `int threadpool_free(threadpool_t *pool);` 释放线程池所申请的内存资源。

## 线程池使用
### 编译
参考项目根目录下的 Makefile, 直接用 `make` 编译。
### 测试用例
项目提供了三个测试用例（见 `threadpool/test/`），我们可以以此来学习线程池的用法并测试是否正常工作。这里提供其中一个：

```c
#define THREAD 32
#define QUEUE  256

#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <assert.h>

#include "threadpool.h"

int tasks = 0, done = 0;
pthread_mutex_t lock;

void dummy_task(void *arg) {
    usleep(10000);
    pthread_mutex_lock(&lock);
    /* 记录成功完成的任务数 */
    done++;
    pthread_mutex_unlock(&lock);
}

int main(int argc, char **argv)
{
    threadpool_t *pool;

    /* 初始化互斥锁 */
    pthread_mutex_init(&lock, NULL);

    /* 断言线程池创建成功 */
    assert((pool = threadpool_create(THREAD, QUEUE, 0)) != NULL);
    fprintf(stderr, "Pool started with %d threads and "
            "queue size of %d\n", THREAD, QUEUE);

    /* 只要任务队列还没满，就一直添加 */
    while(threadpool_add(pool, &dummy_task, NULL, 0) == 0) {
        pthread_mutex_lock(&lock);
        tasks++;
        pthread_mutex_unlock(&lock);
    }

    fprintf(stderr, "Added %d tasks\n", tasks);

    /* 不断检查任务数是否完成一半以上，没有则继续休眠 */
    while((tasks / 2) > done) {
        usleep(10000);
    }
    /* 这时候销毁线程池,0 代表 immediate_shutdown */
    assert(threadpool_destroy(pool, 0) == 0);
    fprintf(stderr, "Did %d tasks\n", done);

    return 0;
}
```

## 源码注释
源码注释一并放在 github, [点我。](https://github.com/AngryHacker/code-with-comments#threadpool)

### threadpool.h

```c
/*
 * Copyright (c) 2013, Mathias Brossard <mathias@brossard.org>.
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions are
 * met:
 *
 *  1. Redistributions of source code must retain the above copyright
 *     notice, this list of conditions and the following disclaimer.
 *
 *  2. Redistributions in binary form must reproduce the above copyright
 *     notice, this list of conditions and the following disclaimer in the
 *     documentation and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
 * "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
 * LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
 * A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
 * HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
 * LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
 * DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
 * THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
 * (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
 * OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 */

#ifndef _THREADPOOL_H_
#define _THREADPOOL_H_

#ifdef __cplusplus
/* 对于 C++ 编译器，指定用 C 的语法编译 */
extern "C" {
#endif

/**
 * @file threadpool.h
 * @brief Threadpool Header File
 */

 /**
 * Increase this constants at your own risk
 * Large values might slow down your system
 */
#define MAX_THREADS 64
#define MAX_QUEUE 65536

/* 简化变量定义 */
typedef struct threadpool_t threadpool_t;

/* 定义错误码 */
typedef enum {
    threadpool_invalid        = -1,
    threadpool_lock_failure   = -2,
    threadpool_queue_full     = -3,
    threadpool_shutdown       = -4,
    threadpool_thread_failure = -5
} threadpool_error_t;

typedef enum {
    threadpool_graceful       = 1
} threadpool_destroy_flags_t;

/* 以下是线程池三个对外 API */

/**
 * @function threadpool_create
 * @brief Creates a threadpool_t object.
 * @param thread_count Number of worker threads.
 * @param queue_size   Size of the queue.
 * @param flags        Unused parameter.
 * @return a newly created thread pool or NULL
 */
/**
 * 创建线程池，有 thread_count 个线程，容纳 queue_size 个的任务队列，flags 参数没有使用
 */
threadpool_t *threadpool_create(int thread_count, int queue_size, int flags);

/**
 * @function threadpool_add
 * @brief add a new task in the queue of a thread pool
 * @param pool     Thread pool to which add the task.
 * @param function Pointer to the function that will perform the task.
 * @param argument Argument to be passed to the function.
 * @param flags    Unused parameter.
 * @return 0 if all goes well, negative values in case of error (@see
 * threadpool_error_t for codes).
 */
/**
 *  添加任务到线程池, pool 为线程池指针，routine 为函数指针， arg 为函数参数， flags 未使用
 */
int threadpool_add(threadpool_t *pool, void (*routine)(void *),
                   void *arg, int flags);

/**
 * @function threadpool_destroy
 * @brief Stops and destroys a thread pool.
 * @param pool  Thread pool to destroy.
 * @param flags Flags for shutdown
 *
 * Known values for flags are 0 (default) and threadpool_graceful in
 * which case the thread pool doesn't accept any new tasks but
 * processes all pending tasks before shutdown.
 */
/**
 * 销毁线程池，flags 可以用来指定关闭的方式
 */
int threadpool_destroy(threadpool_t *pool, int flags);

#ifdef __cplusplus
}
#endif

#endif /* _THREADPOOL_H_ */
```

### threadpool.c
```c
/*
 * Copyright (c) 2013, Mathias Brossard <mathias@brossard.org>.
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions are
 * met:
 *
 *  1. Redistributions of source code must retain the above copyright
 *     notice, this list of conditions and the following disclaimer.
 *
 *  2. Redistributions in binary form must reproduce the above copyright
 *     notice, this list of conditions and the following disclaimer in the
 *     documentation and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
 * "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
 * LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
 * A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
 * HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
 * LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
 * DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
 * THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
 * (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
 * OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 */

/**
 * @file threadpool.c
 * @brief Threadpool implementation file
 */

#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

#include "threadpool.h"

/**
 * 线程池关闭的方式
 */
typedef enum {
    immediate_shutdown = 1,
    graceful_shutdown  = 2
} threadpool_shutdown_t;

/**
 *  @struct threadpool_task
 *  @brief the work struct
 *
 *  @var function Pointer to the function that will perform the task.
 *  @var argument Argument to be passed to the function.
 */
/**
 * 线程池一个任务的定义
 */

typedef struct {
    void (*function)(void *);
    void *argument;
} threadpool_task_t;

/**
 *  @struct threadpool
 *  @brief The threadpool struct
 *
 *  @var notify       Condition variable to notify worker threads.
 *  @var threads      Array containing worker threads ID.
 *  @var thread_count Number of threads
 *  @var queue        Array containing the task queue.
 *  @var queue_size   Size of the task queue.
 *  @var head         Index of the first element.
 *  @var tail         Index of the next element.
 *  @var count        Number of pending tasks
 *  @var shutdown     Flag indicating if the pool is shutting down
 *  @var started      Number of started threads
 */
/**
 * 线程池的结构定义
 *  @var lock         用于内部工作的互斥锁
 *  @var notify       线程间通知的条件变量
 *  @var threads      线程数组，这里用指针来表示，数组名 = 首元素指针
 *  @var thread_count 线程数量
 *  @var queue        存储任务的数组，即任务队列
 *  @var queue_size   任务队列大小
 *  @var head         任务队列中首个任务位置（注：任务队列中所有任务都是未开始运行的）
 *  @var tail         任务队列中最后一个任务的下一个位置（注：队列以数组存储，head 和 tail 指示队列位置）
 *  @var count        任务队列里的任务数量，即等待运行的任务数
 *  @var shutdown     表示线程池是否关闭
 *  @var started      开始的线程数
 */
struct threadpool_t {
  pthread_mutex_t lock;
  pthread_cond_t notify;
  pthread_t *threads;
  threadpool_task_t *queue;
  int thread_count;
  int queue_size;
  int head;
  int tail;
  int count;
  int shutdown;
  int started;
};

/**
 * @function void *threadpool_thread(void *threadpool)
 * @brief the worker thread
 * @param threadpool the pool which own the thread
 */
/**
 * 线程池里每个线程在跑的函数
 * 声明 static 应该只为了使函数只在本文件内有效
 */
static void *threadpool_thread(void *threadpool);

int threadpool_free(threadpool_t *pool);

threadpool_t *threadpool_create(int thread_count, int queue_size, int flags)
{
    if(thread_count <= 0 || thread_count > MAX_THREADS || queue_size <= 0 || queue_size > MAX_QUEUE) {
        return NULL;
    }

    threadpool_t *pool;
    int i;

    /* 申请内存创建内存池对象 */
    if((pool = (threadpool_t *)malloc(sizeof(threadpool_t))) == NULL) {
        goto err;
    }

    /* Initialize */
    pool->thread_count = 0;
    pool->queue_size = queue_size;
    pool->head = pool->tail = pool->count = 0;
    pool->shutdown = pool->started = 0;

    /* Allocate thread and task queue */
    /* 申请线程数组和任务队列所需的内存 */
    pool->threads = (pthread_t *)malloc(sizeof(pthread_t) * thread_count);
    pool->queue = (threadpool_task_t *)malloc
        (sizeof(threadpool_task_t) * queue_size);

    /* Initialize mutex and conditional variable first */
    /* 初始化互斥锁和条件变量 */
    if((pthread_mutex_init(&(pool->lock), NULL) != 0) ||
       (pthread_cond_init(&(pool->notify), NULL) != 0) ||
       (pool->threads == NULL) ||
       (pool->queue == NULL)) {
        goto err;
    }

    /* Start worker threads */
    /* 创建指定数量的线程开始运行 */
    for(i = 0; i < thread_count; i++) {
        if(pthread_create(&(pool->threads[i]), NULL,
                          threadpool_thread, (void*)pool) != 0) {
            threadpool_destroy(pool, 0);
            return NULL;
        }
        pool->thread_count++;
        pool->started++;
    }

    return pool;

 err:
    if(pool) {
        threadpool_free(pool);
    }
    return NULL;
}

int threadpool_add(threadpool_t *pool, void (*function)(void *),
                   void *argument, int flags)
{
    int err = 0;
    int next;

    if(pool == NULL || function == NULL) {
        return threadpool_invalid;
    }

    /* 必须先取得互斥锁所有权 */
    if(pthread_mutex_lock(&(pool->lock)) != 0) {
        return threadpool_lock_failure;
    }

    /* 计算下一个可以存储 task 的位置 */
    next = pool->tail + 1;
    next = (next == pool->queue_size) ? 0 : next;

    do {
        /* Are we full ? */
        /* 检查是否任务队列满 */
        if(pool->count == pool->queue_size) {
            err = threadpool_queue_full;
            break;
        }

        /* Are we shutting down ? */
        /* 检查当前线程池状态是否关闭 */
        if(pool->shutdown) {
            err = threadpool_shutdown;
            break;
        }

        /* Add task to queue */
        /* 在 tail 的位置放置函数指针和参数，添加到任务队列 */
        pool->queue[pool->tail].function = function;
        pool->queue[pool->tail].argument = argument;
        /* 更新 tail 和 count */
        pool->tail = next;
        pool->count += 1;

        /* pthread_cond_broadcast */
        /*
         * 发出 signal,表示有 task 被添加进来了
         * 如果由因为任务队列空阻塞的线程，此时会有一个被唤醒
         * 如果没有则什么都不做
         */
        if(pthread_cond_signal(&(pool->notify)) != 0) {
            err = threadpool_lock_failure;
            break;
        }
        /*
         * 这里用的是 do { ... } while(0) 结构
         * 保证过程最多被执行一次，但在中间方便因为异常而跳出执行块
         */
    } while(0);

    /* 释放互斥锁资源 */
    if(pthread_mutex_unlock(&pool->lock) != 0) {
        err = threadpool_lock_failure;
    }

    return err;
}

int threadpool_destroy(threadpool_t *pool, int flags)
{
    int i, err = 0;

    if(pool == NULL) {
        return threadpool_invalid;
    }

    /* 取得互斥锁资源 */
    if(pthread_mutex_lock(&(pool->lock)) != 0) {
        return threadpool_lock_failure;
    }

    do {
        /* Already shutting down */
        /* 判断是否已在其他地方关闭 */
        if(pool->shutdown) {
            err = threadpool_shutdown;
            break;
        }

        /* 获取指定的关闭方式 */
        pool->shutdown = (flags & threadpool_graceful) ?
            graceful_shutdown : immediate_shutdown;

        /* Wake up all worker threads */
        /* 唤醒所有因条件变量阻塞的线程，并释放互斥锁 */
        if((pthread_cond_broadcast(&(pool->notify)) != 0) ||
           (pthread_mutex_unlock(&(pool->lock)) != 0)) {
            err = threadpool_lock_failure;
            break;
        }

        /* Join all worker thread */
        /* 等待所有线程结束 */
        for(i = 0; i < pool->thread_count; i++) {
            if(pthread_join(pool->threads[i], NULL) != 0) {
                err = threadpool_thread_failure;
            }
        }
        /* 同样是 do{...} while(0) 结构*/
    } while(0);

    /* Only if everything went well do we deallocate the pool */
    if(!err) {
        /* 释放内存资源 */
        threadpool_free(pool);
    }
    return err;
}

int threadpool_free(threadpool_t *pool)
{
    if(pool == NULL || pool->started > 0) {
        return -1;
    }

    /* Did we manage to allocate ? */
    /* 释放线程 任务队列 互斥锁 条件变量 线程池所占内存资源 */
    if(pool->threads) {
        free(pool->threads);
        free(pool->queue);

        /* Because we allocate pool->threads after initializing the
           mutex and condition variable, we're sure they're
           initialized. Let's lock the mutex just in case. */
        pthread_mutex_lock(&(pool->lock));
        pthread_mutex_destroy(&(pool->lock));
        pthread_cond_destroy(&(pool->notify));
    }
    free(pool);
    return 0;
}


static void *threadpool_thread(void *threadpool)
{
    threadpool_t *pool = (threadpool_t *)threadpool;
    threadpool_task_t task;

    for(;;) {
        /* Lock must be taken to wait on conditional variable */
        /* 取得互斥锁资源 */
        pthread_mutex_lock(&(pool->lock));

        /* Wait on condition variable, check for spurious wakeups.
           When returning from pthread_cond_wait(), we own the lock. */
        /* 用 while 是为了在唤醒时重新检查条件 */
        while((pool->count == 0) && (!pool->shutdown)) {
            /* 任务队列为空，且线程池没有关闭时阻塞在这里 */
            pthread_cond_wait(&(pool->notify), &(pool->lock));
        }

        /* 关闭的处理 */
        if((pool->shutdown == immediate_shutdown) ||
           ((pool->shutdown == graceful_shutdown) &&
            (pool->count == 0))) {
            break;
        }

        /* Grab our task */
        /* 取得任务队列的第一个任务 */
        task.function = pool->queue[pool->head].function;
        task.argument = pool->queue[pool->head].argument;
        /* 更新 head 和 count */
        pool->head += 1;
        pool->head = (pool->head == pool->queue_size) ? 0 : pool->head;
        pool->count -= 1;

        /* Unlock */
        /* 释放互斥锁 */
        pthread_mutex_unlock(&(pool->lock));

        /* Get to work */
        /* 开始运行任务 */
        (*(task.function))(task.argument);
        /* 这里一个任务运行结束 */
    }

    /* 线程将结束，更新运行线程数 */
    pool->started--;

    pthread_mutex_unlock(&(pool->lock));
    pthread_exit(NULL);
    return(NULL);
}
```

