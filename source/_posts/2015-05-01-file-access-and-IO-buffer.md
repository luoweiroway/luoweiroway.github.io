---
layout:     post
title:      "linux C标准库函数使用"
subtitle:   "文件访问和IO缓冲区相关"
categories: "内核与系统编程"
date:       2015-05-01
author:     "Roway"
header-img: "http://7xizd0.com1.z0.glb.clouddn.com/blog/universe.jpg"
tags: [linux kernel, C库]
---

# 相关头文件

<!-- more -->

>stdio.h
>>FILE * fopen ( const char * filename, const char * mode );
>>int fclose ( FILE * stream );
>>FILE * freopen ( const char * filename, const char * mode, FILE * stream );
>>void setbuf ( FILE * stream, char * buffer );
>>int setvbuf ( FILE * stream, char * buffer, int mode, size_t size );
>>int fflush ( FILE * stream );



# 1 两类IO
* Unbuffered I/O，是操作系统内核部分，也是系统调用。因为这里的不带缓冲指用户空间没有缓冲，即在用户的进程中对这两个函数不会缓冲，内核空间是有缓存的；

* Buffered I/O，严格来讲应该叫User buffered IO ， C库里面的这些IO也叫Standard IO。这里的缓冲是在用户层再建立一个缓冲区，这个缓冲区的分配和优化长度等细节都是标准IO库代你处理，不依赖系统内核，所以移植性强，一般是为了效率考虑对这些系统调用的封装。目的是为了减少system calls次数以及block-align I/O operations。比如在写的时候，可以先把数据只写在用户空间缓冲区，当设定阈值(ideally, the underlying filesystem's block size or an integer multiple thereof)达到时，再调用write系统调用。

* Unix I/O函数都是针对文件描述符，对于C标准I/O库来说，打开的文件由流(FILE *指针)标识。当打开一个流时，使一个流与文件相结合，标准I/O函数fopen返回一个指向FILE对象的指针。该对象通常是一个结构，它包含了I/O库为管理该流所需要的所有信息：用于实际I/O的文件描述符，指向流缓冲的指针，缓冲区的长度，当前在缓冲区中的字符数，出错标志等等。

## 1.1 磁盘文件IO过程
当应用程序尝试读取某块数据的时候，如果这块数据已经存放在页缓存(内核高速缓冲)中，那么这块数据就可以立即返回给应用程序，而不需要经过实际的物理读盘操作。当然，如果数据在应用程序读取之前并未被存放在页缓存中，那么就需要先将数据从磁盘读到页缓存中去。

对于写操作来说，应用程序也会将数据先写到页缓存中去（如果是调用标准库I/O进行写，那么首先是写到标准库的缓冲区内，如果标准库的缓冲区写满以后，再写到页缓冲内；如果是系统调用，那么直接写到页缓冲内）。

处于页缓存中的数据是否被立即写到磁盘上去取决于应用程序所采用的写操作机制：如果用户采用的是同步写机制,那么数据会立即被写回到磁盘上，应用程序会一直等到数据被写完为止；如果用户采用的是延迟写机制，那么应用程序就完全不需要等到数据全部被写回到磁盘，数据只要被写到页缓存中去就可以了。在延迟写机制的情况下，操作系统会定期地将放在页缓存中的数据刷到磁盘上。与异步写机制不同的是，延迟写机制在数据完全写到磁盘上得时候不会通知应用程序，而异步写机制在数据完全写到磁盘上得时候是会返回给应用程序的。所以延迟写机制本身是存在数据丢失的风险的，而异步写机制则不会有这方面的担心。

* 无缓冲IO操作数据流向路径：数据——内核缓存区——磁盘
* 标准IO操作数据流向路径：数据——流缓冲区——内核缓存区——磁盘
* 访问大量小数据时，标准IO会占优，而访问数据量较大的文件时，无缓存IO会占优。

# 2 标准库IO缓冲区
用户程序调用C标准I/O库函数读写文件或设备，而这些库函数要通过系统调用把读写请求传给内核，最终由内核驱动磁盘或设备完成I/O操作。C标准库为每个打开的文件分配一个I/O缓冲区以加速读写操作，通过文件的FILE结构体可以找到这个缓冲区，这样可以**让用户调用读写函数大多数时候都在I/O缓冲区中读写**，只有少数时候需要把读写请求传给内核。所以，标准I/O库提供缓冲的目的是减少read和write调用次数，节省执行I/O所需的CPU时间。

以fgetc/fputc为例，当用户程序第一次调用fgetc读一个字节时，fgetc函数可能通过系统调用进入内核读1K字节(如果文件有这么大)到I/O缓冲区中，然后返回I/O缓冲区中的第一个字节给用户，把读写位置指向I/O缓冲区中的第二个字符，以后用户再调fgetc，就直接从I/O缓冲区中读取，而不需要进内核了，当用户把这1K字节都读完之后，再次调用fgetc时，fgetc函数会再次进入内核读1K字节到I/O缓冲区中。

C标准库之所以会从内核预读一些数据放在I/O缓冲区中，是希望用户程序随后要用到这些数据，C标准库的I/O缓冲区也在用户空间，直接从用户空间读取数据比进内核读数据要快得多，C标准库和内核之间的关系就像在“Memory Hierarchy”中CPU、Cache和内存之间的关系一样。

另一方面，用户程序调用fputc通常只是写到I/O缓冲区中，这样fputc函数可以很快地返回，如果I/O缓冲区写满了，fputc就通过系统调用把I/O缓冲区中的数据传给内核，内核最终把数据写到磁盘。I/O缓冲区中的数据除了由标准I/O库函数自动冲洗外，也可手动立即强制冲洗，对应的库函数是fflush。

fclose函数在关闭文件之前也会做flush操作，它会冲洗输出缓冲的输出数据，丢弃输入缓冲的输入数据。当一个进程正常终止时，比如调用exit(), 或从main返回(main 函数被启动代码这样调用`exit(main(argc, argv));`)，exit()会执行标准I/O库的清理关闭操作：为所有打开流调用fclose(),然后通过_exit系统调用进入内核退出当前进程。

## 2.1 缓冲区类型
C标准库的I/O缓冲区有三种类型：全缓冲、行缓冲和无缓冲。当用户程序调用库函数做写操作时，不同类型的缓冲区具有不同的特性。
 
* 全缓冲：如果缓冲区满了就写回内核(调用write系统调用)。常规文件(磁盘上的文件)通常是全缓冲的。
* 行缓冲：如果用户程序写的数据中有换行符或者缓冲区写满了就写回内核。标准输入和标准输出对应终端设备时通常是行缓冲的。
* 无缓冲：用户程序每次调库函数做写操作都要通过系统调用写回内核。标准错误输出通常是无缓冲的，这样用户程序产生的错误信息可以尽快输出到设备。

对任何一个给定的流，如果我们并不喜欢这些系统默认的情况，则可调用下列两个函数中的一个更改缓冲类型：

```c
void setbuf( FILE *restrict fp, char *restrict buf );
int setvbuf( FILE *restrict fp, char *restrict buf, int mode, size_t size );
//返回值：若成功则返回0，若出错则返回非0值
```

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/setvbuf-setbuf.jpg)

