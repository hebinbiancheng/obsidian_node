---
title: 07 Go Channel
source: "[[04 知识资料/编程语言/Go/source/Go 原始 XMind 解析.md]]"
created: 2026-06-12
tags:
  - Go
  - 编程语言
  - 面试
status: 已拆分
---

# 07 Go Channel

> 来源：[[04 知识资料/编程语言/Go/source/Go 原始 XMind 解析.md]]
  - 7. channel相关
    - 0、介绍一下 Channel
      - Go 语言中，不要通过共享内存来通信，而要通过通信来实现内存共享。Go 的CSP(Communicating Sequential Process)并发模型，中文可以叫做通信顺序进程，是通过 goroutine 和 channel 来实现的。
      - channel 收发遵循先进先出 FIFO 的原则。分为有缓冲区和无缓冲区，channel中包括 buffer、sendx 和 recvx 收发的位置(ring buffer 记录实现)、sendq、recv。当 channel 因为缓冲区不足而阻塞了队列，则使用双向链表存储。
    - 1、channel 是否线程安全？锁用在什么地方？
      - Golang的Channel,发送一个数据到Channel 和 从Channel接收一个数据 都是 原子性的。
      - 而且Go的设计思想就是:不要通过共享内存来通信，而是通过通信来共享内存，前者就是传统的加锁，后者就是Channel。
      - 也就是说，设计Channel的主要目的就是在多任务间传递数据的，这当然是安全的
    - 2、go channel 的底层实现原理 （数据结构）
      - 数据结构
        - （空标题）
        - （空标题）
      - 总结hchan结构体的主要组成部分有四个：
        - 用来保存goroutine之间传递数据的循环链表。=====> buf。
        - 用来记录此循环链表当前发送或接收数据的下标值。=====> sendx和recvx。
        - 用于保存向该chan发送和从改chan接收数据的goroutine的队列。=====> sendq 和 recvq
        - 保证channel写入和读取数据时线程安全的锁。=====> lock
      - https://juejin.cn/post/7037656471210819614
    - 3、nil、关闭的 channel、有数据的 channel，再进行读、写、关闭会怎么样？（各类变种题型，重要）
      - Channel读写特性(15字口诀)
        - 空读写阻塞，写关闭异常，读关闭空零
      - Channel特性
        - 给一个 nil channel 发送数据，造成永远阻塞
        - 从一个 nil channel 接收数据，造成永远阻塞
        - 给一个已经关闭的 channel 发送数据，引起 panic
        - 从一个已经关闭的 channel 接收数据，如果缓冲区中为空，则返回一个零值
        - 无缓冲的channel是同步的，而有缓冲的channel是非同步的
    - 4、向 channel 发送数据和从 channel 读数据的流程是什么样的？
      - 发送流程：
        - 向一个channel中写数据简单过程如下：
          - 如果等待接收队列recvq不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从recvq取出G,并把数据写入，最后把该G唤醒，结束发送过程；
          - 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程；
          - 如果缓冲区中没有空余位置，将待发送数据写入G，将当前G加入sendq，进入睡眠，等待被读goroutine唤醒；
        - 简单流程图如下：
          - （空标题）
      - 接收流程：
        - 从一个channel读数据简单过程如下：
          - 如果等待发送队列sendq不为空，且没有缓冲区，直接从sendq中取出G，把G中数据读出，最后把G唤醒，结束读取过程；
          - 如果等待发送队列sendq不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程；
          - 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；
          - 将当前goroutine加入recvq，进入睡眠，等待被写goroutine唤醒；
        - 简单流程图如下：
          - （空标题）
      - 关闭channel
        - 关闭channel时会把recvq中的G全部唤醒，本该写入G的数据位置为nil。把sendq中的G全部唤醒，但这些G会panic。
        - 除此之外，panic出现的常见场景还有：
          - 关闭值为nil的channel
          - 关闭已经被关闭的channel
          - 向已经关闭的channel写数据
    - 5、讲讲 Go 的 chan 底层数据结构和主要使用场景
      - channel 的数据结构包含 qccount 当前队列中剩余元素个数，dataqsiz 环形队列长度，即可以存放的元素个数，buf 环形队列指针，elemsize 每个元素的大小，closed 标识关闭状态，elemtype 元素类型，sendx 队列下表，指示元素写入时存放到队列中的位置，recv 队列下表，指示元素从队列的该位置读出。recvq 等待读消息的 goroutine 队列，sendq 等待写消息的 goroutine 队列，lock 互斥锁，chan 不允许并发读写。
      - 无缓冲和有缓冲区别： 管道没有缓冲区，从管道读数据会阻塞，直到有协程向管道中写入数据。同样，向管道写入数据也会阻塞，直到有协程从管道读取数据。管道有缓冲区但缓冲区没有数据，从管道读取数据也会阻塞，直到协程写入数据，如果管道满了，写数据也会阻塞，直到协程从缓冲区读取数据。
      - channel 的一些特点： 1）、读写值 nil 管道会永久阻塞 2）、关闭的管道读数据仍然可以读数据 3）、往关闭的管道写数据会 panic 4）、关闭为 nil 的管道 panic 5）、关闭已经关闭的管道 panic
      - 向 channel 写数据的流程： 如果等待接收队列 recvq 不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从 recvq 取出 G,并把数据写入，最后把该 G 唤醒，结束发送过程； 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程； 如果缓冲区中没有空余位置，将待发送数据写入 G，将当前 G 加入 sendq，进入睡眠，等待被读 goroutine 唤醒；
      - 向 channel 读数据的流程： 如果等待发送队列 sendq 不为空，且没有缓冲区，直接从 sendq 中取出 G，把 G 中数据读出，最后把 G 唤醒，结束读取过程； 如果等待发送队列 sendq 不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把 G 中数据写入缓冲区尾部，把 G 唤醒，结束读取过程； 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；将当前 goroutine 加入 recvq，进入睡眠，等待被写 goroutine 唤醒；
      - 使用场景： 消息传递、消息过滤，信号广播，事件订阅与广播，请求、响应转发，任务分发，结果汇总，并发控制，限流，同步与异步
    - 6、有缓存channel和无缓存channel
      - 无缓部通道中，读取时如果没有数据写入，协程会阻塞至有数据写入。
      - 有缓冲的通道是没有阻塞等待的。
      - 无缓存channel适用于数据要求同步的场景，而有缓存channel适用于无数据同步的场景。可以根据实现项目需求选择。
    - 7、Go 语言当中 Channel（通道）有什么特点，需要注意什么？
      - 如果给一个 nil 的 channel 发送数据，会造成永远阻塞。 如果从一个 nil 的 channel 中接收数据，也会造成永久阻塞。 给一个已经关闭的 channel 发送数据， 会引起 panic 从一个已经关闭的 channel 接收数据， 如果缓冲区中为空，则返回一个零值。
    - 8、Go 语言当中 Channel 缓冲有什么特点？
      - 无缓冲的 channel 是同步的，而有缓冲的 channel 是非同步的。
    - 9、Channel 的 ring buffer 实现
      - channel 中使用了 ring buffer（环形缓冲区) 来缓存写入的数据。ring buffer 有很多好处，而且非常适合用来实现 FIFO 式的固定长度队列。
      - 在 channel 中，ring buffer 的实现如下：
        - （空标题）
      - 上图展示的是一个缓冲区为 8 的 channel buffer，recvx 指向最早被读取的数据，sendx 指向再次写入时插入的位置。

