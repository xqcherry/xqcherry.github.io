---
title: 说说Java中HashMap的原理
date: 2026-00-00
categories:
  - 八股
tags:
  - Java
  - Java集合
---

## 说说Java中HashMap的原理


1. HashMap底层是一个**数组**，结合**链表和红黑树**解决冲突
2. 存key时计算key的哈希，放到对应数组坐标下，如果出现冲突会以链表形式存在同一槽位，链表查询是O(n)
3. 如果某个桶的链表长度>=8且哈希表数组长度>=64，链表会自动转为红黑树，时间复杂度为O(logn)
4. 数组长度小于64不会树化，只会触发扩容, 如果桶节点<=6会自动退化为链表


### HashMap的Key可以用Null吗？

可以，HashMap允许一个null key，存的时候会把null key的hash值当作0，固定放在数组下标0的位置，但ConcurrentHashMap不允许null key，因为多线程场景下无法区分'key不存在'和'key存在但value是null'