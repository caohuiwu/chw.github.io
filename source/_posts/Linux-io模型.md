---
title: Linux-io模型
date: 2019-04-11 15:43:10
categories: Linux
tags: Linux IO
---
### 前言
先了解一下同步、异步、阻塞、非阻塞的相关概念。

- 同步：一般指的是程序的顺序执行
```
    public class TestClass{
        public static void main(String[] args) {
            A();
        }
        public static void A() {
            System.out.println("A执行");
            B();
            System.out.println("A执行完成");
        }
        
        public static void B() {
            System.out.println("B执行");
            while(true) {
                ....
            }
            System.out.println("B执行完成");
        }
    }
```
<!--more-->  

从该例子中，只有当B()执行完成后，A()方法才会输出"A执行完成"，这就是同步。
    整个处理过程顺序执行，当各个过程都执行完毕，并返回结果。是一种线性执行的方式，执行的流程不能跨越
    
- 异步：ajax就是最好的例子
```
    function ajaxTest() {
        console.log("方法开始执行");
        $.ajax({
            url: "/A/remoteCall",
            type: "post",
            success: function (data) {
                while(true) {
                    console.log("方法开始执行");
                }
            }
        });
        console.log("方法执行结束");
    }
```


方法执行，不需要等待ajax的返回结果。

    
- 阻塞：一般指的是线程的状态
```
    public Result export) {
            try {
                InputStream inputStream = new ByteArrayInputStream(ExcelUtil.getInstance().export(excelDataList, map, groupIdNameMap).toByteArray());
                FileCopyUtils.copy(inputStream, outputStream);
            } catch (Exception e) {
                log.error("导出Excel出错了", e);
                result.setMessage("操作失败!");
                result.setStatus(REST_STATUS.FAILD_EXCEPTION);
            }
            return result;
        }
```


>例如导出中，操作系统读取文件返回花较长时间，程序不会往下执行，进入阻塞状态。    
    
More info: [参考文章](https://www.jianshu.com/p/aed6067eeac9)    
    
### Linux IO模型

1. 阻塞IO（bloking IO）
2. 非阻塞IO（non-blocking IO）
3. 多路复用IO（multiplexing IO）
4. 信号驱动式IO（signal-driven IO）
5. 异步IO（asynchronous IO）                


#### 一. 同步阻塞IO
![Linux-io模型](Linux-io-model/sync-block.png)
- 两个步骤：

   1. 步骤一：用户空间的应用程序执行一个系统调用（recvform），linux kernel开始IO的第一阶段：准备数据。这会导致应用程序阻塞（进程自己选择的阻塞），什么也不干，直到数据准备好。
   2. 步骤二：当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。数据从内核复制到用户进程，最后进程再处理数据，在等待数据到处理数据的两个阶段，整个进程都被阻塞。

#### 二. 同步非阻塞IO
![Linux-io模型](Linux-io-model/sync-nonblock.png)
>同步非阻塞就是通过轮训的方式（轮训执行系统调用），去判断数据是否准备好（轮训者：程序进程）
需要注意，拷贝数据整个过程，进程仍然是属于阻塞的状态。


#### 三. IO多路复用
![Linux-io模型](Linux-io-model/async-block.png)


>强调一点就是，IO多路复用模型并没有涉及到非阻塞，进程在发出select后，要一直阻塞等待其监听的所有IO操作至少有一个数据准备好才返回，强调阻塞状态，不存在非阻塞。
而在 Java NIO中也可以实现多路复用，主要是利用多路复用器 Selector，与这里的 select函数类型，Selector会不断轮询注册在其上的通道Channel，如果有某一个Channel上面发生读或写事件，这个Channel处于就绪状态，就会被Selector轮询出来。关于Java NIO实现多路复用更多的介绍请查询相关文章。


More info:[参考文章](https://www.jianshu.com/p/486b0965c296)