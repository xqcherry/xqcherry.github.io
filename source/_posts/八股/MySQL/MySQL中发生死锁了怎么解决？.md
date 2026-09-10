---
title: MySQL中发生死锁了怎么解决？
date: 2026-00-00
categories:
  - 八股
tags:
  - MySQL
  - 数据库
  - 后端
---

## MySQL中发生死锁了怎么解决？

死锁发生后解决方式分两种：自动处理和手动干预。

MySQL InnoDB 自带死锁检测机制，由 innodb_deadlock_detect 参数控制，默认开启。一旦检测到死锁，存储引擎会自动选择一个代价最小的事务回滚掉，释放它持有的锁让另一个事务继续执行。
*
手动干预主要用在自动机制不够快或者需要快速恢复的场景。先用 SHOW ENGINE INNODB STATUS 或者查 INFORMATION_SCHEMA 锁相关表找到阻塞的线程 ID，然后 KILL <thread_id> 杀掉它