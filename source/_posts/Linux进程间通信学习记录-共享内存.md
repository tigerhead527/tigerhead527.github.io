---
title: Linux进程间通信学习记录_共享内存
date: 2026-03-20 17:38:42
tags:
    - C/C++
    - Linux
---

## Linux 共享内存

### 共享内存是什么，有什么作用

在 Linux 系统中，共享内存是一种高效的进程间通信（IPC）方式，它允许多个进程访问同一块物理内存区域，实现跨进程的数据交换。共享内存的另一个作用在于它避免了多个进程在数据交换时必须先把数据复制到生产者进程外，再把数据复制到消费者进程内的操作，通过共享内存，数据交换无需经过内核中转（如管道、消息队列需通过内核拷贝数据），减少了数据复制的消耗（零拷贝技术）。

共享内存的工作原理基于在物理内存中创建一块内存区域，然后通过页表映射，将这块物理内存映射到多个进程的虚拟地址空间中。这样不同的进程就可以通过操作自己的虚拟地址来访问和操作同一块物理内存，从而实现数据交换。

与其他的 IPC 方式相比较，共享内存具有以下优势：

1. 避免内核与用户空间之间的数据拷贝，性能极佳，是速度最快的 IPC 方式。
2. 数据直接在内存中共享，延迟很低，适合高频通信场景（如实时数据采集）。
3. 支持任意数据结构，无需序列化/反序列化，灵活高效。

但同时，共享内存也具有以下劣势：

1. 共享内存本身不提供同步机制，多进程并发读写易引发竞态条件（Race Condition），需额外同步工具（如信号量、互斥锁）。
2. 需要手动管理内存创建、映射、销毁，资源管理复杂，易导致资源泄漏（如进程崩溃未清理共享内存）。
3. 仅支持同一主机内的进程通信，无法跨网络通信。

### Linux 共享内存机制

Linux 提供了三种主流共享内存实现：System V 共享内存（传统），POSIX 共享内存（现代）和 mmap 匿名共享内存（轻量级），下面将它们进行一下对比：

| 特性 | System V 共享内存 | POSIX 共享内存 | mmap 匿名共享内存 |
| --- | --- | --- | --- |
| 命名方式 | shmid（整数）| 通过文件系统路径（如 /my_shm）| 无名称（仅父子共享）|
| 创建/删除复杂度 | 较高（需 shmctl）| 较低（文件系统接口）| 简单（mmap 一行代码）|
| 作用范围 | 任意进程间 | 任意进程间 | 仅父子进程间 |
| 持久化 | 内核管理，需显式删除 | 文件系统管理，引用计数 | 进程退出后释放 |

下面先说明 POSIX 信号量。

### POSIX 共享内存

#### POSIX 共享内存相关 API 函数

| 函数 | 功能 | 参数 | 返回值 |
| --- | --- | --- | --- |
| shm_open | 创建或打开共享内存 | const char *name, int oflag（O_CREAT：若不存在则创建；O_EXCL：若已存在则创建失败；O_TRUNC：若已存在则将其截断为零字节）, mode_t mode | 成功返回文件描述符，失败返回 -1（并设置 errno）|
| shm_unlink | 删除共享内存 | const char *name | 成功返回 0，失败返回 -1（并设置 errno）|
| ftruncate | 设置共享内存大小 | int fd, off_t length | 成功返回 0，失败返回 -1（并设置 errno）|
| close | 关闭文件描述符 | int fd | 成功返回 0，失败返回 -1（并设置 errno）|
| mmap | 将共享内存映射到进程地址空间 | void *addr, size_t length, int prot, int flags, int fd, off_t offset | 成功返回映射区的内存起始地址，失败返回 -1（MAP_FAILED）（并设置 errno）|
| munmap | 解除映射 | void *addr, size_t length | 成功返回 0，失败返回 -1（并设置 errno）|

需要注意：

1. ftruncate 函数用来重置文件大小。该函数不受限于 shm_open 函数打开的文件，它也可以改变使用 open 函数打开的文件的大小。当使用 shm_open 配合 O_CREAT 标志创建了一个全新的共享内存时，只是在操作系统里新建了一个空白的文件，里面的内容是空的，实际空间为 0，所以需要 ftruncate 函数给这个文件描述符代表的共享内存分配空间。一个共享内存必须通过 ftruncate 函数分配空间后才能使用 mmap 函数映射到进程中使用（通常由创建共享内存的进程来分配空间，其他进程不要重复分配空间），否则会报错。

