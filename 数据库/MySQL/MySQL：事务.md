# MySQL 事务

## 1. 事务日志：redo log 与 undo log

InnoDB 事务日志包括 redo log 和 undo log。

| 日志 | 作用 | 类型 |
|:---|:---|:---|
| **redo log（重做日志）** | 保证事务的持久性 | 物理日志，记录数据页的修改 |
| **undo log（回滚日志）** | 保证事务的原子性、支持 MVCC | 逻辑日志，记录反向操作 |

> 事务的概念、ACID、隔离级别等通用内容参见 [事务.md](../理论/事务.md)。

### redo log

- MySQL 每执行一条语句，先将记录写入 redo log buffer，再写入 OS cache，最终通过 fsync() 刷入 redo log 文件
- Redo log 采用**大小固定、循环写入**的方式，写到结尾会回到开头
- MySQL 提供 buffer pool 提高读取速度。数据修改先写入 buffer pool，若在刷盘前宕机，可通过 redo log 恢复数据
- **LSN（逻辑序列号）**：单调递增，记录在 redo log 和数据页中，用于判断数据页是否领先于 redo log

#### 刷盘策略

可通过 `innodb_flush_log_at_trx_commit` 参数控制：
- **0**：每秒刷入磁盘一次（可能丢失 1 秒数据）
- **1**（默认）：每次事务提交都刷入磁盘（最安全）
- **2**：每次事务提交写入 OS cache，每秒刷盘

### undo log

- 当事务对数据库进行修改时，InnoDB 会生成对应的 undo log
- 如果事务执行失败或回滚，利用 undo log 将数据回滚到修改之前
- Undo log 是**逻辑日志**：insert 对应 delete，delete 对应 insert，update 对应相反的 update
- Undo log 还用于 MVCC，为其他事务提供旧版本快照

## 2. binlog（二进制日志）

Binlog 是 MySQL **Server 层**的日志，以事件形式记录所有数据修改操作（不包括 SELECT）。

| 特性 | 说明 |
|:---|:---|
| 产生位置 | Server 层（所有存储引擎通用） |
| 记录内容 | 逻辑 SQL 语句或数据行的变更 |
| 写入时机 | 事务提交时写入 |
| 用途 | 主从复制、数据恢复 |

### binlog 三种格式

| 格式 | 说明 | 优点 | 缺点 |
|:---|:---|:---|:---|
| **STATEMENT** | 记录 SQL 语句 | 日志量小 | 部分函数可能导致主从不一致 |
| **ROW**（默认） | 记录每行数据变更 | 最安全，一致性好 | 日志量大 |
| **MIXED** | 混合模式，自动选择 | 兼顾两者 | 略复杂 |

### 刷盘时机

通过 `sync_binlog` 参数控制：
- **0**：系统自行判断
- **1**：每次 commit 都刷盘
- **N**：每 N 个事务刷盘一次

## 3. redo log vs binlog

| 对比项 | redo log | binlog |
|:---|:---|:---|
| 产生层 | InnoDB 存储引擎层 | MySQL Server 层 |
| 记录形式 | 物理日志（数据页修改） | 逻辑日志（SQL 语句或行变更） |
| 写入时机 | 事务执行过程中不断写入 | 事务提交完成后写入 |
| 空间管理 | 固定大小，循环写入 | 追加写入，可设置大小，可滚动 |
| 用途 | 崩溃恢复、保证持久性 | 主从复制、时间点恢复 |

### redo log 和 binlog 一致性：内部 XA 两阶段提交

MySQL 通过两阶段提交（2PC）保证 redo log 和 binlog 的一致性：

**第一阶段（Prepare）**：InnoDB 写 redo log，事务状态变为 Prepare（不记录 commit 标签）

**第二阶段（Commit）**：
1. 写 binlog（write → fsync）
2. InnoDB 在 redo log 中写入 commit 标签

**为什么这样设计？**—
- 如果在写 binlog **之前**崩溃：恢复后回滚事务，binlog 中无该事务，主从一致
- 如果在写 binlog **之后**崩溃：恢复后对比 binlog 中的 xid，若存在则提交该事务（即使 redo log 没有 commit 标签），主从一致

这样以 binlog 的写入作为事务成功提交的标志，解决了两个日志的一致性问题。

## 4. InnoDB MVCC 实现原理

MVCC（Multi-Version Concurrency Control，多版本并发控制）让 InnoDB 在 **不加锁** 的情况下实现「读写不阻塞」：读操作访问某个历史版本，写操作生成新版本，两者互不干扰。它是 READ COMMITTED 和 REPEATABLE READ 两个隔离级别的实现基础。

### 先记住三件套

| 组件 | 存放位置 | 作用 |
|:---|:---|:---|
| **隐藏字段**（trx_id、roll_ptr、row_id） | 聚簇索引记录 | 标记「这条记录是哪个事务改的」「上一个版本在哪」 |
| **undo log 版本链** | undo 表空间 | 保存历史版本，串成一条链 |
| **ReadView** | 内存（事务私有） | 判断「版本链上哪个版本对当前事务可见」 |

