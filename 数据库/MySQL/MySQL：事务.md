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

MVCC（Multi-Version Concurrency Control）是 InnoDB 实现 READ COMMITTED 和 REPEATABLE READ 隔离级别的关键机制。

### 隐式字段

InnoDB 聚簇索引中的每一行记录包含三个隐藏列：

| 字段 | 说明 |
|:---|:---|
| **DB_TRX_ID** | 记录最后一次修改该记录的事务 ID |
| **DB_ROLL_PTR** | 回滚指针，指向 undo log 中的上一个版本 |
| **DB_ROW_ID** | 隐藏自增 ID（表无主键时使用） |

### Undo 版本链

对数据行的每次修改都会在 undo log 中生成一个新版本，通过回滚指针串联成链。

```
[版本3(最新)] ← 回滚指针 ← [版本2] ← 回滚指针 ← [版本1(最旧)]
```

### ReadView

ReadView 是事务在某个时刻对事务系统的快照，包含：
- **m_ids**：当前活跃的读写事务 ID 列表
- **min_trx_id**：m_ids 中的最小值
- **max_trx_id**：m_ids 中的最大值 + 1

判断规则：
- `trx_id < min_trx_id` → 该版本在 ReadView 创建前已提交，**可见**
- `trx_id >= max_trx_id` → 该版本在 ReadView 创建后才生成，**不可见**
- `min_trx_id <= trx_id < max_trx_id`：
  - 若 trx_id 在 m_ids 中（该事务仍活跃）→ **不可见**
  - 若 trx_id 不在 m_ids 中（已提交）→ **可见**

不可见时沿回滚指针找下一个版本继续判断。

### 快照读与当前读

- **快照读**：普通的 SELECT 操作，不加锁，读的是 MVCC 提供的快照版本
- **当前读**：SELECT ... FOR UPDATE / LOCK IN SHARE MODE 及 INSERT/UPDATE/DELETE，读取最新数据并加锁

## 5. 混合使用存储引擎的问题

尽量不要在同一个事务中混合使用多种存储引擎。MySQL Server 层不管理事务，事务由存储引擎实现。

如果在事务中混合使用 InnoDB 和 MyISAM，正常提交无问题。但若需要回滚，MyISAM 表上的变更无法撤销，会导致数据不一致。

## 6. Next-Key Lock 解决幻读

MySQL 在 REPEATABLE READ 级别通过 **MVCC + Next-Key Lock** 解决幻读。

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
