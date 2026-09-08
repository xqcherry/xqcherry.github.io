---
title: MySQL 中的 MVCC 是什么？
date: 2026-00-00
categories:
  - 八股
tags:
  - MySQL
  - 数据库
  - 后端
---

## MySQL 中的 MVCC 是什么？

MVCC 是多版本并发控制。核心思想是让读写操作互不阻塞

- 写操作修改数据时，MySQL 不会立即覆盖原有数据，而是生成新版本的记录。每个记录保留了对应的版本号或时间戳。多版本之间串联起来就形成了一条版本链

- 读操作（普通读）就可以无锁地根据事务启动时间去版本链上找到属于自己的那个版本，此时读（普通读）写操作不会阻塞。

具体实现是：InnoDB 里每条记录都有两个隐藏字段：

`trx_id`：记录最后修改这条数据的事务 ID，也就是当前事务 ID

`roll_pointer`：指向 undo log 的指针

每次 UPDATE 不会覆盖原数据，而是把旧值写到 undo log 里，新值写到数据页，roll_pointer 指向旧版本，这样就串成了一条版本链
普通 SELECT 走的是快照读，不加锁，直接顺着版本链找到对自己可见的那个版本返回。写操作该怎么写怎么写，读写各走各的，并发性能直接拉满

![](/images/bagu/9.png)

### Undo Log 和版本链

MVCC 的"多版本"并不是真的存了好几份数据。InnoDB 只在数据页上存最新版本，历史版本都在 undo log 里。undo log 里记的是反向操作，比如 UPDATE 就记"改之前是啥"，DELETE 就记"删之前是啥"。需要历史版本的时候，顺着 roll_pointer 往回推就行

### ReadView 可见性判断

版本链有了，怎么判断哪个版本对当前事务可见？靠的是 ReadView。ReadView 有几个关键字段

`creator_trx_id`：当前事务 ID，只读事务这个值是 0

`m_ids`：生成 ReadView 时所有活跃事务的 ID 列表，就是已启动但还没提交的事务

`min_trx_id`：m_ids 里的最小值，也就是当前活跃 ID 之中的最小值

`max_trx_id`：下一个将被分配的事务 ID（事务 ID 是递增分配的，越后面申请的事务 ID 越大）

对于可见版本的判断是从最新版本开始沿着版本链逐渐寻找老的版本，如果遇到符合条件的版本就返回

判断版本可见性的规则：

1. 当前数据版本的 trx_id == creator_trx_id：说明修改这条数据的事务就是当前事务，所以可见

2. 当前数据版本的 trx_id < min_trx_id：说明修改这条数据的事务在当前事务生成 ReadView 的时候已提交，所以可见

3. 当前数据版本的 trx_id >= max_trx_id：修改这条数据的事务在生成 ReadView 之后才启动，不可见（结合事务 ID 递增来看）

4. min_trx_id <= 当前数据版本的 trx_id < max_trx_id：看 trx_id 在不在 m_ids 里，在就说明还没提交不可见，不在就说明已提交可见

### 读已提交和可重复读的区别

两个隔离级别判断版本可见性的逻辑一模一样，差别就一个：ReadView 生成时机不同。

读已提交：每次 SELECT 都重新生成 ReadView。别的事务提交了，下次查询就能看到。

可重复读：第一次 SELECT 生成 ReadView，后面的 SELECT 复用这个 ReadView。别的事务提交了也看不到，因为 ReadView 没变。

所以可重复读和读已提交的 MVCC 判断版本过程是一模一样的，唯一的差别在生成 ReadView 上。读已提交每次查询都会重新生成一个新的 ReadView，而可重复读在第一次生成 ReadView 之后的所有查询都共用同一个 ReadView，所以一个事务里面不论有几次 SELECT，其实看到的都是同一个 ReadView，所以叫可重复读

### 快照读和当前读

快照读：就是普通 SELECT，走 MVCC，读的是历史快照，不加锁。

当前读：是 SELECT ... FOR UPDATE、SELECT ... LOCK IN SHARE MODE、UPDATE、DELETE 这些，必须读最新版本并且加锁。道理很简单，你要 UPDATE 一条数据，不看最新版本怎么行？万一别的事务刚删了，你还 UPDATE 成功了那不乱套了。当前读会加 Next-Key Lock，锁住记录和间隙，防止别的事务插入新数据。

### 可重复读能完全避免幻读吗

不能。快照读和当前读混用的时候会出现问题。

举例：

开始事务

select * from A where id > 2（快照读），此时数据库只有 id 为 3 的数据，得到 1 条数据

事务 2 插入一条新数据提交了

事务 1 再 select * from A where id > 2 for update（当前读），要获取最新版本，所以能看到事务 2 插入的数据。两次查询结果行数不一样，幻读就出现了

解决办法是从头就用 SELECT ... FOR UPDATE，加上锁别的事务就插不进来了