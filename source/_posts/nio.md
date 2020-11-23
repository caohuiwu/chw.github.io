---
title: nio
date: 2019-06-14 23:00:42
tags: nio
categories: io
---
从上篇bio文章，我们了解了传统socket连接为阻塞IO，然后了解了bio的多线程优化方式及其缺点。多路复用则是另外一种优化方式。

一、nio的设计思想

<!--more-->  

![nio-1](nio/nio-common.png)


以下是代码设计思想
```
    //服务端
    public class BioServer {
    
        public static void main(String[] args) {
            List<Socket> socketList = Lists.newArrayList();
            
            byte[] bs = new byte[1024];
            try {
                //服务端的监听socket，只负责监听连接，监听的端口是：9878
                ServerSocket serverSocket = new ServerSocket();
                serverSocket.bind(new InetSocketAddress(9878));
                serverSocket.setBlock(false);//伪代码，表示设置serverSocket为非阻塞
    
                while (true) {//可以进行下一次的通信
                    Socket accept = serverSocket.accept();
                    if (accept == null) {
                        //表示该次while中，无新连接
                        socketList.forEach(socket -> {
                            //遍历socket，看是否有数据发过来
                            int readCount = socket.getInputStream().read(bs);
                            if(readCount > 0) {
                                //输出
                            }
                        });
                    } else {
                        accept.setBlock(false);//伪代码，表示socket.read为非阻塞
                        socketList.add(accept);                            
                        socketList.forEach(socket -> {
                            int readCount = socket.getInputStream().read(bs);
                            if(readCount > 0) {
                                //输出
                            }
                        });
                    
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    //服务端
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.socket().bind(new InetSocketAddress(port));
        Selector selector = Selector.open();
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        while(true){
            int n = selector.select();
            if (n == 0) continue;
            Iterator ite = this.selector.selectedKeys().iterator();
            while(ite.hasNext()){
                SelectionKey key = (SelectionKey)ite.next();
                if (key.isAcceptable()){
                    SocketChannel clntChan = ((ServerSocketChannel) key.channel()).accept();
                    clntChan.configureBlocking(false);
                    //将选择器注册到连接到的客户端信道，
                    //并指定该信道key值的属性为OP_READ，
                    //同时为该信道指定关联的附件
                    clntChan.register(key.selector(), SelectionKey.OP_READ, ByteBuffer.allocate(bufSize));
                }
                if (key.isReadable()){
                    handleRead(key);
                }
                if (key.isWritable() && key.isValid()){
                    handleWrite(key);
                }
                if (key.isConnectable()){
                    System.out.println("isConnectable = true");
                }
              ite.remove();
            }
        }
```      
More info:[参考文章](https://www.bilibili.com/video/av54147951/)

### 二、多路复用具体实现
    
    多路复用是nio模型的一个升级，将由应用程序循环遍历的socket列表交给了内核，由内核去通知应用程序socket是否OK。
      
    
1. 首先先了解Linux的select,poll,epoll三种多路复用模式。     
More info:[参考文章](https://jeff.wtf/2017/02/IO-multiplexing/)        


2. 再了解Java中的多路复用的实现
More info:[参考文章](https://juejin.im/entry/599f971af265da247d728531)

        
### 三、多路复用模式

    Reactor和proactor
    
More info:[参考文章](https://tech.meituan.com/2016/11/04/nio.html)            