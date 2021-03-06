# 多核 CPU 的高速缓存 缓存不一致问题
> 背景：多核 CPU 时，每个 CPU 都有自己的高速缓存，在并发执行时，多个线程可能会分配到多个 CPU 核上。

```
// 例如
i = i + 1;
```

　　比如同时有2个线程执行上述示例代码，假如初始时i的值为0，那么我们希望两个线程执行完之后i的值变为2。但是事实会是这样吗？

　　可能存在下面一种情况：初始时，两个线程分别读取i的值存入各自所在的CPU的高速缓存当中，然后线程1进行加1操作，然后把i的最新值1写入到内存。此时线程2的高速缓存当中i的值还是0，进行加1操作之后，i的值为1，然后线程2把i的值写入内存。

　　最终结果i的值是1，而不是2。这就是著名的缓存一致性问题。通常称这种被多个线程访问的变量为共享变量。

# 如何解决缓存不一致问题

参考 [MESI 协议](/ji-suan-ji-ti-xi-jie-gou/can-kao-mesi-xie-yi.md)
