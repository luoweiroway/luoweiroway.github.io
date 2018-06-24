---
layout:     post
title:      "真正理解Unix IO模型，同步/异步与阻塞/非阻塞"
subtitle:   "unix下网络IO"
categories: "内核与系统编程"
date:       2015-05-21
author:     "Roway"
header-img: "http://7xizd0.com1.z0.glb.clouddn.com/blog/universe.jpg"
tags: [network, IO]
---

本文致力于理清以下几个点：

* 不同知识背景的人对这四个概念的具体理解是不一样的，在不同语境下也会有不一样；
* 当我们谈同步异步的时候，我们更侧重于说什么？当我们谈阻塞非阻塞的时候，我们更侧重于说什么？
* unix环境下网络IO模型的理清；

<!-- more -->

# 概念

首先解决前两个问题，我们从纯概念层面来看：

* 同步异步描述**两个对象**之间的关系，表示**一种协作关系**，不同场景下有细分区别；
* 而阻塞非阻塞描述**一个对象**的状态，就是该对象（如进程）阻在这儿了，不能继续往下执行了。**阻塞可以当做拿来实现同步的一种手段**。


## 同步异步的概念是要分场景的

我们最常见的就是指**编程里面的调用，同步调用与异步调用**：

* 当发出一个同步调用后，在没有得到结果之前，该调用就不返回，而一般来说返回就得到结果。也就是必须一件一件事做，等前一件做完了才能做下一件事。
* 当发出一个异步调用后，会立即返回，但调用者不能立刻得到结果。实际处理这个调用的部件在完成后，通过**状态、通知或回调函数来通知调用者**。

一般来说，**同步会用阻塞来实现**（因为你既然什么事都不做，那还让你占用CPU干嘛），但同步调用也未必一定会导致阻塞(退出CPU执行队列)，例如调用者可以去死循环询问。而异步调用一般会立即返回，只是返回的不是结果，一般来说**异步不会导致阻塞**。


再提几个不同背景下的同步异步概念：

同步线程与异步线程：

* 同步线程：即两个线程步调要一致，要相互协商。
* 异步线程：步调不用一致，各自按各自的步调运行，不受另一个线程的影响。
* 同步是指两个线程的运行是相关的，其中一个线程可能要阻塞等待另外一个线程的运行；异步的意思是两个线程毫无相关，自己运行自己的。


通信领域下的概念，同步通信与异步通信：

* 这里的同步和异步是指：发送方和接收方是否协调步调一致！
* 同步通信是指：发送方和接收方通过一定机制，实现收发步调协调。如：发送方发出数据后，等接收方发回响应以后才发下一个数据包的通讯方式
* 异步通信是指：发送方的发送不管接收方的接收状态，如：发送方发出数据后，不等接收方发回响应，接着发送下个数据包的通讯方式。

同步不等于阻塞，异步也不等于非阻塞

# 实际编程场景下的IO

下面谈谈实际编程场景下的IO，内容出自史诗级巨著Richard Stevens的《UNIX网络编程·卷一》

对于一个network IO，它会涉及到两个系统对象，一个是调用这个IO的用户态进程(或线程)，另一个就是系统内核。当一个read发生时，它会经历两个阶段：

* 等待设备数据的准备；
* 系统从内核态复制数据到用户进程。

这个步骤的区分很重要，因为不同IO模型的区别就在这里。

unix环境下网络IO的五种IO Model：

* blocking IO
* non-blocking IO
* IO multiplexing
* signal driven IO
* asynchronous IO

## blocking IO

linux中，默认情况下所有的socket都是blocking。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/blocking-IO.jpg)

当用户进程调用了系统调用读，kernel就开始了IO的第一个阶段：准备数据。对于network io来说，很多时候数据在一开始还没有到达（比如，还没有收到一个完整的UDP包），这个时候就要等待足够的数据到来。而在用户进程这边，整个进程会被阻塞。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。
所以，blocking IO的特点就是在IO执行的两个阶段都被block了。

## non-blocking IO

在调用读系统调用时，可以通过设置参数使其变为non-blocking。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/non-blocking-IO.jpg)

从图中可以看出，当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的系统调用，那么它马上就将数据拷贝到了用户内存，注意在拷贝数据时用户进程是阻塞的。

这里，用户进程其实是需要不断的主动询问kernel数据好了没有。

## IO multiplexing

就是select，poll，epoll等系统调用，有些地方也称这种IO方式为event driven IO。我们都知道，select/epoll的好处就在于单个process就可以同时处理多个网络连接的IO。它的基本原理就是select/epoll这个系统调用中会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/IO-multiplexing.jpg)

当用户进程调用了select，那么整个进程会被block，而同时，kernel会“监视”所有select中注册的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read系统调用，将数据从kernel拷贝到用户进程。

这个图和blocking IO的图其实并没有太大的不同，但注意这里需要使用两个系统调用，而blocking IO只调用了一个系统调用。用select的优势在于它可以同时处理多个连接。（所以，如果处理的连接数不是很高的话，使用select/epoll的web server不一定比使用multi-threading + blocking IO的web server性能更好，可能延迟还更大。select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。）

在IO multiplexing Model中，实际中，对于每一个socket，一般都设置成为non-blocking，但是，如上图所示，整个用户的process其实是一直被block的。只不过process是被select这个函数block，而不是被socket IO给block。

## signal driven IO

linux下的signal driven IO用的很少。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/signal-driven-IO.jpg)

首先要安装一个信号处理程序。此系统调用立即返回，用户进程继续工作，它是非阻塞的。当数据报准备好时，就生成一个SIGIO信号来通知用户进程。我们就可以立即在信号处理程序中调用recvfrom来读数据报。这种模型的好处是等待数据时可以不阻塞。

## asynchronous IO

inux下的asynchronous IO其实用得也很少。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/asynchronous-IO.jpg)

用户进程发起read操作之后，内核会立即返回状态，所以不会对用户进程产生任何block，用户进程可以继续执行。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal(在aio_read调用中注册)，告诉它read操作完成了。

## 总结

POSIX对同步IO和异步IO的定义：

* A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;
* An asynchronous I/O operation does not cause the requesting process to be blocked;

可知，同步I/O操作引起请求进程阻塞，直到I/O操作完成；异步I/O操作不引起请求进程阻塞。根据这个定义**前四个模型都属于同步I/O模型**，因为他们要么在第一阶段就已经阻塞要么在第二阶段开始阻塞，只有异步I/O模型才与这个异步I/O的定义相符。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/unix-network-io.jpg)

这里，non-blocking IO和asynchronous IO的区别还是很明显的。在non-blocking IO中，虽然进程大部分时间都不会被block，但是它仍然要求进程去主动的check，并且当数据准备完成以后，也需要进程主动的再次调用recvfrom来将数据拷贝到用户内存。而asynchronous IO则完全不同，它就像是用户进程将整个IO操作交给了他人（kernel）完成，然后他人做完后发信号通知。在此期间，用户进程不需要去检查IO操作的状态，也不需要主动的去拷贝数据。

看得出来，这里的同步异步的定义是不能推广开来的，只限于unix下的IO分类，比如wiki上就认为asynchronous IO和non-blocking IO是一个东西。




# 参考
http://www.cnblogs.com/albert1017/p/3914149.html
http://blog.chinaunix.net/uid-26000296-id-3754118.html
http://blog.chinaunix.net/uid-21411227-id-1826898.html
http://blog.csdn.net/historyasamirror/article/details/5778378
http://blog.csdn.net/hguisu/article/details/7453390

updated 20150821





