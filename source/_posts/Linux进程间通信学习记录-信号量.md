---
title: Linux进程间通信学习记录_信号量
date: 2026-03-14 21:57:02
tags:
    - C/C++
    - Linux
---

## Linux 进程间信号量

### 信号量是什么，有什么作用

信号量是一种用于进程/线程同步的内核对象，它通过一个整数计数器 `sem_value` 控制对共享资源的访问。其核心机制是：

1. P 操作（也称等待操作）：当进程/线程需要访问共享资源时，会尝试 “等待” 信号量（即执行 `sem_wait` 操作）。若信号量计数器 > 0，则计数器减 1 并继续执行（表示成功获取资源）；若 = 0，则进程/线程阻塞，直到计数器变为正数。

2. V 操作（也称释放操作）：当进程/线程完成对共享资源的访问后，会 “释放” 信号量（即执行 `sem_post` 操作），使计数器加 1（表示释放资源，唤醒阻塞的进程/线程）。

信号量的本质是通过原子操作（不可被中断的操作序列）实现对计数器的修改，从而避免竞态条件。

根据计数器的取值范围，信号量可分为两类：

1. 二进制信号量（Binary Semaphore）：计数器只能取 0 或 1，本质上是 “互斥锁（Mutex）” 的一种泛化，它用于控制对独占资源（一次只能被一个执行单元访问）的访问。与互斥锁不同的是，二进制信号量可以建立一个简单的 “生产者和消费者模型” 来控制多进程/多线程在关键任务上的执行顺序。通过将计数器初始值设为 0 使需要后执行任务的进程/线程阻塞，然后让需要先执行任务的进程/线程在执行完毕后使计数器加 1，需要后执行任务的进程/线程才可以继续执行。

2. 计数信号量（Counting Semaphore）：计数器可取值为任意非负整数，用于控制对有限数量共享资源的访问。例如一个系统有 5 个可用的网络连接，计数信号量初始值为 5，每个进程获取连接时计数器减 1，释放时加 1，确保了最多有 5 个进程同时使用连接。

### Linux 信号量机制

Linux 提供了两种主流信号量实现：System V 信号量（传统机制，历史悠久）和 POSIX 信号量（现代机制，接口更简洁），下面将它们进行一下对比：

| 特性 | System V 信号量 | POSIX 信号量 |
| --- | --- | --- |
| API 复杂度 | 较复杂（基于信号量集）| 简洁（单个信号量操作）|
| 命名方式 | 通过键值（Key）标识 | 通过文件系统路径（如 /my_sem）|
| 作用范围 | 进程间（不直接支持线程）| 进程间（命名信号量）或线程间（无名信号量）|
| 持久化 | 内核持久化（需显式删除）| 内核持久化（命名信号量）或进程内（无名信号量）|
| 原子操作 | 支持对信号量集的批量原子操作 | 单个信号量的原子操作 |
| 错误处理 | 依赖 errno 和返回值 | 依赖 errno 和返回值 |

可以发现 POSIX 信号量接口更直观，支持线程和进程同步，是新代码的首选；而 System V 信号量主要用于维护旧系统。所以下面先说明 POSIX 信号量。

### POSIX 信号量

POSIX 信号量分为 命名信号量（Named Semaphore）和 无名信号量（Unnamed Semaphore），前者用于进程间同步，后者用于同一进程内的线程同步。

1. 命名信号量：通过文件系统路径标识（如 /my_sem），可被多个进程共享。具有内核持久化的特性（相关进程退出后信号量依然存在），需显式删除（sem_unlink）。常用于多个独立进程间的同步（如两个不同的程序）。

2. 无名信号量：没有显性的标识（名称），通过内存共享（如 mmap 或线程间共同持有的内存）在线程间传递。生命周期与共享内存绑定，进程退出后自动销毁。通常仅用于同一进程内的多线程同步。

#### POSIX 信号量相关 API 函数

POSIX 信号量的 API 同样区分为 命名信号量 和 无名信号量 两种，但是这两种信号量只在创建和销毁时具有不同的 API，它们在进行 P/V 操作时的 API 都是相同的（`sem_wait` `sem_post`）。

| 函数 | 功能 | 参数 | 返回值 |
| --- | --- | --- | --- |
| sem_open | 创建/打开命名信号量 | const char *name, int oflag（O_CREAT：若不存在则创建；O_EXCL：若已存在则创建失败）, mode_t mode, unsigned int valu | 成功返回 sem_t *，失败返回 `SEM_FAILED` |
| sem_close | 关闭信号量（减少引用计数）| sem_t *sem | 成功返回 0，失败返回 -1（并设置 errno）|
| sem_unlink | 删除命名信号量（引用计数为 0 时销毁）| const char *name | 成功返回 0，失败返回 -1 |
| sem_wait | 等待信号量（P 操作：计数器减 1，若 0 则阻塞）| sem_t *sem | 成功返回 0，失败返回 -1（如被信号中断返回 `EINTR`）|
| sem_post | 释放信号量（V 操作：计数器加 1，唤醒阻塞者）| sem_t *sem | 成功返回 0，失败返回 -1 |
| sem_init | 初始化无名信号量 | sem_t *sem, int pshared（为 0：线程间共享；非 0：进程间共享，需结合共享内存）, unsigned int value | 成功返回 0，失败返回 -1 |
| sem_destroy | 销毁无名信号量 | sem_t *sem | 成功返回 0，失败返回 -1 |

