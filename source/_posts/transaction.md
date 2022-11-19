---
title: 事务（一）：本地事务
date: 2022-10-19 22:30:54
tags:
- 数据库
- MySQL
- 事务
cover: https://pic.imgdb.cn/item/6378dcc116f2c2beb17e3666.jpg 
---

# 什么是事务

事务处理几乎在每一个信息系统中都会涉及，它存在的意义是为了**保证系统中所有的数据都是符合期望的，且相互关联的数据之间不会产生矛盾，即数据状态的一致性（Consistency）**。

按照数据库的经典理论，要达成这个目标，需要三方面共同努力来保障。

- **原子性**（**A**tomic）：在同一项业务处理过程中，事务保证了对多个数据的修改，要么同时成功，要么同时被撤销。
- **隔离性**（**I**solation）：在不同的业务处理过程中，事务保证了各自业务正在读、写的数据互相独立，不会彼此影响。
- **持久性**（**D**urability）：事务应当保证所有成功被提交的数据修改都能够正确地被持久化，不丢失数据。

上述特性被称为事务的ACID特性，这也是我比较赞同的一种解释，A、I、D 是手段，C 是目的。

---

# 本地事务

本地事务是最基础的一种事务解决方案，只适用于**单个服务使用单个数据源**的场景。从应用角度看，它是直接**依赖于数据源本身提供的事务能力**来工作的，在程序代码层面，最多只能对事务接口（start、commit、abort）做一层标准化的包装（如 JDBC 接口），并不能深入参与到事务的运作过程当中，事务的开启、终止、提交、回滚、嵌套、设置隔离级别，乃至与应用代码贴近的事务传播方式，全部都要依赖底层数据源的支持才能工作。

本文主要介绍本地事务的实现。

---

# 事务实现

这里以MySQL为例，看看通常是怎么实现ACID特性的，并不是MySQL的详细介绍。

## **实现原子性**

MySQL通过`undo log`实现原子性。

原子性需要保证对多条数据的操作要么全部写入成功，要么全部写入失败。但是有时会在写到一半的时候出现特殊情况导致不得不停止写入，比如执行过程中手动回滚、操作系统错误（比如磁盘满了）、断电等。这些情况都会导致事务执行到一半就结束，但是事务执行过程中可能已经修改了很多东西，为了保证事务的原子性，我们需要把东西改回原先的样子。

为了能够顺利地恢复到写入之前的状态，我们需要先记录**写入之前的数据**，这就是`undo log`。

下图undo log 的通用格式，一般insert和update/delete的undo log日志格式有所不同。