2. close 函数只是关闭共享内存的文件描述符，释放访问文件需要的资源；munmap 函数只是将共享内存在进程的地址空间中的映射撤销；这两个函数都不会从操作系统中销毁这块共享内存。调用 shm_unlink 函数会立刻删除共享内存的文件在文件系统中的目录项（如果所有进程都 close 了这个文件，这个文件也会被删除），但不会立刻释放这块内存，未取消映射的进程可以继续使用这块内存；此时再通过 shm_open 函数去打开这个文件则会打开失败（如果带了 O_CREAT 参数，会创建名字相同但是全新的另一个共享内存）。当没有任何进程再映射这个共享内存时，系统才会真正回收这块内存。

3. 共享内存中的数据结构需考虑内存对齐，避免未对齐访问导致性能下降。

4. 共享内存允许多进程并发访问，但未提供任何同步保证。例如，生产者尚未写入完成，消费者已开始读取，可能导致数据不一致（竞态条件）。一般需要搭配互斥锁或信号量来自行实现进程间的互斥访问。

5. 使用 mmap 函数映射共享内存时，函数入参中的 flags 必须包含 `MAP_SHARED` 标志。

补充说明：

1. 在 linux 系统中，若有两个进程调用 open 函数打开同一个文件，然后其中一个进程对这个文件调用了 close 函数，又对这个文件调用了 unlink 函数，而另一个进程在 open 函数之后没有执行这两个函数。那么此时，只有这个文件在文件系统中的目录项被删除了，但文件本身还没有真正被删除，磁盘空间也没有被释放。持有该文件的文件描述符（没有调用了 close 函数）的进程依然可以访问这个文件。此时再通过 open 函数去打开这个文件则会打开失败（如果带了 O_CREAT 参数，会创建名字相同但是全新的另一个文件）。

2. 使用 open 函数打开一个已存在的文件不会影响该文件的链接数，它只会使调用进程与文件之间建立一种访问关系，即 open 之后返回文件描述符。close 函数就是消除这种调用进程与文件之间的访问关系，它也不会影响文件的链接数。link 函数创建一个新目录项，并且增加一个链接数。unlink 函数会立刻删除目录项，并且减少一个链接数，但直到链接数达到 0 并且没有任何进程还在持有该文件描述符，该文件内容才被真正删除。

##### POSIX 共享内存 API 使用例程

使用共享内存实现两个独立进程在生产者-消费者模式下的数据交换，生产者进程写入数据到共享内存，消费者进程读取数据。

生产者进程代码：

``` c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>
#include <string.h>
 
#define SHM_NAME    "/my_shm"  // 共享内存名称（必须以 / 开头）
#define SHM_SIZE    (1024)
 
int main() 
{
    int shm_fd;
    char *shm_addr;

    // 如果是负责创建共享内存的进程，可以在创建前先尝试删除一次
    // 以免上一次使用时有进程异常退出导致共享内存没有释放
    shm_unlink(SHM_NAME);
 
    // 创建共享内存对象（O_CREAT：创建，O_RDWR：读写，0666：权限）
    shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (shm_fd == -1)
    {
        perror("shm_open 失败");
        exit(1);
    }
 
    // 设置共享内存大小（共享内存默认大小为 0，必须显式设置）
    // 其实我觉得 ftruncate 函数也需要同步，要防止其他进程在给共享内存分配大小之前就调用 mmap 函数
    if (ftruncate(shm_fd, SHM_SIZE) == -1)
    {
        perror("ftruncate 失败");
        exit(1);
    }
 
    // 映射到进程地址空间
    // MAP_SHARED：共享映射，这个是在映射共享内存时必须要填的
    shm_addr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
    if (shm_addr == MAP_FAILED)
    {
        perror("mmap 失败");
        exit(1);
    }

    // 关闭共享内存的文件描述符（不影响映射）
    close(shm_fd);
 
    // 向共享内存中写入数据
    const char *msg = "Hello, POSIX 共享内存！";
    strncpy(shm_addr, msg, SHM_SIZE - 1);
    printf("生产者已写入：%s\n", shm_addr);

    // 此进程不再需要访问共享内存时，解除映射
    if (munmap(shm_addr, SHM_SIZE) == -1)
    {
        perror("munmap 失败");
        exit(1);
    }

    // 等待消费者进程读取，此处省略同步机制
 
    // 删除共享内存对象
    // 一般由创建共享内存的进程或最后一个使用共享内存的进程执行
    if (shm_unlink(SHM_NAME) == -1)
    {
        perror("shm_unlink 失败");
        exit(1);
    }
 
    return 0;
}
```

