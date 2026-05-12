# MySQL八股文2

# 事务与MVCC

## MySQL是如何实现事务的？

MySQL 实现事务靠四个核心组件：Redo Log、Undo Log、锁、MVCC，它们分别对应事务 ACID 特性的不同方面。

- Undo Log 保证原子性。每次修改数据前，先把原值存到 undo log 里。事务回滚时，按 undo log 反向操作把数据恢复回去。要么全做完，要么全撤销，不会出现改了一半的中间状态。
- 锁机制（行锁、间隙锁等等）保证隔离性。两个事务同时改同一行，必须一个等另一个释放锁。InnoDB 的锁粒度精确到行级，还有间隙锁防止幻读。
- MVCC 保证隔离性的读写并发。读操作不加锁，通过 undo log 里的版本链找到自己应该看到的数据版本。写的时候别人照样能读，读的时候别人照样能写，并发度直接拉满。
- Redo Log 保证持久性。事务提交时，修改先写到 redo log 再写磁盘数据页。就算写数据页时宕机了，重启后通过重放redo log 就能恢复数据。这就是经典的 WAL 机制。
- 一致性不是单独实现的，它是原子性、隔离性、持久性共同作用的结果。数据从一个正确状态转移到另一个正确状态，中间不会出现不一致。

![image-20260302210411814](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260302210411814.png)

## 隔离性产生的问题

**脏读**: 一个事务读取到另一个事务未提交的数据

1. 在事务A执行过程中，事务A对数据资源进行了修改，事务B读取了事务A修改后的数据。

2. 由于某些原因，事务A并没有完成提交，发生了RollBack操作，则事务B读取的数据就是脏数据。

这种读取到另一个事务未提交的数据的现象就是脏读(Dirty Read)。

**不可重复读**: 

事务B读取了两次数据资源，在这两次读取的过程中事务A修改了数据，导致事务B在这两次读取出来的数据不一致。

这种在同一个事务中，前后两次读取的数据不一致的现象就是不可重复读(Nonrepeatable Read)。

**幻读**: 

事务A按照条件查询数据时，没有对应的数据行，但是在插入数据时，又发现这行数据已经存在，好像出现了幻觉。(由于解决了不可重复读，所以该事务读取不到别的事务已提交的数据)

**select 某记录是否存在，不存在，准备插入此记录，但执行 insert 时发现此记录已存在，无法插入，此时就发生了幻读。**

不可重复读和幻读区别：**不可重复读**的重点是**某条数据的内容**是否变化，**幻读**的重点在于**一批数据的数量是否变化**

## MySQL的事务隔离级别以及实现方式

![image-20260303004957975](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303004957975.png)

![image-20260303005141211](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303005141211.png)

ReadView 是一个用来判断**当前事务能看到哪些数据版本**的内部快照。

## 可重复读是如何解决不可重复读问题的？他能解决幻读问题吗？

解决不可重复读：

![image-20260303174710343](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303174710343.png)

幻读问题：

在 RR 级别下，MySQL 采用了两套机制共同对抗幻读：

**快照读（Snapshot Read）**

当你执行简单的 `SELECT` 语句时，MySQL 使用 **MVCC（多版本并发控制）**。

- **原理：** 系统会为你创建一个“一致性视图”（Read View）。即便其他事务插入了新数据，由于这些数据的版本号晚于你的 Read View，它们对你来说是“不可见”的。
- **效果：** 完美解决幻读。

**当前读（Current Read）**

当你执行 `SELECT ... FOR UPDATE`、`INSERT`、`UPDATE` 或 `DELETE` 时，MySQL 必须读取最新的数据。

- **原理：** 使用 **Next-Key Locks（记录锁 + 间隙锁）**。它不仅锁住存在的记录，还会锁住记录之间的“间隙”。
- **效果：** 只要你一直使用当前读，由于间隙被锁住，其他事务无法插入数据，从而解决了幻读。

**可重复读隔离级别下虽然很大程度上避免了幻读，但是还是没有能完全解决幻读。**在同一个事务中混用快照读和当前读的时候会出问题：

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303194619901.png" alt="image-20260303194619901" style="zoom: 67%;" />

事务 1 先快照读，拿到的是事务启动时的快照。事务 2 插入一条新数据提交了。事务 1 再当前读，因为当前读要获取最新版本，所以能看到事务 2 插入的数据。两次查询结果行数不一样，幻读就出现了。

解决办法是从头就用 `SELECT ... FOR UPDATE`，加上**临键锁**别的事务就插不进来了。

| **情况**     | **解决机制**   | **是否解决幻读** |
| ------------ | -------------- | ---------------- |
| **纯快照读** | MVCC           | **已解决**       |
| **纯当前读** | Next-Key Locks | **已解决**       |
| **混合读写** | N/A            | **未完全解决**   |

## 快照读和当前读

**快照读**就是普通 SELECT，走 MVCC，读的是历史快照，不加锁。

**当前读**是 `SELECT ... FOR UPDATE`、`SELECT ... LOCK IN SHARE MODE`、UPDATE、DELETE 这些，必须读最新版本并且加锁。道理很简单，你要 UPDATE 一条数据，不看最新版本怎么行？万一别的事务刚删了，你还 UPDATE 成功了那不乱套了。

当前读会加 Next-Key Lock，锁住记录和间隙，防止别的事务插入新数据。

## 什么是MVCC？

MVCC 全称 Multi-Version Concurrency Control,多版本并发控制。核心思想是让读写操作互不阻塞。