上表中，合适长度指的是由stat结构中的成员st_blksize所指定，如果系统不能为该流决定此值（例如若此流涉及一个设备或一个管道），则分配长度为BUFSIZ(一般为1024)的缓存。

# fflush
* **原型**：
int fflush ( FILE * stream );
* **功能**：
 * 如果*stream*被打开来**写**(或者**update**打开，并且最近一次IO是输出操作)，那此函数将输出缓冲区的任何未写入数据写入文件（严格来讲，是写入内存缓存区高速缓存）。
 * 如果*stream*指向输入流（如stdin），那么fflush函数的行为是不确定的。故而使用 fflush(stdin)是不正确的，在有些库实现中，冲刷正在读的stream会导致输入缓冲区内容被丢弃。由于标准未规定，所以这不是一个可移植行为。在其它所有标准未规定的情况下，该调用行为取决于相应库的实现。见示例2。
 * 如果stream参数是null，那么会对所有打开文件的IO缓冲区做flush操作；
 * 该函数调用之后stream仍然保持打开；
 * 当一个流关闭后，比如调用fclose，或者程序正常终止(因为exit()会执行标准I/O库的清理关闭操作，即为所有打开流调用fclose())了，文件的所有缓冲都会被自动冲刷，而且还会调用fsync确保将其写到磁盘上。

* **参数**：
*stream*：Pointer to a FILE object that identifies the stream.

