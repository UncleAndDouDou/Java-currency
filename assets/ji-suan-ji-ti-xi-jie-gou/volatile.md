最核心：Volatile 要解决的问题是——保证修改对其他 CPU 立即可见。限制指令重排，保证写立即刷新到主存

---
举个例子：
```
//线程1
boolean stop = false;
while(!stop){
    doSomething();
}
 
//线程2
stop = true;
```
# 非 volatile 时可能发生的情况
当线程2更改了stop变量的值之后，但是还没来得及写入主存当中，线程2转去做其他事情了，那么线程1由于不知道线程2对stop变量的更改，因此还会一直循环下去。
# volatile 语义
1. 使用volatile关键字会强制将修改的值立即写入主存；
2. 使用volatile关键字的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量stop的缓存行无效；
3. 由于线程1的工作内存中缓存变量stop的缓存行无效，所以线程1再次读取变量stop的值时会去主存读取。

# 对 Volatile 的理解（待进一步验证）
> Volatile 涉及两个层次，一层是 Java 多线程内存模型，一层是多 CPU 高速缓存一致性

![](/assets/Volatile 多线程模型.png)
## Java多线程内存模型
参考[Java多线程内存模型](/jvm/java-nei-cun-mo-xing.md)
可知共享变量实际是被拷贝到了线程的本地内存中。
但是Java多线程内存模型会保证 Volatile 变量在多个线程间的可见性，保证能读到最新的值。

## 最值得思考的问题
一个线程某个时刻只可能在一个 CPU 上执行，也就是说线程本地内存中的 Volatile 变量副本此时只会被一个 CPU 访问，对于该线程中的 Volatile 变量副本来讲，并不会出现多个高速缓存不一致的情况，那么也就不存在高速缓存不一致的问题。

所以，如何保证 CPU 对 Volatile 变量副本的修改能及时得从高速缓存写回线程本地内存，再进一步由 Java 多线程内存模型保证对其他线程可见？（普通的缓存行被修改后，并不是立即写回的。参考[CPU、高速缓存、缓存行](/ji-suan-ji-ti-xi-jie-gou/cpu-huan-cun-yi-zhi-xing.md)）

CPU 并不区分一个变量是不是 Volatile 的（毕竟这只是 Java 特性）。

## 主动触发高速缓存一致性机制
对 Volatile 变量的写操作，在对应的汇编代码里会多出来一条 `lock` 开头的指令。

正是这条 lock 前缀的指令触发了缓存一致性机制， CPU 修改完高速缓存中的 Volatile 变量副本值后，将所在的缓存行写回线程本地内存，随后的事情由 Java 多线程内存模型来完成。