---
{"dg-publish":true,"permalink":"/01.后端技术学习/并发/Lock 与 Condition 机制/"}
---

- Lock 主要使用场景
	- 解决 synchronized 申请不到资源（进入休眠），不释放已占用资源的问题
- Lock 解决互斥，Condition 解决同步，如何解决不释放资源的设计思路如下
	- 能够响应中断 lockInterruptibly ()，获取不到后响应中断信号
	- 支持超时 tryLock (Long time, TimeUnit unit)
	- 非阻塞获取锁，tryLock ()
- 可见性的保障
	- 利用了 volatile 相关的 happens before 原则，锁内部有个 volatile 的变量 state，获锁前读取 state，解锁也会读取 state，结合顺序性规则、volatile 规则、传递性规则，即能做到可见性的保障。[[01.后端技术学习/并发/Happens-before\|Happens-before]]



[Java的内存模型#happens before](https://www.yuque.com/lindd/igiqy5/exi2ghgl24og52tg#ay0Rf)。

- 可重入锁：线程可以重复获取同一把锁
- 可重入函数：多个线程可以同时调用该函数，每个线程都能得到正确结果，线程安全的
- 公平锁与非公平锁：公平锁在唤醒的时候，谁等待时间最长唤醒谁

- Condition 是管程中的条件变量的队列
- Lock+Condition 可以实现异步转同步-RPC 调用