一句话：**版本链负责「存」旧版本，ReadView 负责「选」可见版本**。先有 ReadView，再沿版本链从新到旧找第一个可见版本。

### 隐藏字段

InnoDB 聚簇索引的记录除了用户定义的列，还有几个隐藏信息：

| 字段 | 长度 | 说明 |
|:---|:---|:---|
| **DB_TRX_ID** | 6 字节 | 最后一次插入或修改该记录的事务 ID |
| **DB_ROLL_PTR** | 7 字节 | 回滚指针，指向 undo log 中该记录的上一个版本 |
| **DB_ROW_ID** | 6 字节 | 隐藏自增 ID，表没有主键且没有唯一非空索引时使用 |
| **deleted_flag** | 1 bit（记录头） | 删除标记。`DELETE` 只是把它置 1，并非物理删除 |

> 注意：`deleted_flag` 不在隐藏列里，而在记录头信息中，但它参与可见性判断——被标记删除的记录，只有「删除操作对自己不可见」时才不会被读到。

### undo log 与版本链

每条记录通过 `DB_ROLL_PTR` 指向 undo log 里的上一个版本，而每个 undo 版本同样带 `trx_id` 和 `roll_ptr`，于是形成一条**从新到旧的单向链表**：

```
当前行 (trx_id=30)
   │ roll_ptr
   ▼
undo 版本1 (trx_id=20) ──roll_ptr──► undo 版本2 (trx_id=10)
```

两种 undo log 的生命周期不同：

| 类型 | 产生时机 | 何时可删 |
|:---|:---|:---|
| **insert undo** | `INSERT` | 事务提交后即可删除（插入的记录本来对其他事务不可见，无需保留旧版本） |
| **update undo** | `UPDATE` / `DELETE` | 供 MVCC 使用，提交后仍保留，由 **purge 线程**在确认没有事务需要时清理 |

补充两点：

- `UPDATE` 生成 undo 时记录的是**反向操作**（把被改列改回旧值），因此版本链上的「旧版本」是按需回放出来的，不是整行复制；
- `DELETE` 生成 update undo，并把记录头 `deleted_flag = 1`，版本链上仍能找到它，所以删除后长事务依然能读到。

### ReadView

ReadView 是事务执行快照读时生成的一个「可见性快照」，包含四个字段：

| 字段 | 含义 |
|:---|:---|
| **m_ids** | 生成 ReadView 时，系统中**活跃（未提交）**的读写事务 ID 列表 |
| **min_trx_id** | m_ids 中的最小值 |
| **max_trx_id** | 生成 ReadView 时系统**下一个将要分配**的事务 ID（即 m_ids 最大值 + 1） |
| **creator_trx_id** | 创建这个 ReadView 的事务自己的 ID |

可见性判断规则（对版本链上某个版本的 `trx_id`）：

| 条件 | 结论 | 原因 |
|:---|:---|:---|
| `trx_id == creator_trx_id` | **可见** | 自己改的数据自己能看到 |
| `trx_id < min_trx_id` | **可见** | 生成 ReadView 前它已提交 |
| `trx_id >= max_trx_id` | **不可见** | ReadView 生成后它才开启，必然未提交 |
| `min_trx_id <= trx_id < max_trx_id` | 看是否在 `m_ids` 中：**在 → 不可见**（仍活跃）；**不在 → 可见**（已提交） | — |

遍历过程：从当前记录开始判断，不可见就顺着 `roll_ptr` 取上一个版本继续判断，直到找到可见版本；如果整条链都不可见，说明该行对当前事务**不存在**（这是「可重复读」看不到新插入行的原因）。

### RC 与 RR 的本质区别：ReadView 的生成时机

这是 MVCC 最核心的考点，两者用的机制完全一样，**区别只在 ReadView 何时创建、复用多久**：

| 隔离级别 | ReadView 生成时机 | 效果 |
|:---|:---|:---|
| **READ COMMITTED** | **每次快照读**都生成一个新的 ReadView | 每次都能读到「此刻已提交」的最新数据，即不可重复读 |
| **REPEATABLE READ** | **事务中第一次快照读**时生成，之后整个事务复用这一个 ReadView | 整个事务看到的是同一份快照，即「可重复读」 |

时间线示例（初始 `balance = 100`）：

```
时刻   事务 A（RR）                          事务 B
t1     BEGIN;
t2     SELECT balance;  → 100（生成 ReadView，m_ids 含 B）
t3                                          BEGIN; UPDATE balance = 200; COMMIT;
t4     SELECT balance;  → 100（复用 t2 的 ReadView，B 的 trx_id 仍不可见）
t5     COMMIT;
```

把 A 的隔离级别换成 RC，`t4` 就会读到 200——因为每次都重新生成 ReadView，此时 B 已提交、不在 `m_ids` 里。