需要注意：sem_close 函数只会关闭当前进程对信号量的引用，不会影响信号量的状态。而且即使所有进程都关闭了信号量，只要未调用 sem_unlink 函数，信号量仍然存在于系统中，其值保持不变。

##### 命名信号量 API 使用例程：

两个独立进程通过命名信号量实现对共享文件的互斥写入，保证两个进程不会同时写入文件导致写入的数据穿插重叠。

``` c
#include <stdio.h>
#include <stdlib.h>
#include <semaphore.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>

int main()
{
    sem_t *sem;
    int fd;
    const char *sem_name = "/my_mutex_sem"; // 信号量名称（需以 / 开头）。
    const char *file_name = "shared.txt";

    // 在创建信号量前先尝试删除信号量。
    // 这是防止之前的进程没有正常退出，没有正常删除信号量。
    // 通常是由负责创建信号量的进程来执行这一步。
    // 程序开发时用户自行决定让哪个进程来创建信号量。
    sem_unlink(sem_name);

    // 创建并初始化信号量（初始值 1，权限 0644）。
    // 这里创建时没有加入 O_EXCL，因为不知道哪个进程会先创建信号量。
    // 即使有多个进程同时调用 sem_open，也只会有一个进程可以创建信号量并赋予初始值，
    // 其他进程只能打开信号量，不会重复初始化。
    sem = sem_open(sem_name, O_CREAT, 0644, 1);
    if (sem == SEM_FAILED)
    {
        perror("sem_open failed");
        exit(EXIT_FAILURE);
    }

    // 等待信号量（获取互斥锁）。
    if (sem_wait(sem) == -1)
    {
        perror("sem_wait failed");
        sem_close(sem);
        sem_unlink(sem_name);
        exit(EXIT_FAILURE);
    }

    // 写入共享文件（临界区）。
    fd = open(file_name, O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd == -1)
    {
        perror("open failed");
        sem_post(sem);  // 释放信号量再退出。
        sem_close(sem);
        sem_unlink(sem_name);
        exit(EXIT_FAILURE);
    }
    write(fd, "Process write\n", 16);
    close(fd);

    // 释放信号量（解锁）。
    if (sem_post(sem) == -1)
    {
        perror("sem_post failed");
        sem_close(sem);
        sem_unlink(sem_name);
        exit(EXIT_FAILURE);
    }

    // 关闭信号量。
    sem_close(sem);

    // 删除信号量（若为最后一个使用信号量的进程）。
    // 一般是哪个进程负责创建就是哪个进程负责销毁。
    sem_unlink(sem_name);
    return 0;
}
```

##### 无名信号量 API 使用例程：

生产者线程向缓冲区写入数据，消费者线程从缓冲区读取数据，使用计数信号量控制缓冲区的空/满状态和互斥访问。

``` c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <stdlib.h>
#include <unistd.h>
 
#define BUFFER_SIZE 5  // 缓冲区大小。

int buffer[BUFFER_SIZE];
int in = 0, out = 0;  // 生产者/消费者线程操作缓冲区的索引。

// empty：计数信号量，初始值为缓冲区大小（表示空槽数量）。
// full：计数信号量，初始值为 0（表示满槽数量）。
// mutex：二进制信号量（互斥锁），初始值为 1（保护缓冲区操作）。
sem_t empty, full, mutex;
 
// 生产者线程：向缓冲区写入数据。
void *producer(void *arg)
{
    int item;
    for (int i = 0; i < 10; i++)
    {
        item = rand() % 100;  // 生成随机数据。
 
        sem_wait(&empty);     // 等待空槽（empty 减 1）。
        sem_wait(&mutex);     // 加锁缓冲区。
 
        buffer[in] = item;    // 写入缓冲区。
        printf("Producer: Inserted item %d at %d\n", item, in);
        in = (in + 1) % BUFFER_SIZE;
 
        sem_post(&mutex);     // 解锁缓冲区。
        sem_post(&full);      // 满槽加 1（唤醒消费者）。
 
        sleep(1);  // 模拟生产耗时。
    }
    return NULL;
}
 
// 消费者线程：从缓冲区读取数据
void *consumer(void *arg)
{
    int item;
    for (int i = 0; i < 10; i++)
    {
        sem_wait(&full);      // 等待满槽（full 减 1）。
        sem_wait(&mutex);     // 加锁缓冲区。
 
        item = buffer[out];   // 读取缓冲区。
        printf("Consumer: Removed item %d from %d\n", item, out);
        out = (out + 1) % BUFFER_SIZE;
 
        sem_post(&mutex);     // 解锁缓冲区。
        sem_post(&empty);     // 空槽加 1（唤醒生产者）。
 
        sleep(1);  // 模拟消费耗时。
    }
    return NULL;
}
 
int main()
{
    pthread_t prod_tid, cons_tid;
 
    // 初始化信号量。
    // 线程间同步的信号量通常在主线程中完成初始化。
    sem_init(&empty, 0, BUFFER_SIZE);  // 无名信号量，线程间共享（pshared = 0）。
    sem_init(&full, 0, 0);
    sem_init(&mutex, 0, 1);

    // 创建线程。
    pthread_create(&prod_tid, NULL, producer, NULL);
    pthread_create(&cons_tid, NULL, consumer, NULL);

    // 等待线程结束。
    pthread_join(prod_tid, NULL);
    pthread_join(cons_tid, NULL);

    // 销毁无名信号量。
    // 线程间同步的信号量通常在主线程中完成销毁。
    sem_destroy(&empty);
    sem_destroy(&full);
    sem_destroy(&mutex);

    return 0;
}
```

### System V 信号量

待续
