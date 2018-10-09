# [nanomsg-1.1.4](https://nanomsg.org/) IOCP 代碼分析 

### 參考

 * [Windows I/O Completion Ports](https://docs.microsoft.com/en-us/windows/desktop/FileIO/i-o-completion-ports)


## Windows I/O completion port

### `PostQueuedCompletionStatus()` 的功能

MS 說，`PostQueuedCompletionStatus()` 的功能是： Posts an I/O completion packet to an I/O completion port.

先來看定義：

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
