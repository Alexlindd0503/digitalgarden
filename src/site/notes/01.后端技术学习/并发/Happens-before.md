---
{"dg-publish":true,"permalink":"/01.后端技术学习/并发/Happens-before/"}
---


### 1. 📋 基本概念
Happens-Before 是 **Java 内存模型（JMM）**中的重要概念，用于定义操作之间的偏序关系，确保在并发环境下的**可见性**和**有序性**。

---

### 2. 🎯 八大核心规则

#### 2.1. **程序顺序规则（Program Order Rule）**

- 单个线程内，按照程序代码的执行顺序，书写在前面的操作 happens-before 书写在后面的操作
- 管理流控制依赖和操作依赖关系

#### 2.2. **监视器锁规则（Monitor Lock Rule）**
- unlock操作 happens-before 同一个锁的后续lock操作
- 确保锁释放前的所有修改对后续获取锁的线程可见

#### 2.3. **volatile 变量规则（Volatile Variable Rule）**
- volatile变量的写操作 happens-before 后续对该变量的读操作
- 保证volatile变量的可见性，禁止指令重排序

#### 2.4. **线程启动规则（Thread Start Rule）**
- Thread 对象的 start () 方法 happens-before 此线程的每一个动作
- 主线程启动子线程前的所有修改对子线程可见
```java
Theard B = new Thread(()->{
      //主线程调用B.start()之前
      //所有对共享变量的修改，此处皆可见，即此时var=77
  })
  //此处对共享变量修改
var =77 
B.start()

```

#### 2.5. **线程终止规则（Thread Join Rule）**
- 线程中的所有操作都 happens-before 对此线程的终止检测
- Join 操作能保证线程执行完成后，其他线程能看到其结果
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

#### 2.6. **线程中断规则（Thread Interruption Rule）**
- 线程 A 调用线程 B 的 interrupt () happens-before 线程 B 检测到中断发生
- 保证中断状态能被正确传递和检测

#### 2.7. **对象终结规则（Finalizer Rule）**
- 构造函数的结束 happens-before finalize () 方法的开始
- 保证对象在 finalize () 执行前构造完成

#### 2.8. **传递性规则（Transitivity）**
- 如果 A happens-before B，且 B happens-before C，那么 A happens-before C
- 这是整个体系的逻辑基础

---

### 3. 🔄 传递性示例

```java
int x = 1;        // 操作A
synchronized(lock) {
    x = 2;        // 操作B  
}
// 根据程序顺序规则：A happens-before B

Thread t = new Thread(() -> {
    synchronized(lock) {
        System.out.println(x); // 操作C
    }
});
t.start();

// 根据监视器锁规则：B happens-before C
// 根据传递性：A happens-before C
```

---

### 4. 💡 实际应用场景

**场景 1：共享变量同步**
```java
volatile boolean flag = false;

// 线程1
flag = true;  // 写操作

// 线程2  
if(flag) {    // 读操作，能看到flag的修改
    // do something
}
```

**场景 2：线程间通信**
```java
Thread t = new Thread(() -> {
    // 子线程工作
});
t.start();
t.join();  // 保证子线程完成
// 主线程能看到子线程的所有修改
```

---

### 5. 📝 记忆口诀

> **程 vol 终启传，中锁对象全**
> 
> - 程：程序顺序
> - vol：volatile 变量  
> - 终：线程终止
> - 启：线程启动
> - 传：传递性
> - 中：线程中断
> - 锁：监视器锁
> - 对象：对象终结

---

### 6. ⚠️ 重要提醒

- happens-before 不保证时间上的先后，只保证内存可见性
- 是 JMM 提供的最小保证，实际实现可能更优化
- 多线程编程建议优先使用高级并发工具类
