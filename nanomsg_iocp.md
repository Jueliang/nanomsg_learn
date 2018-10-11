# [nanomsg-1.1.4](https://nanomsg.org/) Windows 平台的 IOCP 代碼分析 

笔记写得比较乱，只是记录自己学习 nanomsg 的一些过程。

參考: [Windows I/O Completion Ports](https://docs.microsoft.com/en-us/windows/desktop/FileIO/i-o-completion-ports)

## Windows I/O completion port 功能簡介

### IOCP 相關函數

 * [CreateIoCompletionPort()][CreateIoCompletionPort]: 創建 IOCP 句柄，以及將其他設備關聯到 IOCP；
 * [ReadFile()][ReadFile]、[ReadFileEx()][ReadFileEx]: 發起異步讀
 * [WriteFile()][WriteFile]、[WriteFileEx()][WriteFileEx]: 發起異步寫
 * [WSARecv()][WSARecv]、[WSARecvFrom()][WSARecvFrom]: Socket 異步接收
 * [WSASend()][WSASend]、[WSASendTo()][WSASendTo]: Socket 異步發送
 * [PostQueuedCompletionStatus()][PostQueuedCompletionStatus]: Posts an I/O completion packet to an I/O completion port. 
 * [GetQueuedCompletionStatus()][GetQueuedCompletionStatus]: 从 IOCP 取出一个成功I/O操作的完成包(CP)
 * [GetQueuedCompletionStatusEx()][GetQueuedCompletionStatusEx]: 同上，但获取多個 CP
 * [GetOverlappedResult()][GetOverlappedResult]: 取回 overlapped 操作的結果
 * [GetOverlappedResultEx()][GetOverlappedResultEx]: 同上，但可以指定等待時間

### How I/O Completion Ports Work

1. 使用 [CreateIoCompletionPort()][CreateIoCompletionPort] 創建一個 IOCP
2. 創建綫程，調用 [GetQueuedCompletionStatus()][GetQueuedCompletionStatus] 取出已經完成的 I/O 操作包，處理之，再循環
3. 使用 [CreateIoCompletionPort()][CreateIoCompletionPort] 將一個或多個文件句柄、socket 句柄關聯到第一步創建的 IOCP
4. [ReadFile()][ReadFile]、[ReadFileEx()][ReadFileEx]: 發起異步讀
5. [WriteFile()][WriteFile]、[WriteFileEx()][WriteFileEx]: 發起異步寫
6. [WSARecv()][WSARecv]、[WSARecvFrom()][WSARecvFrom]: 異步接收
7. [WSASend()][WSASend]、[WSASendTo()][WSASendTo]: 異步發送

#### [GetQueuedCompletionStatus()][GetQueuedCompletionStatus]

Attempts to dequeue an I/O completion packet from the specified I/O completion port. If there is no completion packet queued, the function waits for a pending I/O operation associated with the completion port to complete.

To dequeue multiple I/O completion packets at once, use the [GetQueuedCompletionStatusEx][GetQueuedCompletionStatusEx] function.

```C
BOOL WINAPI GetQueuedCompletionStatus(
  _In_  HANDLE       CompletionPort,
  _Out_ LPDWORD      lpNumberOfBytes,
  _Out_ PULONG_PTR   lpCompletionKey,
  _Out_ LPOVERLAPPED *lpOverlapped,
  _In_  DWORD        dwMilliseconds
);
```

也就是說，[GetQueuedCompletionStatus()][GetQueuedCompletionStatus] 會從 IOCP 取出一個已經完成的 I/O 包。取出並處理完這個包之後，綫程應該再次調用 [GetQueuedCompletionStatus()][GetQueuedCompletionStatus] 來取出下一個完成的 I/O 包。如此循環。


#### [GetOverlappedResult()][GetOverlappedResult]

Retrieves the results of an overlapped operation on the specified file, named pipe, or communications device. To specify a timeout interval or wait on an alertable thread, use [GetOverlappedResultEx][GetOverlappedResultEx].

```C
BOOL GetOverlappedResult(
  HANDLE       hFile,
  LPOVERLAPPED lpOverlapped,
  LPDWORD      lpNumberOfBytesTransferred,
  BOOL         bWait
);
```

這裏説 [GetOverlappedResult()][GetOverlappedResult] 用於文件、命名管道、通信設備， 不知道 socket 算不算 communications device?

#### [ReadFileEx()][ReadFileEx]、[WSARecv()][WSARecv]、[WSARecvFrom()][WSARecvFrom]

這組函數本質上類似，以 [ReadFileEx()][ReadFileEx] 為例：

Reads data from the specified file or input/output (I/O) device. It reports its completion status asynchronously, calling the specified completion routine when reading is completed or canceled and the calling thread is in an alertable wait state.


```C
BOOL ReadFileEx(
  HANDLE                          hFile,
  LPVOID                          lpBuffer,
  DWORD                           nNumberOfBytesToRead,
  LPOVERLAPPED                    lpOverlapped,
  LPOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine
);
```

#### [WriteFileEx()][WriteFileEx]、[WSASend()][WSASend]、[WSASendTo()][WSASendTo]

以 [WriteFileEx()][WriteFileEx] 為例:

Writes data to the specified file or input/output (I/O) device. It reports its completion status asynchronously, calling the specified completion routine when writing is completed or canceled and the calling thread is in an alertable wait state.

```C
BOOL WriteFileEx(
  HANDLE                          hFile,
  LPCVOID                         lpBuffer,
  DWORD                           nNumberOfBytesToWrite,
  LPOVERLAPPED                    lpOverlapped,
  LPOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine
);
```


#### `[PostQueuedCompletionStatus()][PostQueuedCompletionStatus]` 的功能

Posts an I/O completion packet to an I/O completion port.

```C
BOOL WINAPI PostQueuedCompletionStatus(
  _In_     HANDLE       CompletionPort,
  _In_     DWORD        dwNumberOfBytesTransferred,
  _In_     ULONG_PTR    dwCompletionKey,
  _In_opt_ LPOVERLAPPED lpOverlapped
);
```


## nanomsg 與 IOCP 相關函數

### 查找 `CreateIoCompletionPort()` 函數:

```
find . -name "*.[ch]" -o -name "*.inc" | xargs -r -n1 grep -Hn "CreateIoCompletionPort"
```

>>>

```
src/aio/worker_win.inc:85:    self->cp = CreateIoCompletionPort (INVALID_HANDLE_VALUE, NULL, 0, 0);
src/aio/usock_win.inc:539:    cp = CreateIoCompletionPort (...）
```

在 worker_win.inc:85 内，是 `nn_worker_init()` 函數調用了 `CreateIoCompletionPort()`：

```C
int nn_worker_init (struct nn_worker *self)
{
    self->cp = CreateIoCompletionPort (INVALID_HANDLE_VALUE, NULL, 0, 0);
    win_assert (self->cp);
    nn_timerset_init (&self->timerset);
    nn_thread_init (&self->thread, nn_worker_routine, self);

    return 0;
}
```    

在 usock_win.inc:539 内，則是 `nn_usock_create_io_completion()` 函數:
 
```C 
static void nn_usock_create_io_completion (struct nn_usock *self)
{
    struct nn_worker *worker;
    HANDLE cp;

    /*  Associate the socket with a worker thread/completion port. */
    worker = nn_fsm_choose_worker (&self->fsm);
    cp = CreateIoCompletionPort (
        self->p,
        nn_worker_getcp(worker),
        (ULONG_PTR) NULL,
        0);
    nn_assert(cp);
}
```

查看 `CreateIoCompletionPort()` 的定義：

```C
HANDLE WINAPI CreateIoCompletionPort(
  _In_     HANDLE    FileHandle,
  _In_opt_ HANDLE    ExistingCompletionPort,
  _In_     ULONG_PTR CompletionKey,
  _In_     DWORD     NumberOfConcurrentThreads
);  
```

由此可見，worker_win.inc 内的 `nn_worker_init()` 函數創建了 IOCP，並將 IOCP 句柄保持到 nn_worker::cp，而 usock_win.inc 的 
`nn_usock_create_io_completion()` 則將 socket 句柄 nn_usock::p 與 IOCP 相關聯。

### 現在來看看都有誰用了 nn_worker::cp

```
find . -name "*.[ch]" -o -name "*.inc" | xargs -r -n1 grep -Hn "\bcp\b"
```
>>>

```
src/aio/usock_win.inc:539:    cp = CreateIoCompletionPort (
src/aio/worker_win.inc:85:    self->cp = CreateIoCompletionPort (INVALID_HANDLE_VALUE, NULL, 0, 0);
src/aio/worker_win.inc:98:    brc = PostQueuedCompletionStatus (self->cp, 0,
src/aio/worker_win.inc:106:   brc = CloseHandle (self->cp);
src/aio/worker_win.inc:114:   brc = PostQueuedCompletionStatus (self->cp, 0, (ULONG_PTR) task, NULL);
src/aio/worker_win.inc:132:   return self->cp;
src/aio/worker_win.inc:169:   brc = GetQueuedCompletionStatusEx (self->cp, entries,
```

查看源代碼，下列代碼與 nn_worker::cp 有関：

 * src/aio/usock_win.inc:539 之 `nn_usock_create_io_completion()`, 將 nn_usock::p 關聯到 IOCP；
 * src/aio/worker_win.inc:85 之 `nn_worker_init()`, 創建 IOCP 並保存到 nn_worker::cp；
 * src/aio/worker_win.inc:98 之 `nn_worker_term()`，通知 IOCP 要關閉了；
 * src/aio/worker_win.inc:106 同上，之 `nn_worker_term()`，這是要關閉 IOCP 了；
 * src/aio/worker_win.inc:114 之 `nn_worker_execute()`, 調用 `PostQueuedCompletionStatus()`；
 * src/aio/worker_win.inc:132: 之 `nn_worker_getcp()`，只做一件事：返回 nn_worker::cp；
 * src/aio/worker_win.inc:169: 之 `nn_worker_routine()`, 調用 `GetQueuedCompletionStatusEx()`。


### 再來看看都有哪裏調用了 `nn_worker_getcp()`：

```
find . -name "*.[ch]" -o -name "*.inc" | xargs -r -n1 grep -Hn "\bnn_worker_getcp\b"
```

>>>

```
src/aio/usock_win.inc(541):  nn_worker_getcp(worker),
src/aio/worker_win.h(71):    HANDLE nn_worker_getcp (struct nn_worker *self);
src/aio/worker_win.inc(130): HANDLE nn_worker_getcp (struct nn_worker *self)
```

可見，只有 `nn_usock_create_io_completion()` 調用了 `nn_worker_getcp()` 用來將 nn_usock::p 關聯到 nn_worker::cp。


### 由以上分析可知，使用了 IOCP 句柄的有以下函數：

 * `nn_worker_init()`，               創建 IOCP 並保存到 nn_worker::cp。 位於 src/aio/worker_win.inc；
 * `nn_usock_create_io_completion()`, 將 nn_usock::p 關聯到 IOCP。 位於 src/aio/usock_win.inc:539；
 * `nn_worker_execute()`，            調用 `PostQueuedCompletionStatus()`。 位於 src/aio/worker_win.inc；
 * `nn_worker_routine()`,             調用 `GetQueuedCompletionStatusEx()`。位於 src/aio/worker_win.inc:169: 
 * `nn_worker_term()`,                關閉 IOCP。 位於 src/aio/worker_win.inc:98 之 `nn_worker_term()`，通知 IOCP 要關閉了。

從 IOCP 系列函數查看：

 * `CreateIoCompletionPort()`     ： 創建和關聯 IOCP 各用了一次；
 * `PostQueuedCompletionStatus()` ： nn_worker_execute() 一次，nn_worker_term() 一次；
 * `GetQueuedCompletionStatusEx()`： nn_worker_routine 一次；
 * `GetOverlappedResult()`        ： 未使用；
 * `GetOverlappedResultEx()`      ： 未使用。



## nanomsg IOCP 工作过程

### 创建 IOCP 

src/aio/worker_win.inc 内的 nn_worker_init() 函数创建了 IOCP:

```C
int nn_worker_init (struct nn_worker *self)
{
    self->cp = CreateIoCompletionPort (INVALID_HANDLE_VALUE, NULL, 0, 0);
    win_assert (self->cp);
    nn_timerset_init (&self->timerset);
    nn_thread_init (&self->thread, nn_worker_routine, self);

    return 0;
}
```

代码显示 nn_worker_init() 同时也创建了定时器集 timerset 以及 I/O 线程，线程函数是 nn_worker_routine()。

posix 版的 nn_worker_init() 同样创建了定时器集 timerset 以及 I/O 线程，线程函数名称也是 nn_worker_routine()，不过 
posix 版 nn_worker_routine() 在 src\aio\worker_posix.inc 内实现，与 windows 版是不同的。

关于 posix 版的实现，在此且按过不表。

再来看看何处调用了 nn_worker_init():

```
find . -name "*.[ch]" -o -name "*.inc" | xargs -r -n1 grep -Hn "nn_worker_init"
```

发现只有一处：

```
src/aio/pool.c:30:    return nn_worker_init (&self->worker);
```

源代码如下：

```C
int nn_pool_init (struct nn_pool *self)
{
    return nn_worker_init (&self->worker);
}
```

而调用 nn_pool_init() 的也只有一处：

```
src/core/global.c:265:    nn_pool_init (&self.pool);
```
源代码如下：

```C
static void nn_global_init (void){
    ...
    
    /*  Start the worker threads. */
    nn_pool_init (&self.pool);
}
```  

唯有 src/core/global.c 调用了 nn_global_init()
```C
int nn_socket (int domain, int protocol){
    ...
    nn_global_init ();
    ...
}
```    

那么每一个实际的应用，都需要调用 nn_socket()，而且可能多次调用。这样的话，nn_global_init() 也会被多次调用。
那么 nn_pool_init() 会不会被多次调用呢？进一步说，IOCP 会不会被创建多个？

让我们来看一下 nn_global_init() 的代码: 

```C
static void nn_global_init (void)
{
    ...
    
    /*  Check whether the library was already initialised. If so, do nothing. */
    if (self.socks)
        return;

    /*  Allocate the global table of SP sockets. */
    self.socks = nn_alloc ((sizeof (struct nn_sock*) * NN_MAX_SOCKETS) +
        (sizeof (uint16_t) * NN_MAX_SOCKETS), "socket table");
    alloc_assert (self.socks);
    
    ...

    /*  Start the worker threads. */
    nn_pool_init (&self.pool);
}
```
由源代码可知，nn_pool_init() 只会被调用一次，也即 IOCP 在整个进程内只会被创建一次。

这里的 self，是 src/core/global.c 里面的一个静态对象： static struct nn_global self; 不过为什么要叫做 self 呢？

小结一下 IOCP 创建的过程：

> App => nn_socket() => nn_global_init() => nn_pool_init() => nn_worker_init() => CreateIoCompletionPort()

### 创建线程

Windows 平台下，nanomsg 创建线程是通过 src/utils/thread_win.inc() 里面的 nn_thread_init() 函数：

```C
void nn_thread_init (struct nn_thread *self,
    nn_thread_routine *routine, void *arg)
{
    self->routine = routine;
    self->arg = arg;
    self->handle = (HANDLE) _beginthreadex (NULL, 0,
        nn_thread_main_routine, (void*) self, 0 , NULL);
    win_assert (self->handle != NULL);
}
```

nanomsg 未使用 CreateThread() 来创建线程。

nn_thread_init() 被 src/aio/worker_win.inc 内的 nn_worker_init）所调用：

```C
int nn_worker_init (struct nn_worker *self)
{
    self->cp = CreateIoCompletionPort (INVALID_HANDLE_VALUE, NULL, 0, 0);
    win_assert (self->cp);
    nn_timerset_init (&self->timerset);
    nn_thread_init (&self->thread, nn_worker_routine, self);

    return 0;
}
```

另外一个调用了 nn_thread_init() 的地方，是 src/devices/device.c 的 nn_device_twoway() 函数，用于两个 socket 之间转发。按过不表。

查询源码可知，nn_worker_init() 只被调用过一次，就是 src/aio/pool.c 内的 nn_pool_init() 函数：

```C
int nn_pool_init (struct nn_pool *self)
{
    return nn_worker_init (&self->worker);
}
```

前面分析过，nn_pool_init() 只被调用过一次。

小结一下线程创建过程:

> App => nn_socket() => nn_global_init() => nn_pool_init() => nn_worker_init() => nn_thread_init() => _beginthreadex()

### 工作线程

工作线程 nn_worker_routine() 有 nn_worker_init() 创建。工作线程实现的代码在 src/aio/worker_win.inc() 内：

```C
static void nn_worker_routine (void *arg){
    ...
    OVERLAPPED_ENTRY entries [NN_WORKER_MAX_EVENTS];
    struct nn_worker_op *op;
    ...
    while (1) {
        ...
        brc = GetQueuedCompletionStatusEx (self->cp, entries,
            NN_WORKER_MAX_EVENTS, &count, timeout < 0 ? INFINITE : timeout,
            FALSE);
        ...
        for (i = 0; i != count; ++i) {

            /*  Process I/O completion events. */
            if (nn_fast (entries [i].lpOverlapped != NULL)) {
                DWORD nxfer;
                op = nn_cont (entries [i].lpOverlapped,
                    struct nn_worker_op, olpd);
                rc = entries [i].Internal & 0xc0000000;
                switch (rc) {
                case 0x00000000:
                     nxfer = entries[i].dwNumberOfBytesTransferred;
                     if (op->start != NULL) {
                         op->resid -= nxfer;
                         op->buf += nxfer;

                         /*  If we still have more to transfer, keep going. */
                         if (op->resid != 0) {
                             op->start (op->arg);
                             continue;
                         }
                     }
                     rc = NN_WORKER_OP_DONE;
                     break;
                ...
                default:
                     continue;
                }

                /*  Raise the completion event. */
                nn_ctx_enter (op->owner->ctx);
                nn_assert (op->state != NN_WORKER_OP_STATE_IDLE);

                op->state = NN_WORKER_OP_STATE_IDLE;

                nn_fsm_feed (op->owner, op->src, rc, op);
                nn_ctx_leave (op->owner->ctx);

                continue;
            }
            ...
            /*  Process tasks. */
            task = (struct nn_worker_task*) entries [i].lpCompletionKey;
            nn_ctx_enter (task->owner->ctx);
            nn_fsm_feed (task->owner, task->src,
                NN_WORKER_TASK_EXECUTE, task);
            nn_ctx_leave (task->owner->ctx);
        }
    }
}
```

为简洁起见，上面源码省略掉了错误处理。

从该代码可知，IOCP 调用 GetQueuedCompletionStatusEx() 来取出完成的 I/O packet，然后调用对应的 nn_worker_op 的 start 函数来处理该 packet。

### CP packet 处理函数

通过前面的分析可知，GetQueuedCompletionStatusEx() 取出已经完成的 I/O packet 后，调用 nn_worker_op 的 start 函数来处理该 packet。

nn_worker_op 定义在 src/aio/worker_win.h 内：

```C
struct nn_worker_op {
    ...
    OVERLAPPED olpd;

    size_t resid;
    char *buf;
    void *arg;
    void (*start)(struct nn_usock *);
    ...
};
```

那么 nn_worker_op::start 什么时候设置呢？就在 src/aio/usock_win.inc 的 nn_usock_recv() 内：

```C
void nn_usock_recv (struct nn_usock *self, void *buf, size_t len, int *fd)
{
    ...
    if (self->domain == AF_UNIX) {
        self->in.start = nn_usock_recv_start_pipe;
    }
    else {
        self->in.start = nn_usock_recv_start_wsock;
    }

    self->in.start (self->in.arg);
}
```



[CreateIoCompletionPort]: https://docs.microsoft.com/zh-cn/windows/desktop/FileIO/createiocompletionport
[PostQueuedCompletionStatus]: https://msdn.microsoft.com/en-us/library/aa365458.aspx
[GetQueuedCompletionStatus]: https://msdn.microsoft.com/en-us/library/Aa364986.aspx
[GetQueuedCompletionStatusEx]: https://msdn.microsoft.com/en-us/library/aa364988.aspx
[GetOverlappedResult]: https://msdn.microsoft.com/7f999959-9b22-4491-ae2b-a2674d821110
[GetOverlappedResultEx]: https://msdn.microsoft.com/2f77f7fe-bdde-4c52-8571-fe0ab533aa7f
[ReadFile]: https://docs.microsoft.com/en-us/windows/desktop/api/FileAPI/nf-fileapi-readfile
[ReadFileEx]: https://msdn.microsoft.com/6c1a4de1-6cae-4c35-bfba-0bc252fadbd9
[WriteFile]: https://docs.microsoft.com/en-us/windows/desktop/api/FileAPI/nf-fileapi-writefile
[WriteFileEx]: https://msdn.microsoft.com/6995c4ee-ba91-41d5-b72d-19dc2eb95945
[WSASend]: https://msdn.microsoft.com/library/windows/desktop/ms742203
[WSASendTo]: https://msdn.microsoft.com/library/windows/desktop/ms741693
[WSARecv]: https://msdn.microsoft.com/library/windows/desktop/ms741688
[WSARecvFrom]: https://msdn.microsoft.com/library/windows/desktop/ms741686
