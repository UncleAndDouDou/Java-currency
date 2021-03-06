# 基于AQS的Lock

## 基于AQS的Lock

### interface Lock

Lock接口提供了锁功能。

* void lock\(\) // 阻塞式获取锁
* void lockInterruptibly\(\) throws InterruptedException
* boolean tryLock\(\) // 非阻塞式获取锁，无论成功失败直接返回
* boolean tryLock\(long time, TimeUnit unit\) throws InterruptedException
* Condition newCondition\(\) // 获取针对某种条件的等待队列
* void unlock\(\)

### Lock + AbstractQueuedSynchronized

* Lock对锁的使用者提供公共接口
* AQS对锁的开发者提供了锁的语义实现

### 如何实现基于AQS的Lock呢

#### 以TwinsLock为例

**要实现的同步规则**

该锁要实现的同步规则：在同一时刻，只允许至多两个线程同时访问，超过两个线程的访问将被阻塞。

**独占 or 共享？**

同一时刻支持多个线程访问，显然是共享的。

**AQS同步状态的设计**

同步状态state变化：合法值为0，1，2，初始为2，当一个线程获取，status减1，当线程释放，status+1

**AQS共享类方法设计**

* 使用模板方法acquireShared\(int arg\)等share相关方法
* 重写tryAcquireShared\(int arg\)方法

```text
/*
* Lock对锁的使用者提供公共接口
* AQS对锁的开发者提供了锁的语义实现
*/
public class TwinsLock implements Lock {
   /*
   * 通常将AQS封装到锁的静态内部类中
   * 锁的语义实现交由静态内部类来完成（最终是AQS）
   */
   private final Sync sync = new Sync(2);

   // 封装AQS
   private static final class Sync extends AbstractQueuedSynchronizer {
      Sync(int count) {
         if (count < 0) {
            throw new IllegalArgumentException("count must large than zero.");
         }
         setState(count); // AQS#setState
      }

      // 重写AQS#tryAcquireShared
      public int tryAcquireShared(int reduceCount) {
         for (;;) {
            int current = getState(); // AQS#getState()
            int newCount = current - reduceCount;

            // AQS#compareAndSetState()
            if (newCount < 0 || compareAndSetState(current, newCount)) { 
               return newCount;
            }
         }
      }

      // 重写AQS#tryReleaseShared
      public boolean tryReleaseShared(int returnCount) {
         for (;;) {
            int current = getState();
            int newCount = current + returnCount;
            if (compareAndSetState(current, newCount)) {
               return true;
            }
         }
      }

      /*
      * 锁对外提供的功能，最终都交由静态内部类实例来完成
      */
      public void lock() {
         sync.acquireShare(1);
      }

      public void unlock() {
         sync.releaseShared(1);     
      }

      // 其他略
   }   
}
```

**上述代码分析**

* 实现Lock接口
* 用静态内部类封装AQS
  * 为啥？？？
* 对外提供的锁功能，最终都由AQS实现
* 从语义上来讲，tryXXX类方法不应该以阻塞的方式实现，参考Lock.tryLock\(\)的定义，改为非阻塞试试

## 几种重要的同步规则

## 重入锁

> 互斥锁

支持一个线程对资源的重复加锁

支持获取锁时的公平/非公平（为什么可重入要和公平非公平捆绑？）

两个核心问题：

1. 可再次获取锁（锁要记录正在拥有锁的线程，才能保证该线程可再次获取锁）
2. 最终释放（要记录重入次数，释放时次数-1，次数为0时表示释放完成）

用state记录重入次数，因此本锁不支持共享（因为共享和重入次数都需要用state来保证）

#### 公平 or 非公平

公平：获取锁的顺序，严格按照请求的时间先后顺序，即FIFO（保证公平的核心：判断是否有前驱节点，有则表示在自己前面还有其他线程等待）

非公平：只要自旋阻塞过程中CAS获取锁成功，就获取了优先执行权，不按请求时间先后顺序来

## 读写锁

> 集共享与排他于一体，读是共享，写是排他
>
> 但读与写又不是毫无关联，写时不能有其他写或读

1. 如何仅通过一个同步状态state来同时达到共享与排他？
2. 可以用两个状态来实现读写锁吗？

#### 读写锁同步状态的设计

在一个同步状态上（整型）维护多个读线程和一个写线程

在一个整型上维护多种类型的状态，必然需要“按位切割使用”

高16位：读，低16位：写

同步状态值为S，写状态=S & 0x0000FFFF，读状态=S&gt;&gt;&gt;16

#### 写锁的获取与释放

获取：

1. 拿到同步状态 + 写状态
2. 状态 ！= 0，说明有读或者写
   1. 如果有读（写状态==0），失败
   2. 如果有写，但不是当前线程在写，失败
   3. 重入
3. 状态==0，说明无锁
   1. CAS获取锁

释放：

1. 检查是否是当前线程拥有锁
2. 写锁释放一次（重入对应的释放）

#### 读锁的获取与释放

获取：

1. 有写锁 && 不是当前线程在写，失败
2. 只要读状态&lt;MAX，就可以CAS获取读锁（因此也支持可重入）

#### 锁降级

> 把持住写锁，再获取读锁，然后释放写锁

为什么要锁降级？

## Condition接口

#### 监视器

每个Java对象都关联一个监视器，都拥有一组监视器方法（定义在Object中）

等待/通知模型：wait\(\) notify\(\) + synchromized

#### 等待通知模型

方式一：synchromized + wait\(\) notify\(\)

方式二：Lock + Condition

核心区别：Lock + Condition 支持多个等待队列

#### 监视器的队列模型

一个同步队列，一个等待队列

#### Lock+Condition的队列模型

一个同步队列，多个等待队列

（同步队列里的线程都处在竞争锁的状态中，而等待队列里的线程就是在等待）