写操作修改数据时，MySQL不会立即覆盖原有数据，而是生成新版本的记录。每个记录都保留了对应的版本号或时间戳。多版本之间串联起来就形成了一条版本链。

读操作(普通读)就可以无锁地根据事务启动时间去版本链上找到属于自己的那个版本，此时读(普通读)写操作不会阻塞。

具体实现是：InnoDB 里每条记录都有两个隐藏字段

- trx_id 记录最后修改这条数据的事务 ID，也就是当前事务 ID
- roll_pointer：指向 undo log 的指针。

每次 UPDATE 不会覆盖原数据，而是把旧值写到 undo log 里，新值写到数据页，roll pointer 指向旧版本，这样就串成了条版本链。

普通 SELECT 走的是快照读，不加锁，直接顺着版本链找到对自己可见的那个版本返回。写操作该怎么写怎么写，读写各走各的，并发性能直接拉满。

![image-20260303192530141](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303192530141.png)

### **MVCC 能不能解决写写冲突？**

不能。MVCC 解决的是读写冲突，让普通读不阻塞写、写不阻塞普通读。写写冲突还是要靠锁来解决。两个事务同时 UPDATE 同一行，后执行的那个要等先执行的提交或回滚才能拿到锁。

## ReadView 可见性判断

版本链有了，怎么判断哪个版本对当前事务可见？靠的是 **ReadView**。ReadView 里有四个关键字段：

1）creator_trx_id：当前事务 ID，只读事务这个值是 0

2）m_ids：生成 ReadView 时所有活跃事务的 ID 列表，就是已启动但还没提交的事务

3）min_trx_id：m_ids 里的最小值，也就是当前活跃ID之中的最小值

4）max_trx_id：下一个将被分配的事务 ID（事务 ID 是递增分配的，越后面申请的事务ID越大）

**对于可见版本的判断是从最新版本开始沿着版本链逐渐寻找老的版本，如果遇到符合条件的版本就返回**。

判断版本可见性的规则：

1）当前数据版本的 trx_id == creator_trx_id：说明修改这条数据的事务就是当前事务，所以可见

2）当前数据版本的 trx_id < min_trx_id：说明修改这条数据的事务在当前事务生成 ReadView 的时候已提交，所以可见。

3）当前数据版本的 trx_id >= max_trx_id：改这条数据的事务在生成 ReadView 之后才启动，不可见(结合事务ID递增来看)

4）min_trx_id <= 当前数据版本的 trx_id < max_trx_id：看 trx_id 在不在 m_ids 里，在就说明还没提交不可见，不在就说明已提交可见

无论是在 RC 还是 RR 级别，当事务执行 `SELECT` 时，MySQL 都会严格按照你列出的 4 条规则，从最新版本开始沿着版本链对比 `trx_id`。

- **如果命中规则 1 或 2**：判定可见，停止寻找。
- **如果命中规则 3 或 4（且在 m_ids 中）**：判定不可见，顺着 `roll_pointer` 找老版本。

## 读已提交和可重复读的区别

两个隔离级别判断版本可见性的逻辑一模一样，差别就一个：ReadView 生成时机不同。

**读已提交**：每次 SELECT 都重新生成 ReadView。别的事务提交了，下次查询就能看到。

**可重复读**：第一次 SELECT 生成 ReadView，后面的 SELECT 复用这个 ReadView。别的事务提交了也看不到，因为 ReadView 没变。

举个例子说明**读已提交的情况**。

假设事务 1 已提交，事务 5 正在执行还没提交，事务 6 也在执行没提交。此时一个新事务执行 `SELECT name WHERE id=1`，由于查询的是 ID 为 1 的记录，所以先找到 ID 为 1 的这条记录，此时的版本如下：

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303195957442.png" alt="image-20260303195957442" style="zoom:50%;" />

此时 ReadView：

- creator_trx_id=0，因为一个事务只有当有修改操作的时候才会被分配事务 ID。
- m_ids=[5,6]，这两个事务都未提交，是活跃的。
- min_trx_id=5
- max_trx_id=7，因为最新分配的事务 ID 为 6，那么下一个就是7，事务 ID 是递增分配的

最新版本 trx_id=5，在 m_ids 里，对当前事务**不可见**。顺着 roll_pointer 通过版本链找到上一个版本 trx_id=1 的版本，比 min_trx_id 小，说明在生成 readView 的时候已经提交，对当前事务**可见**。因此返回 **name=XX**。

**然后事务 5 提交**，再次查询生成新的 ReadView：

- creator_trx_id 为 0，因为还是没有修改操作。
- m_ids 为 [6]，因为事务5提交了。
- min_trx_id=6
- max_trx_id=7，此时没有新的事务申请。

同样还是查询的是 ID 为 1 的记录，所以还是先找到 ID 为 1 的这条记录，此时的版本如下(和上面一样，没变)：

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303200023438.png" alt="image-20260303200023438" style="zoom:50%;" />

此时最新版本 trx_id=5，比 min_trx_id=6 小，说明事务已经提交了，对当前事务**可见**。返回 **name=NO**。

**这就是读已提交的 MVCC 操作**，同一个事务两次查询结果不一样，这就是不可重复读。

**如果现在的隔离级别是可重复读**。

那么第二次查询 **复用** 第一次的 ReadView，m_ids 还是 [5,6]，min_trx_id 还是 5，所以 trx_id=5 的版本还是不可见，两次都返回 name=XX。

