---
{"dg-publish":true,"permalink":"/01.后端技术学习/并发/Semaphore 信号量/"}
---

- 主要使用场景
	- Semaphore 可以允许多个线程访问一个临界区，如对象池，不允许多于 N 个线程进入临界区
- 基本原理
	- ![信号量机制原理.png|300](/img/user/Resource/%E9%99%84%E4%BB%B6/%E4%BF%A1%E5%8F%B7%E9%87%8F%E6%9C%BA%E5%88%B6%E5%8E%9F%E7%90%86.png)
	- 一个计数器、一个等待队列，三个方法
		- init 方法 设置计数器的初始值，代表多少线程可进入
		- down 方法，计数器减 1，如果此时计数器值<0，当前线程阻塞，进入等待队列
		- up，计数器加 1，如果此时计数器值<=0, 唤醒队列线程，并移除
- 示例代码
- ```java
  //共享资源
  static int count;
  static final Semaphore=s=new Semaphore(1);
  
  static void addone(){
  	//acquire与release需成对出现
  	s.acquire();
  	try{
  		count+=1;
  	}
  	finally{
  		s.release();
  	}
  }
  ```
- 示例代码二，多线程
- ```java 对象池
  class ObjPool<T, R> {  
      final List<T> pool;  
      final Semaphore sem;  
  	  //初始化
      ObjPool(int size, T t) {  
          pool = new Vector<T>() {};  
          for (int i = 0; i < size; i++) {  
              pool.add(t);  
          }  
          sem = new Semaphore(size);  
      }  
      
      //利用对象调用func  
      R exec(Function<T, R> func) throws InterruptedException {  
          T t = null;  
          sem.acquire();  
          try {  
              t = pool.remove(0);  
              return func.apply(t);  
          } finally {  
              pool.add(t);  
              sem.release();  
          }  
      }  
  }
  
  public static void main(String[] args) throws InterruptedException {  
    // 创建StringBuffer对象池，大小为3  
    ObjPool<StringBuffer, String> objPool = new ObjPool<>(3, new StringBuffer());  
        // 测试基本功能  
        String result = objPool.exec(buffer -> {  
            buffer.setLength(0); // 清空  
            buffer.append("Hello from object pool!");  
            return buffer.toString();  
        });  
          
}
  ```