消费者进程代码：

``` c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>
#include <string.h>
 
#define SHM_NAME    "/my_shm"  // 共享内存名称（必须以 / 开头）
#define SHM_SIZE    (1024)
 
int main() 
{
    int shm_fd;
    char *shm_addr;
 
    // 打开共享内存对象（O_RDWR：读写，0666：权限）
    shm_fd = shm_open(SHM_NAME, O_RDWR, 0666);
    if (shm_fd == -1)
    {
        perror("shm_open 失败");
        exit(1);
    }
 
    // 等待创建共享内存的进程调用 ftruncate 函数设置共享内存大小，此处省略同步机制
 
    // 映射到进程地址空间
    // MAP_SHARED：共享映射，这个是在映射共享内存时必须要填的
    shm_addr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
    if (shm_addr == MAP_FAILED)
    {
        perror("mmap 失败");
        exit(1);
    }

    // 关闭共享内存的文件描述符（不影响映射）
    close(shm_fd);

    // 等待生产者进程写入，此处省略同步机制

    // 读取共享内存中的数据
    char msg[SHM_SIZE];
    strncpy(msg, shm_addr, SHM_SIZE - 1);
    printf("消费者已读取：%s\n", msg);

    // 此进程不再需要访问共享内存时，解除映射
    if (munmap(shm_addr, SHM_SIZE) == -1)
    {
        perror("munmap 失败");
        exit(1);
    }
 
    // 删除共享内存对象
    // 一般由创建共享内存的进程或最后一个使用共享内存的进程执行
    if (shm_unlink(SHM_NAME) == -1)
    {
        perror("shm_unlink 失败");
        exit(1);
    }
 
    return 0;
}
```

### mmap 匿名共享内存

#### mmap 匿名共享内存相关 API 函数

| 函数 | 功能 | 参数 | 返回值 |
| --- | --- | --- | --- |
| mmap | 将共享内存映射到进程地址空间，若入参的 fd 为 -1 则是创建匿名共享内存 | void *addr, size_t length, int prot, int flags, int fd, off_t offset | 成功返回映射区的内存起始地址，失败返回 -1（MAP_FAILED）（并设置 errno）|
| munmap | 解除映射 | void *addr, size_t length | 成功返回 0，失败返回 -1（并设置 errno）|

需要注意：

1. 使用 mmap 函数创建匿名共享内存时，函数入参中的 flags 必须包含 `MAP_ANONYMOUS` 和 `MAP_SHARED` 标志，fd 必须为 `-1`。

##### mmap 匿名共享内存 API 使用例程

使用 mmap 匿名共享内存实现父子进程间的数据交换

``` c
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>
#include <sys/wait.h>
 
#define SHM_SIZE    (1024)
 
int main()
{
    char *shm_addr;
    pid_t pid;
 
    // 创建匿名共享内存（父子共享）
    shm_addr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_SHARED, -1, 0);
    if (shm_addr == MAP_FAILED)
    {
        perror("mmap 失败");
        exit(1);
    }
 
    pid = fork();   // 创建子进程
    // 父进程中 fork 函数返回新创建子进程的进程 ID
    // 子进程中 fork 函数返回 0
    if (pid == -1)
    {
        perror("fork 失败");
        exit(1);
    }
 
    if (pid == 0)
    {  
        // 子进程：写入数据
        strncpy(shm_addr, "Hello from 子进程！", SHM_SIZE - 1);
        printf("子进程写入：%s\n", shm_addr);
        munmap(shm_addr, SHM_SIZE);  // 子进程解除映射
        exit(0);
    } 
    else
    {  
        // 父进程：读取数据
        wait(NULL);  // 等待子进程退出
        // 当父进程调用 wait 函数时，它会立即阻塞自己，等待其任何一个子进程结束。
        // 如果子进程已经结束，wait函数就会回收子进程的资源并返回
        // 如果没有子进程结束，wait函数会一直阻塞直到有子进程结束
        printf("父进程读取到：%s\n", shm_addr);
        munmap(shm_addr, SHM_SIZE);  // 父进程解除映射
    }
 
    return 0;
}
```

### System V 共享内存

待续