所以可重复读和读已提交的 MVCC 判断版本的过程是一模一样的，**唯一的差别在生成 readView 上**。

**读已提交**每次查询都会重新生成一个新的 readView ，而**可重复读**在第一次生成 readView 之后的所有查询都共用同一个 readView，所以一个事务里面不论有几次 select ，其实看到的都是同一个 readView，所以叫**可重复读**。





# 日志

## MySQL中的日志类型有哪些？

MySQL 有三种核心日志，各司其职：**binlog** 负责主从复制和数据恢复，**redo log** 保证崩溃后数据不丢，**undo log** 支撑事务回滚和 MVCC。

1）binlog 是 Server 层的日志，记录的是逻辑操作，也就是原始 SQL 或者行变更前后的值。它的核心场景是主从同步，从库拉取主库的 binlog 重放一遍就能保持数据一致。另外做数据恢复的时候，也是靠 binlog 配合全量备份回放到指定时间点。

2）redo log 是 InnoDB 引擎独有的，记录的是物理变更，具体就是"某个数据页的某个偏移量改成了什么值"。它的作用是 crash-safe，MySQL 挂了重启后，InnoDB 会用 redo log 把没来得及刷盘的脏页恢复出来。redo log 是循环写的，空间固定，写满了就得等 checkpoint 推进才能继续。

3）undo log 也是 InnoDB 引擎独有的，记录的是数据修改前的旧值。事务回滚的时候，就靠 undo log 把数据改回去。另外 MVCC 的快照读也依赖它，别的事务要读历史版本，顺着 undo log 链往前找就行。

![image-20260303214124277](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303214124277.png)

三者的本质区别：binlog 记录的是数据变更的逻辑语义，由 Server 层生成，跨引擎通用。redo log 和 undo log 服务于 InnoDB 的事务与恢复机制，强依赖引擎内部实现，只有 InnoDB 才有。binlog 可以无限追加，redo log 是循环覆盖写。

### **三种日志对比**

| 维度     | binlog               | redo log           | undo log       |
| -------- | -------------------- | ------------------ | -------------- |
| 所属层级 | Server 层            | InnoDB 引擎        | InnoDB 引擎    |
| 日志类型 | 逻辑日志             | 物理日志           | 逻辑日志       |
| 写入方式 | 追加写，文件无限增长 | 循环写，固定大小   | 链式存储       |
| 核心作用 | 主从复制、数据恢复   | 崩溃恢复           | 事务回滚、MVCC |
| 事务相关 | 事务提交后写入       | 事务执行中持续写入 | 事务执行中写入 |

## Redo Log 的工作原理

InnoDB 用 Buffer Pool 缓存数据页，修改操作先改内存里的脏页，然后异步刷盘。问题来了，如果脏页还没刷盘 MySQL 就崩了，数据不就丢了？

最简单的方案是每次修改都立刻刷盘，但数据页是 16KB，改一个字节就要写 16KB，而且数据页在磁盘上是随机分布的，随机 IO 性能很差，一台普通 SSD 每秒也就几千次随机写。

redo log 的思路是**先写日志再写数据页**。redo log 是顺序追加写的，几十个字节就能记录一次修改，顺序 IO 性能比随机 IO 高一个数量级。只要 redo log 落盘了，就算数据页没刷，重启后也能恢复。这就是经典的 **WAL**，Write-Ahead Logging。

**先把修改操作顺序写到 redo log，再找机会把数据页刷到磁盘（刷到磁盘的这个过程依然是随机IO，只不过异步了）。顺序写比随机写快几个数量级。**

redo log 的写入流程分三步：

1）事务执行过程中，先把修改写到 redo log buffer，这块内存默认 16MB

2）事务提交时，根据 `innodb_flush_log_at_trx_commit` 参数决定刷盘策略：

- 设为 0，每秒刷一次，可能丢 1 秒数据
- 设为 1，每次提交都刷盘，最安全但性能最差
- 设为 2，每次提交写到 OS 缓存，MySQL 挂了不丢数据，服务器挂了可能丢

3）后台线程定期把脏页刷到磁盘，推进 checkpoint

redo log 采用循环写的方式，有两个指针：write pos 表示当前写到哪了，checkpoint 表示已经刷盘的位置。两个指针之间就是待刷盘的脏数据。

```sql
+---+---+---+---+
| 0 | 1 | 2 | 3 |   redo log 文件组
+---+---+---+---+
    ^       ^
    |       |
checkpoint  write_pos
```

![image-20260303222509552](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303222509552.png)

### **什么是“刷脏页”？**

**脏页（Dirty Page）是指在内存中被修改过，但还没有同步到磁盘上的数据页。**

为了性能，MySQL 修改数据时是先改内存（Buffer Pool），并记录日志（Redo Log），而不是直接写磁盘（磁盘太慢）。

- **过程：**后台线程会定期把内存里的修改后的页写回磁盘，这个动作叫**刷脏（Flushing）**。
- **页大小的影响：**如果你改了页里的 1 字节数据，刷脏时也要把整个 16KB（或 64KB）的页写回磁盘。
  - **写放大：**页越大，为了改一点点数据而必须写入磁盘的物理字节数就越多，对磁盘带宽压力越大。

## Undo Log 和版本链

