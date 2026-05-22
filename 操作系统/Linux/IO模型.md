# IO 模型

## 文件描述符

文件描述符（File Descriptor）是一个非负整数，是进程打开文件记录表的索引。每个进程的 `files` 指针数组前三位默认指向：0（stdin）、1（stdout）、2（stderr）。Linux 中一切皆文件，设备、管道、socket 等都通过文件描述符操作。

## 缓存 I/O

缓存 I/O 又称标准 I/O，数据先拷贝到内核缓冲区（page cache），再从内核拷贝到应用程序地址空间。缺点：数据在用户空间和内核空间之间多次拷贝，CPU 和内存开销大。

## 一个 IO 操作的两个阶段

1. **等待数据准备好**（数据从网络到达内核缓冲区）
2. **从内核向进程复制数据**（内核缓冲区 → 应用进程缓冲区）

## 五种 IO 模型

### 1. 阻塞式 IO（Blocking IO）

进程调用系统调用后一直阻塞，直到数据准备好并从内核复制到用户空间后才返回。程序简单，但每个连接需独立线程。

### 2. 非阻塞式 IO（Non-blocking IO）

数据未准备好时立即返回错误码，进程不断轮询检查。不会阻塞，但轮询占用大量 CPU。

### 3. IO 多路复用（I/O Multiplexing）

内核代替用户进程轮询多个文件描述符，当有数据就绪时通知进程。常见实现：select、poll、epoll。

至少两次系统调用，单连接性能不及阻塞式，但可同时监听大量连接。

### 4. 信号驱动式 IO（Signal Driven IO）

进程先注册 SIGIO 信号处理函数，数据准备好时内核发信号通知，进程再执行 IO 操作。UDP 场景适用，TCP 场景近乎无用。

### 5. 异步 IO（Asynchronous IO）

真正的非阻塞。进程发起 IO 调用后立即返回，内核完成数据准备和复制后通过回调通知进程。IO 两个阶段进程都不阻塞。

| 模型 | 阻塞阶段 | 是否轮询 | 特点 |
|:---|:---:|:---:|:---|
| 阻塞式 IO | 两个阶段 | 否 | 简单，每连接需独立线程 |
| 非阻塞式 IO | 复制阶段 | 是 | 轮询消耗 CPU |
| IO 多路复用 | 两个阶段 | 内核代劳 | 单线程管理大量连接 |
| 信号驱动式 IO | 复制阶段 | 否 | UDP 友好，TCP 不适用 |
| 异步 IO | 无 | 否 | 真正非阻塞，实现复杂 |

## select、poll、epoll

| 特性 | select | poll | epoll |
|:---|:---|:---|:---|
| 最大连接数 | 1024（有限） | 无上限 | 无上限 |
| 效率 | 遍历 fd，O(n) | 遍历 fd，O(n) | 回调通知，O(1) |
| 工作模式 | LT | LT | LT / ET |
| 跨平台 | 好 | 较好 | Linux 特有 |
| 内存拷贝 | 每次调用需从用户态拷贝到内核态 | 同左 | 只需拷贝一次 |

### epoll 工作模式

| 模式 | 说明 |
|:---|:---|
| **LT（Level Triggered，水平触发）** | 默认模式。若数据未读完，下次 epoll_wait 会再次通知，不易漏事件但可能重复触发 |
| **ET（Edge Triggered，边缘触发）** | 高速模式。状态变化时通知一次，须一次读完所有数据，否则可能漏事件。需配合非阻塞 socket |

### epoll 优势

- 监视描述符数不受限制（上限为最大可打开文件数）
- 效率不随 fd 数量增长而下降（基于回调机制，而非轮询）
- 大量 idle-connection 场景效率远高于 select/poll

### epoll 操作过程

```c
int epoll_create(int size);                                      // 创建 epoll 句柄
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); // 注册/修改/删除 fd
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout); // 等待事件
```

**epoll_event.events** 可选值：`EPOLLIN`（可读）、`EPOLLOUT`（可写）、`EPOLLERR`（错误）、`EPOLLHUP`（挂断）、`EPOLLET`（边缘触发）、`EPOLLONESHOT`（只监听一次）。
