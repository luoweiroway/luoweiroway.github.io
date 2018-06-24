---
layout:     post
title:      "linux C标准库IO缓冲区--行缓冲实现"
subtitle:   "图解"
categories: "内核与系统编程"
date:       2015-06-01
author:     "Roway"
header-img: "http://7xizd0.com1.z0.glb.clouddn.com/blog/gnu.jpg"
tags: [linux kernel, C库]
---

glibc中对C标准库I/O缓冲区的实现，这篇讲行缓冲。
待完善。

<!-- more -->

# 1 glibc中的FILE数据结构实现

```
//<stdio.h>
typedef struct _IO_FILE __FILE;
```

```
//<libio.h>
struct _IO_FILE {
  int _flags;		/* High-order word is _IO_MAGIC; rest is flags. */
#define _IO_file_flags _flags

  /* The following pointers correspond to the C++ streambuf protocol. */
  /* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
  char* _IO_read_ptr;	/* Current read pointer */
  char* _IO_read_end;	/* End of get area. */
  char* _IO_read_base;	/* Start of putback+get area. */
  char* _IO_write_base;	/* Start of put area. */
  char* _IO_write_ptr;	/* Current put pointer. */
  char* _IO_write_end;	/* End of put area. */
  char* _IO_buf_base;	/* Start of reserve area. */
  char* _IO_buf_end;	/* End of reserve area. */
  /* The following fields are used to support backing up and undo. */
  char *_IO_save_base; /* Pointer to start of non-current get area. */
  char *_IO_backup_base;  /* Pointer to first valid character of backup area */
  char *_IO_save_end; /* Pointer to end of non-current get area. */

  struct _IO_marker *_markers;

  struct _IO_FILE *_chain;

  int _fileno;
#if 0
  int _blksize;
#else
  int _flags2;
#endif
  _IO_off_t _old_offset; /* This used to be _offset but it's too small.  */

#define __HAVE_COLUMN /* temporary */
  /* 1+column number of pbase(); 0 is unknown. */
  unsigned short _cur_column;
  signed char _vtable_offset;
  char _shortbuf[1];

  /*  char* _save_gptr;  char* _save_egptr; */

  _IO_lock_t *_lock;
#ifdef _IO_USE_OLD_IO_FILE
};
```

其中：

|指针	|意义|
| --------   | :----:  |
| \_IO\_buf\_base | 指向"缓冲区"|
| \_IO\_buf\_end   |指向"缓冲区"的末尾|
| \_IO\_read\_base |指向"读缓冲区"|
|\_IO\_read\_end | 指向"读缓冲区"的末尾|
| \_IO\_read\_ptr|读缓冲区的读指针|
| \_IO\_write\_base| 指向"写缓冲区"|
| \_IO\_write\_end |指向"写缓冲区"的末尾|
| \_IO\_write\_ptr|"写缓冲区"的写指针|

上面的定义看起来给出了3个缓冲区,实际上`_IO_read_base`、`_IO_write_base`、 `_IO_buf_base`都指向了同一个缓冲区。缓冲区在第一次buffered I/O操作时由库函数自动申请空间,最后由相应库函数负责释放

# 一个小现象引起

```
int ch1;
int ch2;
ch1 = getchar();
ch2 = getchar();
```

上面这个代码,第一次调用getchar的时候，其会调用读系统调用，以让用户输入，我们输入字符d然后换行表示输入结束，getchar调用成功后，输入缓冲区如下所示：

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-linebuffer-stdin-1.jpg)

再次调用getchar时，我们会发现我们没有输入任何东西，getchar就已反回了内容给ch2，这时因为`_IO_read_ptr`还没有与`_IO_read_end`重合，表明输入缓冲区中仍有内容可读，而该内容即为第一次输入时的结束换行符，可以发现ch2正好为10(换行符\n的ASCII码值)。
此次调用后，输入缓冲区如下所示：

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-linebuffer-stdin-2.jpg)