MVCC 的"多版本"并不是真的存了好几份数据。InnoDB 只在数据页上存最新版本，历史版本都在 undo log 里。undo log 里记的是反向操作，比如 UPDATE 就记"改之前是啥"，DELETE 就记"删之前是啥"。需要历史版本的时候，顺着 roll_pointer 往回推就行。

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303192649419.png" alt="image-20260303192649419" style="zoom:50%;" />

拿 `INSERT (1, XX)` 举例，插入成功后数据页上除了 ID=1、name=XX，还有 trx_id=1 和 roll_pointer。这条 undo log 类型是 `TRX_UNDO_INSERT_REC`，里面存了主键信息。回滚的时候根据主键找到这条记录删掉就行。

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303192804756.png" alt="image-20260303192804756" style="zoom:50%;" />

事务 1 提交后，另一个事务 5 执行 `UPDATE name='NO' WHERE id=1`：

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303192906741.png" alt="image-20260303192906741" style="zoom:50%;" />

INSERT 产生的 undo log 在事务提交后就回收了，因为不可能有别的事务会访问比插入更早的版本。UPDATE 产生的 undo log 类型是 `TRX_UNDO_UPD_EXIST_REC`，不会马上删，因为可能有别的事务需要读历史版本。

事务 5 提交后，事务 11 再执行 `UPDATE name='Yes' WHERE id=1`：

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303192950302.png" alt="image-20260303192950302" style="zoom:50%;" />

这样版本链就串起来了，id=1 这条记录加上两条 undo log，一共三个版本。

### undo log 什么时候会被清理？

INSERT 产生的 undo log 在事务提交后立刻就能清理，因为没人会去读插入之前的版本。UPDATE 和 DELETE 产生的 undo log 要等到没有任何事务需要这个版本才能清。InnoDB 后台有个 purge 线程专门干这事，它会根据当前活跃事务的 ReadView 判断哪些 undo log 可以安全删除。长事务会导致 undo log 堆积，这也是为什么要尽量避免长事务。

## MySQL 事务的二阶段提交是什么？

二阶段提交是 MySQL 用来保证 **redo log** 和 **binlog** 数据一致性的机制。因为这两个日志分属不同层，一个是 InnoDB 引擎层的，一个是 Server 层的，如果写入过程中宕机，就可能出现两边数据对不上的问题。二阶段提交就是用来解决这个问题的。

整个过程分两步走：

1） **Prepare 阶段**：事务提交时，InnoDB 先把修改写到 redo log，但状态标记为 prepare，表示"我准备好了，但还没真正提交"

2） **Commit 阶段**：redo log 写完后，Server 层把操作写到 binlog。binlog 落盘成功，再通知 InnoDB 把 redo log 状态改成 commit，整个事务才算提交完成

![image-20260303222746324](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303222746324.png)

## binlog 有哪几种格式？各有什么优缺点？

三种格式： 

1）statement 格式记录原始 SQL，日志量小但可能导致主从不一致，比如 now() 函数在主从执行时间不同

2）row 格式记录行变更前后的完整数据，日志量大但绝对安全，生产环境基本都用这个

3）mixed 格式自动判断，普通语句用 statement，有风险的用 row，实际很少用因为判断逻辑不够智能

## MySQL 默认的事务隔离级别是什么？为什么选择这个级别？

MySQL 默认隔离级别是**可重复读** Repeatable Read，简称 RR。

选择这个级别是历史原因。早期 MySQL 的 binlog 只有 statement 格式，记录的是原始 SQL 语句。在读已提交级别下，并发事务提交顺序和实际执行顺序可能不一致，binlog 重放的时候就会出错，主从数据对不上。可重复读级别有间隙锁兜底，能保证 binlog 记录顺序和执行顺序一致，所以被选为默认值。

现在 binlog 有了 row 格式，记录的是行数据变更而不是 SQL，读已提交级别也不会有主从不一致问题了。但 MySQL 一直没改默认值，向前兼容嘛。

![image-20260304012500717](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304012500717.png)

### **大厂为什么改用读已提交**

互联网公司基本都把隔离级别调成读已提交，原因有几个：

1）binlog 现在默认用 row 格式了，记录的是数据变更不是 SQL，不存在主从不一致问题

2）可重复读的间隙锁范围大，高并发场景锁等待严重，死锁概率也高

3）可重复读要维护更多的 undo log 版本，长事务内存消耗大

4）大多数业务场景允许出现**“不可重复读”**问题

阿里内部规范明确要求用读已提交。用的时候把 binlog 格式改成 row 就行。

![image-20260304012903988](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304012903988.png)

## 你们生产环境的 MySQL 中使用了什么事务隔离级别？为什么？

MySQL 默认隔离级别是 RR，但很多互联网公司生产环境用的是 **RC**。主要原因是为了**提高并发**和**降低死锁概率**。

RR 级别下为了防止幻读，引入了间隙锁和临键锁，锁的范围更大。RC 只用行锁，锁定范围更小，并发能力更强。

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304015741151.png" alt="image-20260304015741151" style="zoom:50%;" />

# 锁

## MySQL 中有哪些锁类型？

MySQL InnoDB 的锁可以从两个维度来分：粒度和模式。

**粒度上分**：表锁和行锁。表锁锁整张表，行锁只锁具体的行，粒度越细并发越高。

**模式上分**：共享锁 S 锁和排他锁 X 锁。S 锁允许多个事务同时读，X 锁独占，读写都不让别人碰。

行锁又细分为三种：记录锁锁住具体那一行，间隙锁锁住两条记录之间的空隙防止插入，临键锁是记录锁加间隙锁的组合。

