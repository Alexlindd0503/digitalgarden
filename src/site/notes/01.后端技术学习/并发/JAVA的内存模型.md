---
{"dg-publish":true,"permalink":"/01.后端技术学习/并发/JAVA的内存模型/"}
---

### 1. JMM（Java Memory Model）
什么是 JMM？
<span class="cloze-span">它是一个规定。
在多线程模式下，不同线程如何进行变量的读写，以及如何保证读写操作的正确性。</span>

### 2. 例子
```java title:示例代码
public class HelloWorld {
	private int data = 0;

    public void increment(){
        data++;
    }
}
HelloWorld helloWorld=new HelloWorld();
//线程1
new Thread(){
    public void run(){
        helloWorld.increment();
    }
}
//线程2
new Thread(){
    public void run(){
        helloWorld.increment();
    }
}
```

1. 对象在==堆内存==里，对象包含==实例变量==（不包括局部变量，方法参数，因为它是线程私有的）
2. 每个线程有一个自己的工作内存（CPU 级别的缓存）
3. 线程将主内存数据 load 进自己的工作内存，再进行操作
4. 数据变更后，再写入主内存中
![java内存模型.png](/img/user/Resource/%E9%99%84%E4%BB%B6/java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.png)