![未命名文件](https://pic.imgdb.cn/item/6377b78016f2c2beb1da25a3.png)

## 实现持久性

MySQL通过`redo log`实现持久性。

我们能够想到最简单实现持久性的方法就是：在事务提交完成之前把该事务所修改的所有页面都刷新到磁盘。这种方式称为`Commit Logging` ，即在log最后写入一个commit标识表示事务写入完成

但是为了提高性能，MySQL是先内存buffer中写入数据，之后再刷到磁盘上的，即使返回事务执行完成，所有的数据也不一定都刷到了磁盘上，这种方式称为`WAL（write-ahead-logging）`。

因为内存是易失性存储，为了保证持久性，即对内存buffer的修改不会丢失，我们需要在磁盘上**记录我们写入了什么数据**，包括写入的位置和实际值。

绝大部分类型的`redo`日志都有下面这种通用的结构，当然实际的结构肯定更复杂多样

![img](https://pic.imgdb.cn/item/6377b31816f2c2beb1d2cb83.png)

（附一张MySQL的写入流程图，可以更好的理解undo log 和 redo log）

![image](https://pic.imgdb.cn/item/6377a9b416f2c2beb19e4fb6.jpg)

## **实现隔离性**

隔离性保证了每个事务的操作不会彼此影响，理论上在某个事务对某个数据进行访问时，其他事务应该进行排队，当该事务提交之后，其他事务才可以继续访问这个数据。但是这样子的话对性能影响太大，所以这里的影响应该是相对的，即不同的影响程度对应不同的**隔离级别**。

- **串行化（SERIALIZABLE）**：事务之间完全独立，读和写都不会影响其他事务和被其他事务影响。
- **可重复读（REPEATABLE READ）**：在一个事务中，对同一字段的多次读取结果都是一致的，除非数据是被本事务修改，即限制其他事务对已读取数据的更新，但不限制其他事务的数据插入。
- **读已提交 （READ COMMITTED）**：只能读取其他事务已经提交的数据。
- **读未提交（READ UNCOMMITTED）**：可以读取其他事务未提交的数据。

### 不同隔离级别的问题

多个事务并发访问数据，势必会造成一些并发问题，在不同的隔离级别下，表现也不相同。

注意这里只讨论通用的隔离级别**可能**会出现的问题，并不具体到某一实现层面上，因为各个数据库产品实现的隔离级别支持的特性会有差异，导致某些问题在某些实现上不会出现，比如MySQL的可重复读的实现机制是可以避免一部分幻读的。

- 串行化：强制事务并发执行，因此不会出现并发方法

- 可重复读：这一隔离级别限制其他事务对已读取数据的更新，但不限制其他事务的数据插入，因此会出现经典的**[幻读](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Phantom_reads)**问题。

  - **幻读：在事务执行过程中，两个完全相同的范围查询得到了不同的结果集**。即当用户读取**某一范围**的数据行时，另一个事务又在该范围内**插入**了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。虽然两次读取到的数据集不同，但是这并没有违背可重复读隔离级别的定义，因为**可重读只要求第一次已经读到的数据在后续的读取中是相同的**。

    更具体点，幻读可以分为两种情况：

    - **只读事务**：事务T1连续两次范围读取
    - **读写事务**：事务T1先进行一次范围读取，然后根据第一次读取的结果，将范围内的数据进行更改

    ```sql
    -- 只读事务
    SELECT count(1) FROM books WHERE price < 100					  		/* 时间顺序：1，事务： T1 */
    INSERT INTO books(name,price) VALUES ('深入理解Java虚拟机',90); COMMIT;   /* 时间顺序：2，事务： T2 */
    SELECT count(1) FROM books WHERE price < 100; COMMIT;					/* 时间顺序：3，事务： T1 */
    
    
    -- 读写事务
    SELECT count(1) FROM books WHERE price < 100					  		/* 时间顺序：1，事务： T1 */
    INSERT INTO books(name,price) VALUES ('深入理解Java虚拟机',90);COMMIT;	   /* 时间顺序：2，事务： T2 */
    UPDATE books SET price = price*0.9 WHERE price < 100; COMMIT;			/* 时间顺序：3，事务： T1 */
    ```

- 读已提交：会造成[**不可重复读**](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Non-repeatable_reads)问题，它是指在事务执行过程中，对同一行数据的两次查询得到了不同的结果
- 读未提交：会造成[**脏读**](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Dirty_reads)问题，即读取到未提交的脏数据，因为可能会被修改或回滚。

### 不同隔离级别的实现

本节讨论如何实现不同的隔离级别，重点在于可重读隔离级别的实现，并且结合MySQL，进一步解决可重复读下会出现的幻读问题。

普遍实现隔离性的方式就是加锁，现代数据库均提供了以下三种锁：

- **写锁**（Write Lock，也叫作排他锁，eXclusive Lock，简写为 X-Lock）：如果数据有加写锁，就只有持有写锁的事务才能对数据进行写入操作，**数据加持着写锁时，其他事务不能写入数据，也不能施加读锁**。
- **读锁**（Read Lock，也叫作共享锁，Shared Lock，简写为 S-Lock）：多个事务可以对同一个数据添加多个读锁，**数据被加上读锁后就不能再被加上写锁，所以其他事务不能对该数据进行写入，但仍然可以读取**。对于持有读锁的事务，如果该数据只有它自己一个事务加了读锁，允许直接将其升级为写锁，然后写入数据。
- **范围锁**（Range Lock）：对于某个范围直接加排他锁，在这个范围内的数据不能被写入。

我们可以通过组合使用这些锁来实现不同的隔离级别，并尽可能保证并发度。

- **可串行化**：最最直接的方式实现可串行化就是设置一个全局锁，所有的事务都需要获取这个全局锁再进行操作，但是这样太慢了，连基本的并发读都不能支持。因此可以使用读写锁的方式提高性能，我们对事务所有读、写的数据全都加上读锁、写锁和范围锁可做到可串行化的隔离级别，具体可参考[2PL](https://en.wikipedia.org/wiki/Two-phase_locking)。

- **可重复读**：**对事务所涉及的数据加读锁和写锁，且一直持有至事务结束**，但不再加范围锁。其实这里的并发程度还是很低的，因为我们读取的时候还是会加写锁，以防止其他事务update当前事务已经读到的数据。

  MySQL 中InnoDB引擎是通过[MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)（Multi-Version Concurrency Control）来实现隔离性的，为了提高数据库并发性能，用更好的方式去处理**读-写冲突**，做到即使有读写冲突时，也能做到不加锁。MVCC的简单原理就是在事务**第一次**执行查询操作时，会记录下这次查询后的结果即快照，之后**查询**的时候就会返回这次快照的数据，天然的保证的一次事务中多次读取的结果相同。但是对于更新操作，需要去读最新的版本，防止产生写-写冲突。因此使用MySQL MVCC的方式实现可重复读可以提高读-写场景下的并发度，但如果是两个事务同时修改数据，即写-写的情况，那就没有多少优化的空间了，加锁几乎是唯一可行的解决方案，当然也有其他稍微优化的解决方案，比如各种乐观锁的实现。

  回过头来看幻读问题，一般的可重复读是无法解决幻读问题的，MySQL的MVCC机制可以解决幻读的第一种情况即只读事务，但无法解决第二种情况即读写事务。如果有需要，我们可以在事务的第一次查询上，显式的加上范围锁来完全解决幻读的问题，个人感觉这种做法已经到了可串行化的级别了。

  ```sql
  -- 读写事务
  SELECT count(1) FROM books WHERE price < 100 FOR UPDATE					/* 时间顺序：1，事务： T1 */
  INSERT INTO books(name,price) VALUES ('深入理解Java虚拟机',90);COMMIT;	   /* 时间顺序：2，事务： T2，由于加了范围锁，这里应该会被阻塞，等待T1执行完毕 */
  UPDATE books SET price = price*0.9 WHERE price < 100; COMMIT;			/* 时间顺序：3，事务： T1 */
  ```

- **读已提交**：对事务涉及的数据加的写锁会一直持续到事务结束，**但加的读锁在查询操作完成后就马上会释放**。
- **读未提交**：对事务涉及的数据只加写锁，会一直持续到事务结束，但**完全不加读锁**。



---

>  参考
>
>  [凤凰架构](http://icyfenix.cn/architect-perspective/general-architecture/transaction/local.html)