![image-20260304105535299](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304105535299.png)

还有几个辅助性质的锁：意向锁用来标记表里有没有行锁，加表锁的时候不用遍历；元数据锁 MDL 保护表结构，DDL 和 DML 不能同时跑；插入意向锁表示有事务在等着往某个间隙插数据；自增锁保证 AUTO_INCREMENT 分配不重复。

![image-20260304105644279](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304105644279.png)

## 锁的实现细节

InnoDB 的行锁实际上是加在索引上的，不是加在数据行上。如果 SQL 没走索引，就会锁全表，这一点很多人踩过坑.

InnoDB 有三种行锁:

- Record Lock:锁单条记录
- Gap Lock:锁一个区间，不包含记录本身
- Next-Key Lock: Record Lock + Gap Lock，锁记录和它前面的间隙

可重复读隔离级别下默认用 Next-Key Lock，就是为了防幻读。

![image-20260303190213121](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260303190213121.png)

**举个例子**

假设数据库表 `t_stu` 中有一列 `id`（主键），目前有两条数据：`id = 5` 和 `id = 10`。

- **间隙锁 (Gap Lock)**：
  - 如果你锁定了 (5, 10) 这个间隙。
  - **效果**：别人不能插入 `id = 6, 7, 8, 9` 的记录，但可以修改 `id = 5` 或 `id = 10` 的现有记录。
- **临键锁 (Next-Key Lock)**：
  - InnoDB 默认的锁定单位是 Next-Key Lock，它是一个**左开右闭**的区间。
  - 例如锁定区间 (5, 10]。
  - **效果**：别人既不能插入 `id = 6, 7, 8, 9`，也**不能修改或删除** `id = 10` 这条记录。

## MySQL 中如果发生死锁应该如何解决？

死锁发生后解决方式分两种：**自动处理**和**手动干预**。

MySQL InnoDB 自带死锁检测机制，由 innodb_deadlock_detect 参数控制，默认开启。一旦检测到死锁，存储引擎会自动选择一个代价最小的事务回滚掉，释放它持有的锁让另一个事务继续执行。

另外还有个兜底机制是锁等待超时 innodb_lock_wait_timeout，默认 50 秒，等锁超过这个时间就自动放弃并回滚。

手动干预主要用在自动机制不够快或者需要立刻恢复的场景。先用 `SHOW ENGINE INNODB STATUS` 或者查 INFORMATION_SCHEMA 里的锁相关表找到阻塞的线程 ID，然后 `KILL <thread_id>` 杀掉它。

![image-20260304113804672](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304113804672.png)

### **死锁日志分析**

线上遇到死锁第一步是拿日志。执行 `SHOW ENGINE INNODB STATUS` 找到 LATEST DETECTED DEADLOCK 那段，能看到两个互相等待的事务各自持有什么锁、在等什么锁。

举个实际的死锁日志片段：

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
170219 13:31:31
*** (1) TRANSACTION:
TRANSACTION 2A8BD, ACTIVE 11 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 376, 1 row lock(s)
MySQL thread id 448218, query id 18923238 root updating
delete from test where a = 2
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 0 page no 923 n bits 80 index `a` of table `oauthdemo`.`test` trx id 2A8BD lock_mode X waiting

*** (2) TRANSACTION:
TRANSACTION 2A8BC, ACTIVE 18 sec inserting
4 lock struct(s), heap size 1248, 3 row lock(s), undo log entries 2
MySQL thread id 448217, query id 18923239 root update
insert into test (id,a) values (10,2)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 0 page no 923 n bits 80 index `a` of table `oauthdemo`.`test` trx id 2A8BC lock_mode X locks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 0 page no 923 n bits 80 index `a` of table `oauthdemo`.`test` trx id 2A8BC lock mode S waiting

*** WE ROLL BACK TRANSACTION (1)
```

分析一下这个死锁的形成过程：

1）事务 1 执行 `delete from test where a = 2`，想要索引 a 上的 X 锁，但在排队等着 

2）事务 2 执行 `insert into test (id,a) values (10,2)`，它已经持有索引 a 上的 X 锁。因为 a 字段有唯一索引，插入前要先申请 S 锁做重复检查 

3）事务 2 申请 S 锁时发现前面有事务 1 在排队等 X 锁，于是也得等 

4）事务 1 等事务 2 释放锁，事务 2 等事务 1 让出队列位置，形成循环依赖

最后 MySQL 选择回滚事务 1，因为它持有的锁资源更少。

### **常见避免死锁的手段**

1）拆分大事务。事务越大持锁时间越长，死锁概率越高。能拆就拆，快进快出

2）固定加锁顺序。如果业务上必须同时操作表 A 和表 B，所有地方都先锁 A 再锁 B，循环依赖就不会形成

3）降低隔离级别。可重复读比读已提交多了间隙锁和临键锁，锁的范围更大。如果业务允许，用读已提交能减少锁冲突

4）合理建索引。没命中索引的更新会锁全表，几万行数据一锁，和别的事务撞上的概率大增

5）调整锁等待超时。innodb_lock_wait_timeout 默认 50 秒太长，高并发系统可以调到 5-10 秒，早点超时早点释放

### **手动排查死锁的完整步骤**

第一步，查当前锁信息：

```sql
-- MySQL 8.0 之前
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;

-- MySQL 8.0+
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

第二步，找到事务和线程的对应关系：

```sql
SELECT trx_id, trx_state, trx_started, trx_mysql_thread_id, trx_query
FROM INFORMATION_SCHEMA.INNODB_TRX;
```

