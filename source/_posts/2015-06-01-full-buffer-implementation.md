---
layout:     post
title:      "linux C标准库IO缓冲区--全缓冲实现"
subtitle:   "图解"
categories: "内核与系统编程"
date:       2015-06-01
author:     "Roway"
header-img: "http://7xizd0.com1.z0.glb.clouddn.com/blog/gnu.jpg"
tags: [linux kernel, C库]
---

glibc中对C标准库I/O缓冲区的实现，这篇讲全缓冲。
待完善。

<!-- more -->

全缓冲，在读空或写满标准I/O缓冲区后才进行实际的read或write系统调用操作。

# 1 读
全缓冲读的时候:

* `_IO_read_base`始终指向缓冲区的开始；
* `_IO_read_end`始终指向已从内核读入缓冲区的字符的下一个(对全缓冲来说,buffered I/O读每次都试图都将缓冲区读满,除非文件大小不足缓冲区大小)；
* `_IO_read_ptr`始终指向缓冲区中已被用户读走的字符的下一个；
* `(_IO_read_end < _IO_buf_end) && (_IO_read_ptr == _IO_read_end)`时则表明已到达文件末尾；
* `(_IO_read_end = _IO_buf_end) && (_IO_read_ptr == _IO_read_end)`时则表明缓冲区已填满且已读完。
   
![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-fullbuffer-1.jpg)

# 2 写
全缓冲写的时候:

* `_IO_write_base`始终指向缓冲区的开始；
* `_IO_write_end`始终指向缓冲区的最后一个字符的下一个；
* `_IO_write_ptr`始终指向缓冲区中已被用户写入的字符的下一个；
* 当`_IO_write_ptr`与`_IO_write_end`重合时，表明写缓冲区满。

当写缓冲区满时或用户主动调用fflush函数，将对输出缓冲区进行flush操作，flush的时候，将`_IO_write_base`和`_IO_write_ptr`之间的字符通过系统调用write写入内核.

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-fullbuffer-2.jpg)




