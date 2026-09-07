---
title: CurrentHashMap怎么实现的？
date: 2026-00-00
categories:
  - 八股
tags:
  - Java
  - Java集合
---

## CurrentHashMap怎么实现的？

### JDK 1.7
JDK 1.7 采用**分段锁**设计, 底层把数组分为16个段，每个段里面都是一个完整的HashMap加可重入锁(ReentrantLock)
不同线程访问不同段不挡，同一段才有锁竞争,并发度最高16

![](/images/bagu/1.png)

### JDK 1.8
JDK 1.8 移除了段，锁粒度细化到了数组槽位，数据结构跟HashMap一致，底层是数组，链表和红黑树解决冲突
插入时先用**CAS无锁**尝试插入，**冲突**了才用**可同步锁(synchronized)**, 只锁树根节点或链表头节点
其它线程照样可以操作别的桶，并发度大大增加

![](/images/bagu/2.png)

两者区别:
![](/images/bagu/3.png)

### 1.8 为什么用 synchronized 而不是继续用 ReentrantLock？

主要是 synchronized 在 1.6 之后做了大量优化,包括偏向锁、轻量级锁,性能已经和 ReentrantLock 差不多了
用 synchronized 有几个好处：
1. 代码更简洁，不用显式加锁解锁
2. JVM 可以在运行时做更多优化，比如锁消除、锁粗化
3. synchronized 在 JIT 编译后有更好的内联

### 1.8 的渐进式扩容会不会导致读到不一致的数据？

不会。扩容时 ConcurrentHashMap 中保留新旧两个数组，读操作要么读到旧数组的数据，要么读到新数组的数据，都是正确的

### 1.7 的分段锁在计算 size 时为什么先尝试三次不加锁？

因为加锁代价太高了，要把 16 个 Segment 全锁住，等于整个 map 暂停服务,大部分情况下 map 没有并发写入，三次统计一样的概率很高，这时候直接返回就省了加锁的开销。只有真的有并发写入导致统计结果不一致时，才需要加锁来保证准确性

### ConcurrentHashMap 的 key 和 value 能不能为 null？为什么？

都不能为 null，会直接抛出 NullPointerException。原因是在并发环境下，null 会产生歧义
比如你调用 get(key) 返回 null，你分不清是 key 不存在还是 key 对应的 value 就是 null