trx_mysql_thread_id 就是线程 ID，trx_state 如果是 LOCK_WAIT 说明这个事务在等锁。

第三步，干掉造成阻塞的线程：

```sql
KILL 1234;  -- 替换成实际的线程 ID
```

注意 MySQL 8.0 把 INFORMATION_SCHEMA 里的 INNODB_LOCKS 和 INNODB_LOCK_WAITS 表移到了 performance_schema，旧的查法会报表不存在。

# 性能调优

## 如何在 MySQL 中监控和优化慢 SQL？ 

监控慢 SQL 的核心手段是开启 MySQL 的**慢查询日志**，它会把执行时间超过阈值的 SQL 自动记录下来。拿到慢 SQL 后，用 EXPLAIN 分析执行计划，找出问题所在，再针对性优化。

整个流程分三步：

1）开启慢查询日志，设置合理的时间阈值，线上一般设 1-2 秒

2）定期分析慢查询日志，用 mysqldumpslow 或 pt-query-digest 工具汇总统计

3）对高频慢 SQL 用 EXPLAIN 分析，检查是否走了索引、扫描了多少行、有没有文件排序等，再针对性优化

```sql
-- 查看慢查询日志是否开启
SHOW VARIABLES LIKE 'slow_query_log';
-- 查看慢查询阈值
SHOW VARIABLES LIKE 'long_query_time';
-- 查看慢查询日志路径
SHOW VARIABLES LIKE 'slow_query_log_file';
```

### **慢查询日志配置**

生产环境排查问题，慢查询日志是第一手资料：

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
-- 设置阈值，执行时间超过 1 秒的 SQL 会被记录
SET GLOBAL long_query_time = 1;
-- 记录没有使用索引的查询
SET GLOBAL log_queries_not_using_indexes = ON;

-- 查看配置
SHOW VARIABLES LIKE '%slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

日志文件路径一般在`/var/lib/mysql/xxx-slow.log`，可以用`mysqldumpslow`工具分析：

```sql
# 按照查询时间排序，取前 10 条
mysqldumpslow -s t -t 10 /var/lib/mysql/xxx-slow.log

# 按照扫描行数排序
mysqldumpslow -s r -t 10 /var/lib/mysql/xxx-slow.log
```

不过现在一般用云厂商提供的服务器了，面板上能直接查看和搜索慢 SQL

### **线上突然出现大量慢查询，排查思路是什么?**

先看是不是某个时间点突然出现的。如果是，查一下那个时间有没有发布、有没有跑批任务。然后看慢查询日志，找出最频繁的几条 SQL。用 EXPLAIN 分析执行计划，检查索引有没有失效。同时看一下服务器指标，CPU、磁盘 I/O、连接数是不是打满了。如果是锁等待导致的，用`SHOW PROCESS LIST`和`SHOW ENGINE INNODB STATUS` 排查。

## 如何进行sql调优

SOL 调优的核心思路就是减少磁盘 I/0 和避免无效计算。实际操作分三步走：先定位慢 SQL、再分析执行计划、最后针对性优化。

定位慢 SQL 靠 MySQL 的慢查询日志，分析执行计划用 EXPLAIN，优化手段主要有这几类:

1）索引层面优化

- 合理设计联合索引，利用覆盖索引避免回表。比如査询只需要 name 和 age，那索引建成(name，age)就不用回表去主键索引拿数据了
- 注意最左匹配原则，`WHERE b=1`吃不到(a，b，c)这个联合索引
- 避免在索引列上做函数运算， `WHERE YEAR(create_time) = 2024` 会让索引失效，改成 `WHERE create_time >= '2024-01-01' AND  create_time < '2025-01-01'`

2）SQL写法优化

- 禁止`SELECT *`，只查必要字段，减少网络传输和内存占用
- 避免`%LIKE`前缀模糊査询，`LIKE '%关键词'`必然全表扫描
- 连表査询时检査字段字符集是否一致，utf8 和 utf8mb4 的字段 JOIN 会导致隐式转换，索引直接废掉

3）架构层面优化

- 热点数据上 Redis 缓存，访问频率高但变化少的数据没必要每次都査库
- 大表考虑分库分表，单表超过 2000 万行查询性能会明显下降
- 读写分离，把查询压力分摊到从库

还可以通过业务来优化，例如少展示一些不必要的字段，减少多表査询的情况，将列表查询替换成分页分批查询等等

### **大表优化策略**

当单表数据量上来之后，光靠索引已经不够用了。MySQL 单表数据太多的话，即使走索引，B+ 树层级也会变深，查询性能下降明显。

分页优化是大表场景的高频问题。 LIMIT 1000008，10 这种深分页会扫描前 100 万条数据然后丢弃，非常浪费。优化方案是用游标分页：

```sql
-- 深分页，性能差
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- 游标分页，性能好
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

冷热数据分离也很有效。把3个月前的订单挪到历史表，主表只保留热数据，查询压力小很多。

### **什么情况下 MySQL 优化器会放弃使用索引？**

优化器是基于成本估算的，它认为全表扫描更快就不走索引。典型场景是査询结果集占总数据量比例太高，比如表里100 万条数据，查询条件能匹配 60 万条，走索引反而要多一次回表操作，不如直接全表扫。另外统计信息不准也会导致优化器误判，这时候可以用 ANALYZE TABLE 更新统计信息。

## MySQL 中如何解决深度分页的问题？

深度分页的核心问题在于 `LIMIT 99999990, 10` 这种写法会让 MySQL **扫描前 99999990 条记录再丢弃**，白白浪费大量 IO。解决思路就是想办法减少扫描的数据量，常见的优化方案有三种：

**1）子查询优化**

把原本的查询拆成两步：先用子查询在二级索引上快速定位起始 id，再用这个 id 去主键索引取数据。

```sql
-- 原始写法，慢
SELECT * FROM mianshiya
WHERE name = 'yupi'
ORDER BY id
LIMIT 99999990, 10;