如果我们再加一个getchar，即代码如下：

```
int ch1;
int ch2;
ch1 = getchar();
ch2 = getchar();
ch1 = getchar();
```

则运行到第三次getchar时，会调用读系统调用，新输入的内容会覆盖原内容，假设输入字符b，然后换行结束输入，则输入缓冲区如下所示:

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-linebuffer-stdin-3.jpg)

如果输入多个字符，如hello，则他们都会被读到缓冲区，然后之后如果有调用getchar或者其他标准输入函数，则函数会直接从缓冲区中读，不再发起读系统调用。输入缓冲区如下所示：

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-linebuffer-stdin-4.jpg)

要想清除输入缓冲区的内容，某些C库(比如MSCRT)实现了fflush(stdin)，而glibc没有实现，调用此函数，stdin缓冲区没有变化。glibc下调用rewind(stdin)也没有效果。

# 3 行缓冲实现

## 3-1 stdin输入

假设调用`fgets(buf,n,stdin);`，意欲读n-1个字符，但是，输入行中的所有数据都会被写入缓冲区（不超过缓冲区大小的前提），包括表示结束输入的换行符(如果输入Ctrl-D表示文件结尾，则不会有换行符)。

如果缓冲区中的内容（即输入的字节数）大于n-1，则`_IO_read_ptr`向前移动了n位(即指向缓冲区中已被用户读走的字符的下一个),下次再次调用fgets操作时,就不需要再次调用系统调用read,而是直接从`_IO_read_ptr`开始拷贝,当`_IO_read_ptr`等于`_IO_read_end`时，标准I/O会认为已到达文件末尾或者填满的输入缓冲区已读空，再次读则需要再次调用read。

如果缓冲区中的内容（即输入的字节数）小于n-1，则有多少读多少，读完后`_IO_read_ptr`指向缓冲区中已被用户读走的字符的下一个。

正常读取完毕后(即`_IO_read_ptr`与`_IO_read_end`之间没有内容了)，所有`_IO_read_ptr`、`_IO_read_base`、`_IO_read_end`系列指针全部指向`_IO_buf_base`。

总结来说，就是：

* `_IO_read_base`始终指向缓冲区的开始；
* `_IO_read_end`始终指向已从内核读入缓冲区的字符的下一个；
* `_IO_read_ptr`始终指向缓冲区中已被用户读走的字符的下一个；
* `(_IO_read_end < _IO_buf_end) && (_IO_read_ptr == _IO_read_end)`时则表明已到达文件末尾；
* `(_IO_read_end = _IO_buf_end) && (_IO_read_ptr == _IO_read_end)`时则表明缓冲区已填满且已读完。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-linebuffer-stdin-5.jpg)


## 3-2 stdout输出

标准库函数先从用户指定的地址出读数据到输出缓冲区中，读多少则`_IO_write_ptr`往后移多少，满足以下3个条件之一会导致输出缓冲区立即被flush：

* 缓冲区已满;
* 遇到一个换行符;
* 用户主动调用fflush函数;
* 调用标准输入函数时;

输出缓冲区flush的时候，将`_IO_write_base`和`_IO_write_ptr`之间的字符通过系统调用write写入内核，然后令`_IO_write_ptr`等于`_IO_write_base`，此时标准IO认为输出缓冲区为空。

行缓冲写的时候:
* `_IO_write_base`始终指向输出缓冲区的开始；
* `_IO_write_end`始终指向输出缓冲区的开始；
* `_IO_write_ptr`始终指向输出缓冲区中已被用户写入的字符的下一个。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/glibc-linebuffer-stdout-1.jpg)

# 4 无缓冲

无缓冲时,标准I/O不对字符进行缓冲存储.典型代表是stderr,glibc中将缓冲区大小(`_IO_buf_end-_IO_buf_base`)设为1个字节。
对无缓冲的流的每次读写操作都会引起系统调用。