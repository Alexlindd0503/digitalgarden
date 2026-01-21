---
{"dg-publish":true,"permalink":"/01.后端技术学习/并发/Happens-before/"}
---

happens-before 六项原则：
1. 程序的顺序性规则
2. volatile变量规则：对一个volatile变量的写操作，Happens-before于后续对这个变量的读操作
3. 传递性 ：A Hb B，B Hb C则A Hb C
4. 管程中的锁的规则：
	1. 对一个锁的解锁 Hb 于后续对这个锁的加锁
	2. 管程是一种通用的同步原语，在java中即synchronzied

5. 线程start()规则
	主线程A启动子线程B，B能看到A启动前的操作
	```java
	Theard B = new Thread(()->{
	      //主线程调用B.start()之前
	      //所有对共享变量的修改，此处皆可见，即此时var=77
	  })
	  //此处对共享变量修改
	var =77 
	B.start()
	```
6. 线程join的规则
线程A 调用线程 B 的 join 并成功返回，线程 B 中的任意操作 Hb 于该 join 操作的返回
```java
Theard B = new Thread(()->{
    //此处对共享变量修改
    var =66
})
B.start()
B.join()
//子线程所有对共享变量的修改
//在主线程调用join之后皆可见,即此时var=66
```