> 由此可推出两个常被追问的点：
> 1. **RR 的快照建立于「第一次快照读」那一刻，而不是 `BEGIN` 那一刻**。因此 `BEGIN` 之后、第一次 `SELECT` 之前，其他事务提交的修改，在本事务第一次读时是**能读到**的（此时创建 ReadView，对方已提交、不在 `m_ids` 中）。
> 2. **一致性读不保证一定读到旧值**：若版本链上的旧版本已被 purge 清理，只能退化为读最新版本（长事务叠加大量更新时会出现）。

### 快照读与当前读

MVCC 只作用于快照读，这是最容易被忽略的边界：

| 语句 | 类型 | 读什么 | 是否加锁 |
|:---|:---|:---|:---|
| 普通 `SELECT` | **快照读**（一致性读） | ReadView 判定的可见版本 | 不加锁 |
| `SELECT ... FOR UPDATE` / `FOR SHARE` | **当前读** | 最新已提交版本 | 加锁（RR 下为 Next-Key Lock） |
| `INSERT` / `UPDATE` / `DELETE` | **当前读** | 最新版本 | 加锁 |

关键结论：

- **快照读靠 MVCC，当前读靠锁**，两者配合才能满足隔离级别；
- 一旦语句是当前读，ReadView 就完全不起作用，读到的一定是最新数据，这也是「RR 下仍可能幻读」的根源（见下）；
- `UPDATE` 的 `WHERE` 条件扫描时，在 RC 下还有 **semi-consistent read（半一致性读）**：不匹配的行提前释放锁，减少锁冲突；RR 下不启用。

### RR 下为什么还会幻读

RR 只在**快照读**场景下靠 MVCC 屏蔽幻读，当前读仍要依赖间隙锁：

```
时刻   事务 A（RR）                                事务 B
t1     BEGIN;
t2     SELECT * FROM t WHERE id = 5;  → 空（快照建立）
t3                                                INSERT INTO t(id, c) VALUES (5, 1); COMMIT;
t4     UPDATE t SET c = 2 WHERE id = 5;  → 当前读，能看到 B 插入的行，affected rows = 1
t5     SELECT * FROM t WHERE id = 5;  → 能读到这一行（因为是自己改的）
```

`t2` 和 `t5` 两次快照读之间凭空多出一行，这就是 RR 下仍存在的幻读。因此面试标准回答是：**RR 通过「MVCC（快照读）+ Next-Key Lock（当前读）」基本解决幻读，但两者都绕开的场景仍会出现**（详见第 6 节）。

### undo log 什么时候清理：purge 与长事务

- 后台 **purge 线程**负责清理不再需要的 update undo：只要还有 ReadView 可能需要某个版本，该版本就不能删；
- 判断依据是「系统中最早的 ReadView」，因此**一个长事务会拖住整个实例的 undo 清理**；
- 后果：版本链越来越长（每次读都要遍历更多版本）、undo 表空间膨胀（`ibdata` 或独立 undo 文件持续增长且不回收）、历史版本堆积导致查询变慢。

> 实践建议：避免长事务（`information_schema.innodb_trx` 中 `trx_started` 很早的会话）；`SHOW ENGINE INNODB STATUS` 的 `History list length` 持续增长，通常就是有大事务或长事务卡住了 purge。

### 常见误区

1. **「MVCC 让读写完全无锁」**——错。只有快照读不加锁，当前读和写操作照样加锁。
2. **「RR 靠 MVCC 就解决了幻读」**——错。当前读仍需 Next-Key Lock，且如上例仍可能幻读。
3. **「版本链上存的是整行副本」**——错。undo 记录的是反向操作（或旧值），按需回放。
4. **「删除是物理删除」**——错。`DELETE` 只是置 `deleted_flag`，由 purge 真正清理。
5. **「MVCC 对所有存储引擎有效」**——错。只有 InnoDB 实现了 MVCC，MyISAM 只能表锁。

## 5. 混合使用存储引擎的问题

尽量不要在同一个事务中混合使用多种存储引擎。MySQL Server 层不管理事务，事务由存储引擎实现。

如果在事务中混合使用 InnoDB 和 MyISAM，正常提交无问题。但若需要回滚，MyISAM 表上的变更无法撤销，会导致数据不一致。

## 6. Next-Key Lock 解决幻读

MySQL 在 REPEATABLE READ 级别通过 **MVCC（快照读）+ Next-Key Lock（当前读）** 基本解决幻读，两者都绕开的边界情况见第 4 节「RR 下为什么还会幻读」。

Next-Key Lock = Record Lock（行锁）+ Gap Lock（间隙锁），锁定一个前开后闭区间。

例如一个索引包含值 10, 11, 13, 20，则锁定以下区间：
```
(-∞, 10]
(10, 11]
(11, 13]
(13, 20]
(20, +∞)
```

其他事务无法在这些区间内插入新数据，从而防止幻读。