* **返回值**：
成功则返回零，如果发生错误，则返回EOF， error indicator也会被设置。
* **使用**：
 * 无论当前采用何种缓冲区模式，在任何时候都可以使用fflush库函数强制将stream输出缓冲区的数据刷新到`内核缓冲区`，注意与fsync区分，详细分析见后文[I/O缓冲区探讨](http://todo)。
 * 在包括glibc库在内的许多C函数库实现中，**若stdin和stdout指向同一终端，那么无论何时从stdin中读取输入时，都将隐含调用一次fflush(stdout)函数**,这将刷新写入stdout的任何提示。当然SUSv3和C99并未规定这个行为，所以有的C库没有实现这点，要保证程序的可移植性最好显式调用fflush(stdout)。
 * 若打开一个流同时用于输入和输出，则C99标准提出了两个要求。首先，一个输出操作后不能紧跟一个输入操作，必须在二者之间调用fflush函数或是一个文件定位函数(fseek,fsetpos,rewind等)。其次，一个输入操作后不能紧跟一个输出操作，必须在二者之间调用一个文件定位函数，除非输入操作遭遇文件结尾。
* **示例 1**：

当文件以update (既可读又可写)方式打开,output操作之后input之前，stream必须被冲刷。可以**调用repositioning相关的调用(fseek, fsetpos, rewind等)**或者调用精确的fflush：

```c
/****************************************************************
***Author: luowei
***Description: none
****************************************************************/
#include <stdio.h>
char mybuffer[80];
int main()
{
   FILE * fp; 
   fp = fopen ("test.txt","r+");
   if (fp == NULL) perror ("Error opening file");
   else {
     fputs ("test",fp);
     //fflush (fp);    // flushing or repositioning required
     rewind(fp);
     fgets (mybuffer,80,fp);
     puts (mybuffer);
     fclose (fp);
     return 0;
  }
}
```

* **示例 2**：

```c
/****************************************************************
***Author: luowei
***Description: none
****************************************************************/
#include <stdio.h>
int main()
{
    int num,j,scanf_ret;
    for (j=0;j<10;j++) {
	    fputs("Input an integer: ", stdout);
	    scanf_ret = scanf("%d", &num);
	    printf("num=%d,scanf_ret=%d\n", num,scanf_ret);
    }
    return 0;
}
```

当输入非数字时，比如字母时，那scanf会读取失败，同时输入缓冲区不变，也就是刚输入的字母还在缓冲区中，
运行结果：
可以用FILE* fp指针，并赋值为stdin，这样可以在调试中观察输入缓冲区的内容。第一次输入之前，stdin缓冲区如下：

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/stdio-stdin-1.jpg)

第一次输入，输入数字1scanf调用成功，读取数字1：

```
Input an integer: 1
num=1,scanf_ret=1
```

此时stdin缓冲区如下，可以发现`fp->_base`指向的缓冲区大小为0x1000，目前有三个字符，分别为`1\n\n`（前两个字节为用户输入的字符，即1和换行符\n，再后面一个\n自己猜测应该是标准库额外加入的，如果输入123，则缓冲区有5个字符，分别为`123\n\n`），输入函数运行完毕后，`fp->_ptr`在`fp->_base`后面一字节，表明读取了一个字节给用户，用户输入的换行符和系统加的换行符还在后面。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/stdio-stdin-2.jpg)

第二次输入时，输入12345678，scanf调用成功，读取数字12345678：

```
Input an integer: 12345678
num=12345678,scanf_ret=1
```

此时`fp->_base`指向的缓冲区由`1\n\n`被覆盖为`12345678\n\n`，输入函数运行完毕后，`fp->_ptr`在`fp->_base`后面8字节，表明读取了8个字节给用户，然后转换为数字12345678，用户输入的换行符和系统加的换行符还在后面。

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/stdio-stdin-3.jpg)

第三次输入，输入b，scanf调用失败，返回值为0，num值不变：

```
Input an integer: b
num=12345678,scanf_ret=0
```

此时，`fp->_base`指向的缓冲区由`12345678\n\n`覆盖为`b\n\n45678\n\n`，只覆盖了前面的123，因为只输入了b和换行符，加上系统加的换行符，所以前面的123被覆盖为b\n\n，但是`fp->_ptr`仍然是`fp->_base`，并没有往后推移，表明调用失败，因为b不是一个数字字符。可以看到此时的`fp->_cnt`（其指代用户输入的字符数（包括换行符）被成功读取后剩下的数量）不为1，表明用户输入的b和\n被传送至缓冲区后都没有被成功读取，前面成功读取的都为1（剩下用户输入的换行符）.

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/stdio-stdin-4.jpg)

第三次运行scanf函数时，因为`fp->_ptr`指向的并不是\n，换句话说scanf函数的用户态代码还以为内核已经把数据传过来了，于是就直接读取，然后发现b不是数字字符，于是又返回调用失败，num值仍然不变。

```
Input an integer: num=12345678,scanf_ret=0
```

此时，stdin缓冲区不变：

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/stdio-stdin-5.jpg)

再之后，都是如此。

```
Input an integer: num=12345678,scanf_ret=0
Input an integer: num=12345678,scanf_ret=0
Input an integer: num=12345678,scanf_ret=0
Input an integer: num=12345678,scanf_ret=0
Input an integer: num=12345678,scanf_ret=0
Input an integer: num=12345678,scanf_ret=0
```

所以说，当scanf调用失败后，程序员需要去清空输入缓冲区，有的C库实现了fflush(stdin),比如我们在代码中加入：