-- 优化写法
SELECT * FROM mianshiya
WHERE name = 'yupi'
  AND id >= (
    SELECT id FROM mianshiya
    WHERE name = 'yupi'
    ORDER BY id
    LIMIT 99999990, 1
  )
ORDER BY id
LIMIT 10;
```

name 字段有索引的情况下，子查询只扫描 name 的二级索引，二级索引只存了 name 和 id，数据量比主键索引小很多。拿到起始 id 后，再去主键索引取 10 条完整记录，速度很快。

用 JOIN 写法也是一样的原理：

```sql
SELECT * FROM mianshiya
INNER JOIN (
  SELECT id FROM mianshiya
  WHERE name = 'yupi'
  ORDER BY id
  LIMIT 99999990, 10
) AS t ON mianshiya.id = t.id;
```

![image-20260304121147172](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304121147172.png)

**2）游标分页**

每次查询都返回当前页的最大 id，下次查询时带上这个 id 作为起点：

```sql
-- 第一页
SELECT * FROM mianshiya WHERE name = 'yupi' ORDER BY id LIMIT 10;
-- 假设最大 id 是 100

-- 第二页
SELECT * FROM mianshiya WHERE name = 'yupi' AND id > 100 ORDER BY id LIMIT 10;
```

这种方式利用 `id > maxId` 直接过滤，MySQL 可以从索引定位到起始位置，不用扫描前面的数据。缺点是只能连续翻页，没法跳到第 10000 页。

![image-20260304121426111](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304121426111.png)

**3）搜索引擎**

把数据同步到 Elasticsearch，用 search_after 做深度分页。不过 ES 也有深度分页问题，用 search_after 配合 PIT 才能高效处理。如果对 ES 不熟，面试时慎用这个答案。

![image-20260304121959748](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304121959748.png)

### **为什么深度分页会慢**

MySQL 执行 `LIMIT 99999990, 10` 时，并不是直接跳到第 99999990 条记录，而是从第一条开始扫描，数够 99999990 条后丢弃，再取后面 10 条返回。

如果走主键索引，相当于把前 1 亿条记录都读了一遍。即使走二级索引，也要扫描 1 亿次索引项，再回表 10 次。这种 O(n) 的时间复杂度，数据量一大就扛不住。

### **子查询优化的原理**

子查询优化的关键在于**延迟关联**：

1）子查询 `SELECT id FROM ... LIMIT 99999990, 1` 只扫描二级索引。二级索引是个小表，只存了索引列和主键，不包含其他字段，扫描速度比主键索引快很多。

2）拿到起始 id 后，`id >= xxx LIMIT 10` 在主键索引上是个范围查询，B+ 树可以直接定位到这个 id，然后顺序读 10 条记录，不用从头扫描。

假设一张表有 1 亿条记录，每条记录 1KB，主键索引大约 100GB。如果 name 字段建了索引，二级索引可能只有 2-3GB。扫描 2GB 和扫描 100GB 的差距是巨大的。

### **游标分页的局限**

游标分页虽然性能好，但有几个限制：

1）只能连续翻页，没法直接跳到第 N 页。用户习惯了 "跳到第 100 页" 的功能就没法支持了。

2）必须有一个唯一且有序的字段做游标，一般用主键 id。如果排序字段不唯一，比如按创建时间排序且时间可能重复，就要用联合游标 `(create_time, id)`。

3）删除数据可能导致分页错乱。比如用户在看第 2 页时，第 1 页的数据被删了，刷新后原本在第 2 页的内容会“顶”到第 1 页，导致翻页时漏掉部分信息。

实际项目中，列表页通常用游标分页，跳页需求就限制最大页数，比如百度搜索结果最多显示 68 页。

# 主从同步

## 什么是 MySQL 的主从同步机制？它是如何实现的？

MySQL 主从同步的核心就是 **binlog 复制**：主库把写操作记到二进制日志里，从库拉过来重放一遍，数据就同步了。

整个流程涉及三个线程配合：

1）主库的 dump 线程：监听 binlog 变更，有新内容就推事件给从库

2）从库的 I/O 线程：拉取主库数据，把收到的 binlog 写进本地的 relay log

3）从库的 SQL 线程：读 relay log，逐条执行 SQL 语句

![image-20260304124008058](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304124008058.png)

### **三种复制模式**

MySQL 支持异步、同步、半同步三种复制模式，区别在于主库什么时候给客户端返回响应：

| 复制模式   | 主库返回时机         | 性能 | 数据可靠性               |
| ---------- | -------------------- | ---- | ------------------------ |
| 异步复制   | 写完 binlog 立即返回 | 最高 | 最低，主库挂了数据可能丢 |
| 同步复制   | 等所有从库确认       | 最差 | 最高                     |
| 半同步复制 | 等至少 N 个从库确认  | 折中 | 较高                     |

**MySQL 默认是异步复制**，主库写完 binlog 就直接返回，压根不管从库有没有收到。好处是快，坏处是主库突然挂了，那些还没同步过去的数据就丢了。

同步复制要等所有从库都确认收到才返回，一般没人用，太慢了。

### **半同步复制**

MySQL 5.5 引入了半同步复制插件，5.7 做了增强。核心思路是折中：不用等所有从库，只要有一个从库确认收到就行。

```sql
-- 主库开启半同步
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
-- 至少等 1 个从库确认
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;
```

比如 3 个从库，配置等 1 个确认，只要有一个从库说"收到了"，主库就返回响应。这样只有那个最快的从库和主库同时挂掉，数据才会丢。

**5.7 还区分了两种半同步模式**：

1） AFTER_COMMIT：主库先提交事务，再等从库确认。万一主库提交后、从库确认前主库挂了，新主库上没这条数据，但老主库的客户端已经收到成功响应了，造成数据不一致

2） AFTER_SYNC：主库先等从库确认，再提交事务。这样主库挂了，新主库上一定有这条数据，更安全

## 如何处理 MySQL 的主从同步延迟？

先明确一点：**主从延迟是必然存在的**，不可能完全消除，只能尽量缩短延迟时间或者在业务层面做规避。

业务层面常见的处理方案：

1）关键业务强制走主库。比如用户注册完立马登录，这种写后读的场景直接走主库，不走从库。虽然牺牲了读写分离的意义，但这类操作频次不高，对主库压力有限

2）延迟感知。写操作后记录时间戳，短时间内的读请求强制走主库，过了延迟窗口再走从库。可以用 ThreadLocal 或 Redis 记录写操作时间

3）二次查询兜底。从库查不到就再查一次主库，属于兜底策略。问题是如果有人恶意查询不存在的数据，每次都打到主库，变相攻击

4）缓存前置。写入主库的同时写入缓存，读请求先查缓存。不过又引入了缓存一致性问题，属于拿一个问题换另一个问题

除此之外，也可以提一提配置问题，例如主库的配置高，从库的配置太低了，可以提升从库的配置等。如果面试官对 MySQL 比较熟，可能会追问一些偏 DBA 侧的问题，例如并行复制等。

### **延迟的根本原因**

从库复制主库的流程是：主库写 binlog → dump 线程推送变更事件 → 从库拉取数据 → 从库 I/O 线程写 relay log → SQL 线程重放。只要这条链路上任何一个环节慢了，延迟就上来了。

![image-20260304124008058](MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304124008058.png)

常见原因和对应优化：

| 原因           | 现象                             | 优化方案                                                     |
| -------------- | -------------------------------- | ------------------------------------------------------------ |
| 从库单线程重放 | Seconds_Behind_Master 持续增长   | 开启并行复制，[MySQL 并行复制](https://www.mianshiya.com/question/1780933295538728962#heading-6) |
| 大事务         | 单个事务执行几分钟甚至更久       | 拆分大事务，[MySQL 中长事务可能会导致哪些问题？](https://www.mianshiya.com/bank/1791003439968264194/question/1780933295480008706#heading-0) |
| 从库配置低     | 从库 CPU/IO 打满                 | 升级从库硬件配置                                             |
| 从库查询压力大 | 从库慢查询多，SQL 线程抢不到资源 | 增加从库数量，优化慢查询                                     |
| 网络延迟       | 跨机房部署                       | 缩短主从物理距离，用专线                                     |
| 从库太多       | 主库 dump 线程压力大             | 用级联复制，从库再挂从库                                     |

### **并行复制优化**

早期从库只有一个 SQL 线程串行重放，主库写得快从库跟不上。MySQL 5.6 开始引入并行复制，不同库的事务可以并发执行。5.7 引入 LOGICAL_CLOCK，同一 group commit 的事务可以并发。8.0 引入 WriteSet，只要更新的行不冲突就能并发。

```sql
-- 查看从库并行复制配置
SHOW VARIABLES LIKE 'slave_parallel%';

