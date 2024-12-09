---
title: linux pagecache
date: 2020-08-08 21:24:46
tags: linux pagecache buffercache
categories:
  - [linux]
---

## 一、磁盘文件的存储

1. 磁盘的最小存储单元是扇区sector，一个扇区是512个字节
2. 存储在磁盘的文件系统采用块作为最小的存储单元，1个块大小一般是4KB(1KB, 2KB, 4KB, 8KB)
3. 磁盘管理器负责处理块到扇区的映射，给定设备号和块号，磁盘管理器可以很方便找到对应的扇区

<!--more-->    

## 二、page cache
缓存**文件**内容，以page为单位。

## 三、buffer cache
缓存**硬盘**内容，以块为单位。
磁盘的最小数据单位为sector（扇区），内核会在磁盘sector上构建一层缓存，他以sector的整数倍力度单位(block)，缓存部分sector数据在内存中


## 四、逻辑关系

从linux-2.6.18的内核源码来看，Page Cache和Buffer Cache是一个事物的两种表现：对于一个Page而言，对上，他是某个File的一个Page Cache，而对下，他同样是一个Device上的一组Buffer Cache。
实际IO操作，都是与page cache交互，不与内存直接交互

*The term, Buffer Cache, is often used for the Page Cache. Linux kernels up to version 2.2 had both a Page Cache as well as a Buffer Cache. As of the 2.4 kernel, these two caches have been combined. Today, there is only one cache, the Page Cache.*

***假设page cache=4K，buffer cache=1k，则一个page cache中有4个buffer cache。（page cache和buffer cache合并后）***

[注意]：这里的Page Cache与Buffer Cache的融合，是针对文件这一层面的Page Cache与Buffer Cache的融合。对于跨层的：File层面的Page Cache和裸设备Buffer Cache，虽然都统一到了基于Page的实现，但File的Page Cache和该文件对应的Block在裸设备层访问的Buffer Cache，这两个是完全独立的Page，这种情况下，一个物理磁盘Block上的数据，仍然对应了Linux内核中的两份Page，一个是通过文件层访问的File的Page Cache(Page Cache)，一个是通过裸设备层访问的Page Cache(Buffer Cache)。

![逻辑关系](2020-04-10-linux-pagecache/27_file_page_device_block.png)


## 五、write->系统调用->内核是怎么处理的？

写的时候，内核应该是从用户态进程空间将数据copy至内核态的page cache，此时内核会返回程序写入成功结果，但是数据并没有实际写入硬盘;
程序可以调用flush方法将数据写入硬盘，或者是等内核自己写入磁盘（例如page cache空间不足时，或者是定时写磁盘）;
页表，只是一个虚拟地址跟物理地址的一个映射存储容器；
page cache，是为了解决内存跟硬盘速度问题，是一个缓存容器，以页为单位对文件内容进行缓存;
在内存中的数据，是怎样的写入硬盘呢？是需要通过CPU呢还是可以使用DMA让内存跟硬盘直接交互

当然，目前 BufferCache 仍然是存在的，因为还存在需要执行的块 IO。因为大多数块都是用来存储文件数据，所以大部分 BufferCache 都指向了 PageCache；但还是有一小部分块并不是文件数据，例如元数据、RawBlock IO，此时还需要通过 BufferCache 来缓存。



## 六、mmp

用户调用mmap将文件映射到内存时，内核进行一系列的参数检查，然后创建对应的vma，然后给该vma绑定vma_ops。当用户访问到mmap对应的内存时，CPU 会触发page fault，在page fault回调中，将申请pagecache中的匿名页，读取文件到其物理内存中，然后将pagecache中所属的物理页与用户进程的vma进行映射。

***简单点说：就是在用户进程中创建变量vma和物理内存进行映射，而不需要再将数据从page cache再拷贝至用户进程空间。***


## 七、read系统调用
调用open函数时，可以指定是以阻塞方式还是以非阻塞方式打开一个文件描述符。

    阻塞方式打开：
    int fd = open("/dev/tty", O_RDWR|O_NONBLOCK);
    非阻塞方式打开：
    int fd = open("/dev/tty", O_RDWR);


**对于网络IO之socket**，默认是阻塞的，即当去read的时候，用户进程会阻塞，由内核去获取数据然后拷贝至用户空间；
设置成非阻塞时，read时候，用户进程立马得到消息（是否有内容），此时用户进程不是阻塞的，那么如果没有获取数据的话，按一般做法肯定是轮训的询问内核数据是否准备好，会加大内核压力；因此，多路复用就出现了....


**对于普通文件IO（IO包下）**，我觉得默认是阻塞的，读一次，就获取多少数据，直到wile循环结束将文件读完...
Java中IO包下的都是bio，代表是open系统调用是采用默认的阻塞方式，此时进程会阻塞，直到内核将数据从磁盘读到page cache，再从page cache读到用户空间

[Linux Cache VS. Buffer](https://gohalo.me/post/linux-memory-buffer-vs-cache-details.html)
[Linux内核Page Cache和Buffer Cache关系及演化历史](http://lday.me/2019/09/09/0023_linux_page_cache_and_buffer_cache/)
[文件IO系统调用内幕](https://lrita.github.io/2019/03/13/the-internal-of-file-syscall/)