```c
/****************************************************************
***Author: luowei
***Description: none
****************************************************************/
#include <stdio.h>
int main()
{
    int num,j,scanf_ret;
    for (j=0;j<10;j++) {
	    fputs("Input an integer: ", stdout);
	    scanf_ret = scanf("%d", &num);
	    if(scanf_ret==0){
	        fflush(stdin);
        }
	    printf("num=%d,scanf_ret=%d\n", num,scanf_ret);
    }
    return 0;
}
```

假设之前因为输入b而导致scanf调用失败，此时调用fflush清空输入缓冲区后，效果如下：

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/stdio-stdin-6.jpg)

可以发现其将`fp->_cnt`变为0，下次再调用scanf函数，虽然`fp->_ptr`仍然是`fp->_base`指向，但是scanf已经可以正常让用户输入而不是直接读取缓冲区。

可以发现，scanf函数在运行时：

* 先判断需不需要调用读系统调用：
 * 如果`fp->_cnt`等于0，或者`fp->_cnt`不等于0但是其指向的内容是换行符\n，则调用读系统调用，用户需要输入新的输入，输入的内容（包括输入的换行符）加上库再添加的换行符会覆盖`fp->_base`指向的区域，然后`fp->_ptr`也会先定位到`fp->_base`，然后从`fp->_ptr`指向的位置开始读取（库缓冲区到用户态的读取）。
 * 如果`fp->_cnt`不等于0且其指向的内容不是换行符\n，则表明缓冲区仍有内容，scanf这时就不去调用读系统调用，也就是不需要用户输入新的输入，而是直接用已有的输入去进行读取处理，从`fp->_ptr`指向的位置开始读取（库缓冲区到用户态的读取）。
* 读成功则`fp->_ptr`往后移动，读失败则`fp->_ptr`不动，当内容为非数字时，比如字母之类的字符，scanf会返回失败。

当然，由于fflush(stdin)不是所有的C库都支持，所以我们可以自己写代码来完成清空缓冲区的内容，而且是真正的清空，不像fflush(stdin)那样，只是将`fp->_cnt`置零，`fp->_ptr`不变。代码如下：

```c
/****************************************************************
***Author: luowei
***Description: none
****************************************************************/
#include <stdio.h>
int main()
{
    int num,j,scanf_ret;
    for (j=0;j<10;j++) {
	    fputs("Input an integer: ", stdout);
	    scanf_ret = scanf("%d", &num);
	    if(scanf_ret==0){
	        //fflush(stdin);
	        int c;
	        while ( (c=getchar()) != '\n' && c != EOF );
        }
	    printf("num=%d,scanf_ret=%d\n", num,scanf_ret);
    }
    return 0;
}
```

如果因为输入错误字符，比如b，则运行完12-16行之间的代码后，效果如下图：

![](http://7xizd0.com1.z0.glb.clouddn.com/blog/stdio-stdin-7.jpg)

可以看到，由于缓冲区仍有内容，getchar不会调用读系统调用，循环调用getchar读取后，`fp->_ptr`指向由`b\n\n`变为`\n`，也就是说之前输入的b和换行符已被用getchar函数读走，此时`fp->_cnt`为0，这时再运行scanf就可以让用户正常输入了。当然，如果此时再调用getchar，那么getchar就会调用读系统调用了，用于需要输入内容以供getchar读取。这里直接使用gets函数效果与上面完全一样。

**总结**：
getchar、gets、scanf这些输入函数，都会先判断要不要调用读系统调用：

* 如果`fp->_cnt`等于0，则所有读IO函数都需要调用读系统调用。
* 如果`fp->_cnt`不等于0且`fp->_ptr`指向的第一个字节是换行符：getchar和gets直接会读取缓冲区，getchar会将换行符返回，而gets会将读取的换行符丢弃；scanf会调用读系统调用让用户输入。
* 如果`fp->_cnt`不等于0且`fp->_ptr`指向的第一个字节不是换行符，那三者都不会调用读系统调用，而是直接从输入缓冲区中读内容以作相应处理。



# fclose
* **原型**：int fclose ( FILE * stream );

* **功能**：
 * 关闭*stream*指定的文件；
 * 所有该stream相关的内部缓冲都会与其解除映射关系并被刷洗：未写入的输出缓冲区内容会被写入，未读的输入缓冲区内容会被丢弃(the content of any unwritten output buffer is written and the content of any unread input buffer is discarded.)；
 * 即使该调用失败，参数指定的stream也不再与其文件及缓冲相映射。

* **参数**：
*stream*：Pointer to a FILE object that identifies the stream.

* **返回值**：
如果stream成功关闭则返回零，如果发生错误，则返回EOF。

>updated:20150726