-- 开启并行复制（MySQL 5.7+）
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8;
```

`slave_parallel_workers` 设成 CPU 核心数的 2 倍左右，太多了线程切换开销大，太少了并行度不够。

<img src="MySQL%E5%85%AB%E8%82%A1%E6%96%872.assets/image-20260304124323429.png" alt="image-20260304124323429" style="zoom:50%;" />

### **监控延迟**

最常用的是 `SHOW SLAVE STATUS` 里的 `Seconds_Behind_Master`，表示从库落后主库多少秒。但这个值不一定准，它算的是 relay log 里最后一条事件的时间戳和当前时间的差值，如果主库长时间没写入，这个值会是 0，但实际可能网络已经断了。

更靠谱的做法是用 pt-heartbeat 这类工具。原理是在主库定时写入时间戳，从库查出来和当前时间比较，算出真实延迟。

```bash
# 主库启动心跳写入
pt-heartbeat --database=test --table=heartbeat --update --daemonize

# 从库监控延迟
pt-heartbeat --database=test --table=heartbeat --monitor
```

### **半同步复制**

如果对数据一致性要求高，可以开启半同步复制。主库等至少一个从库确认收到 binlog 再返回客户端，这样从库延迟最多就是一个事务的执行时间，不会无限累积。代价是写性能下降，每次写操作多了一次网络往返。

```sql
-- 主库开启半同步
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 超时时间，毫秒
```

如果从库在 timeout 时间内没响应，主库会自动降级成异步复制，不会一直阻塞。
