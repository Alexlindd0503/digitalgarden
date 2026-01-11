---
{"dg-publish":true,"permalink":"/Computer/Linux/Linux中的零拷贝技术/"}
---

#### 1. 传统数据拷贝发送

![20250420105125.png](/img/user/Resource/%E9%99%84%E4%BB%B6/20250420105125.png)
一共出现了**四次**数据拷贝
可以直接从内核缓冲区传输到 socket 缓冲区，少掉 2、 3 两步的拷贝
Linux 调用 sendfile 可以实现将数据从一个文件描述符传输到另一个文件描述符，从而实现了零拷贝技术。
Java 中  NIO FileChannel 类中的 transferTo() 方法就是依赖操作系统的零拷贝机制。
![20250420111422.png|725](/img/user/Resource/%E9%99%84%E4%BB%B6/20250420111422.png)

在 Linux 2.4 版本之后，开发者对 Socket Buffer 追加一些 Descriptor 信息来进一步减少内核数据的复制。如下图所示，DMA 引擎读取文件内容并拷贝到内核缓冲区，然后并没有再拷贝到 Socket 缓冲区，只是将数据的长度以及位置信息被追加到 Socket 缓冲区，然后 DMA 引擎根据这些描述信息，直接从内核缓冲区读取数据并传输到协议引擎中，从而消除最后一次 CPU 拷贝。
![20250420112722.png](/img/user/Resource/%E9%99%84%E4%BB%B6/20250420112722.png)
