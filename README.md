# [nanomsg][nanomsg] 學習與使用筆記

 * [nanomsg 官網: https://nanomsg.org/][nanomsg]
 * [nanomsg github: https://github.com/nanomsg/nanomsg](https://github.com/nanomsg/nanomsg)

参考：

 * 云风： [ZeroMQ 的模式](https://blog.codingnow.com/2011/02/zeromq_message_patterns.html)

### 筆記

 * [nanomsg-1.1.4 IOCP 代碼分析](https://github.com/Jueliang/nanomsg_learn/blob/master/nanomsg_iocp.md)
 * [ZMQ 指南](https://github.com/anjuke/zguide-cn)
 * [ZeroMQ及其模式](https://zhuanlan.zhihu.com/p/22947038)
 * CSDN: [zeromq源代码分析1------基本工作流程分析](https://blog.csdn.net/kaka11/article/details/6614479)
 * CSDN: [zeromq源代码分析2------线/进程间通信方式](https://blog.csdn.net/kaka11/article/details/6617046)
 * CSDN: [zeromq源代码分析3------zeromq中的消息 ](https://blog.csdn.net/kaka11/article/details/6617473)
 * CSDN: [zeromq源代码分析4------encoder，decoder，multipart_message](https://blog.csdn.net/kaka11/article/details/6620150)
 * CSDN: [zeromq源代码分析5-1------管道相关的数据结构yqueue, ypipe, pipe等](https://blog.csdn.net/kaka11/article/details/6622081)
 * CSDN: [zeromq源代码分析5-2------管道相关的数据结构yqueue, ypipe, pipe等](https://blog.csdn.net/kaka11/article/details/6622768)
 * CSDN: [zeromq源代码分析5-3------管道相关的数据结构yqueue, ypipe, pipe等](https://blog.csdn.net/kaka11/article/details/6623769)
 * CSDN: [zeromq源代码分析6-1------zeromq各种类型的socket之socket_base](https://blog.csdn.net/kaka11/article/details/6626131)
 * CSDN: [zeromq源代码分析6-2------REQ和REP](https://blog.csdn.net/kaka11/article/details/6626318)
 * CSDN: [zeromq源代码分析6-3------ROUTER和DEALER](https://blog.csdn.net/kaka11/article/details/6630303)


### [nng][nng]

[nanomsg 官網][nanomsg]說：

 > A new project, nng, is underway as reimplementation of these same protocols. nng is wire compatible with nanomsg, and offers a number of additional advanced capabilities. Although nng itself is still in pre-release state, we are encouraging people using or considering using nanomsg to look at nng. 

[nng][nng]說：

 > This project is a rewrite of the Scalability Protocols library known as libnanomsg, and adds significant new capabilities, while retaining compatibility with the original.


[nng][nng] 描述 [nanomsg][nanomsg] 為 legacy：

> If you are looking for the legacy version of nanomsg, please see the nanomsg repository. 


好吧，學習完 [nanomsg][nanomsg]，倘若行有餘力，就再寫一篇《[nng][nng] 學習與使用筆記》;p


[nanomsg]:https://nanomsg.org/
[nng]:https://github.com/nanomsg/nng
