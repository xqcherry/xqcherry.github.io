---
title: Java中有哪些集合类?
date: 2026-00-00
categories:
  - 八股
tags:
  - Java
  - Java集合
---

## Java中有哪些集合类?

分Collection和Map两大类型

![](/images/bagu/4.png)

Collection下分三个小类: List, Set, Queue

**List**
ArrayList 底层是数组，查询快增删慢
LinkedList 底层是双向链表，增删快查询慢
Vector 是线程安全版的ArrayList，现在基本没人用了

**Set**
HashSet 基于哈希表，元素无序
LinkedListHashSet 用链表维护插入顺序
TreeSet 基于红黑树，元素按大小排序

**Queue**
PriorityQueue 是优先级队列
LinkedList 也实现了 Queue 接口，可以当普通队列用

![](/images/bagu/5.png)

**Map**
1. **HashMap** 最常用，基于哈希表
2. LinkedListHashMap 维护插入顺序，遍历时按放入顺序输出
3. TreeMap 基于红黑树，key 按大小排序
4. Hashtable 是远古时代的线程安全 Map, 现在被 ConcurrentHashMap 替代了
5. **ConcurrentHashMap** 是高并发场景的首选，分段锁设计，性能比 Hashtable 强很多

### ArrayList 和 LinkedList 到底该怎么选？

90% 的情况选 ArrayList
虽然理论上 LinkedList 插入删除快，但实际上 ArrayList 移动元素用的是 native 方法，做了优化，拷贝起来比想象中快得多，再加上 CPU 缓存友好性，ArrayList 在大多数场景下都更快
LinkedList 唯一的优势是当队列用

### ConcurrentHashMap 和 Hashtable 都是线程安全的，有什么区别？

**锁粒度不一样**
Hashtable 是整个表一把大锁，同一时刻只有一个线程能操作
ConcurrentHashMap 在 JDK 7 用分段锁，把表分成16个段，最多支持16个线程并发操作
JDK 8 取消了段机制，锁粒度细化到了数组，数据结构同HashMap，采用链表和红黑树解决冲突，插入时先用CAS无锁插入，冲突了才用synchronized

### HashMap 什么时候会退化成链表？

当大量 key 的冲突时，都落到同一个桶里，就会形成长链表，查找复杂度从 O(1) 退化到 O(n)，JDK 8 做了优化，当链表长度超过 8 且数组长度超过 64，会把链表转成红黑树，查找复杂度变成 O(log n)。当元素被删除到 6 个以下，又会退化回链表