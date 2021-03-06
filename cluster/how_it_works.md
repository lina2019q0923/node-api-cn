
<!--type=misc-->

工作进程由 [`child_process.fork()`] 方法创建，因此它们可以使用 IPC 和父进程通信，从而使各进程交替处理连接服务。

cluster 模块支持两种分发连接的方法。

第一种方法（也是除 Windows 外所有平台的默认方法）是循环法，由主进程负责监听端口，接收新连接后再将连接循环分发给工作进程，在分发中使用了一些内置技巧防止工作进程任务过载。

第二种方法是，主进程创建监听 socket 后发送给感兴趣的工作进程，由工作进程负责直接接收连接。

理论上第二种方法应该是效率最佳的。
但在实际情况下，由于操作系统调度机制的难以捉摸，会使分发变得不稳定。
可能会出现八个进程中有两个分担了 70% 的负载。

因为 `server.listen()` 将大部分工作交给主进程完成，因此导致普通 Node.js 进程与 cluster 工作进程差异的情况有三种：

1. `server.listen({fd: 7})` 因为消息会被传给主进程，所以父进程中的文件描述符 7 将会被监听并将句柄传给工作进程，而不是监听文件描述符 7 指向的工作进程。
2. `server.listen(handle)` 显式地监听句柄，会导致工作进程直接使用该句柄，而不是和主进程通信。
3. `server.listen(0)` 正常情况下，这种调用会导致 server 在随机端口上监听。
  但在 cluster 模式中，所有工作进程每次调用 `listen(0)` 时会收到相同的“随机”端口。
  实质上，这种端口只在第一次分配时随机，之后就变得可预料。
  如果要使用独立端口的话，应该根据工作进程的 ID 来生成端口号。

Node.js 不支持路由逻辑。
因此在设计应用时，不应该过分依赖内存数据对象，例如 session 和登陆等。

由于各工作进程是独立的进程，它们可以根据需要随时关闭或重新生成，而不影响其他进程的正常运行。
只要有存活的工作进程，服务器就可以继续处理连接。
如果没有存活的工作进程，现有连接会丢失，新的连接也会被拒绝。
Node.js 不会自动管理工作进程的数量，而应该由具体的应用根据实际需要来管理进程池。

虽然 `cluster` 模块主要用于网络相关的情况，但同样可以用于其他需要工作进程的情况。

