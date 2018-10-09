# [nanomsg-1.1.4](https://nanomsg.org/) IOCP 代碼分析 

參考: [Windows I/O Completion Ports](https://docs.microsoft.com/en-us/windows/desktop/FileIO/i-o-completion-ports)

## Windows I/O completion port 功能簡介

### IOCP 相關函數

 * [CreateIoCompletionPort()][CreateIoCompletionPort]: 創建 IOCP 句柄，以及將其他設備關聯到 IOCP；
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


## 查找 nanomsg 與 IOCP 相關函數

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
