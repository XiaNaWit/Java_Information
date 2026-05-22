# MySQL 优化

## 1. Explain 执行计划

MySQL 通过 `EXPLAIN` 命令查看 SQL 语句的执行计划，是定位和优化低性能 SQL 最重要的方法。

> SQL 优化的通用原则（大表优化、分页优化、查询优化、结构优化等）参见 [SQL优化.md](../理论/SQL优化.md)。

### type（访问类型）

效率从高到低：`NULL > system > const > eq_ref > ref > range > index > ALL`

| type | 说明 |
|:---|:---|
| NULL | 无需访问表或索引（如 `SELECT MIN(id) FROM t`） |
| system | 表只有一行（const 的特例） |
| const | 通过主键或唯一索引一次找到 |
| eq_ref | 唯一性索引扫描，每个索引键匹配一行 |
| ref | 非唯一性索引扫描，返回匹配的所有行 |
| range | 索引范围扫描（`=, <>, >, <, BETWEEN, IN`） |
| index | 全索引扫描（扫描整个索引树） |
| ALL | 全表扫描（性能最差） |

### 常用字段

| 字段 | 说明 |
|:---|:---|
| id | SELECT 的标识符，id 越大越先执行 |
| select_type | SIMPLE / PRIMARY / SUBQUERY / UNION 等 |
| table | 查询的表 |
| partitions | 匹配的分区 |
| possible_keys | 可能使用的索引 |
| key | 实际使用的索引 |
| key_len | 使用的索引长度（字节） |
| rows | 扫描行数（估计值） |
| extra | 额外信息 |

### Extra 常见值

| 值 | 说明 |
|:---|:---|
| **Using index** | 覆盖索引，无需回表 |
| **Using where** | Server 层对存储引擎返回的数据做过滤 |
| **Using temporary** | 使用了临时表（常见于 GROUP BY / ORDER BY） |
| **Using filesort** | 无法使用索引排序，需文件排序 |
| **Using join buffer** | JOIN 时未使用索引，需连接缓冲区 |
| **Impossible where** | WHERE 条件永远为假 |

## 2. 慢查询日志

慢查询日志记录执行时间超过 `long_query_time`（默认 10 秒）的 SQL 语句。

```sql
-- 查看是否开启
SHOW VARIABLES LIKE '%slow_query_log%';

-- 临时开启
SET GLOBAL slow_query_log = 1;

-- 查看慢查询数量
SHOW GLOBAL STATUS LIKE '%Slow_queries%';
```

### 分析工具

**mysqldumpslow**（MySQL 自带）：
```bash
mysqldumpslow /var/lib/mysql/mysql-slow.log
```
统计慢 SQL 的出现次数、执行时间、等待锁时间、扫描行数等。

**mysqlsla**（需单独安装）：
```bash
mysqlsla -lt slow -sf "+select" -top 100 /data/mysql/127-slow.log > /tmp/sql_select.log
```

## 3. MySQL CPU 飙升排查

1. 用 `top` 命令确认是否为 `mysqld` 进程导致
2. 用 `SHOW PROCESSLIST`（或 `SHOW FULL PROCESSLIST`）查看当前运行的会话
3. 定位消耗高的 SQL，分析执行计划
4. 加索引、改 SQL 或调整内存参数
5. 若需立即止损，先 `KILL` 掉消耗资源的线程
6. 如果连接数激增，与应用端配合分析原因，限制连接数

## 4. select * 与 select col 区别

- `SELECT *` 查询所有字段，数据量大，无法利用覆盖索引
- 若只需要个别字段，应指定列名，可利用覆盖索引避免回表
- 例如只需 `abc` 列且有索引，写 `SELECT abc` 可以在索引树直接获取；`SELECT *` 则需要回表读取全部数据

## 5. 大表 COUNT(*) 优化

### 场景

单表数据量在千万级别，前端需要分页展示，涉及用 `COUNT(*)` 统计全表数量，耗时一般在 4-5 秒以上。

### 为什么 InnoDB 的 COUNT(*) 慢？

在不同的 MySQL 引擎中，`COUNT(*)` 的实现方式不同：

| 引擎 | 实现方式 |
|:---|:---|
| **MyISAM** | 将表的总行数存储在磁盘上，直接返回，效率极高 |
| **InnoDB** | 需要逐行扫描并累积计数，因为 MVCC 使得同一时刻不同事务看到的行数不同 |

虽然无 WHERE 条件时优化器会选择成本最小的辅助索引查询，但有时预估成本仍高于全表扫描。

### 解决方案

#### show table status（估算）

```sql
SHOW TABLE STATUS WHERE Name = '表名';
```

`TABLE_ROWS` 通过采样估算，官方文档说明误差可能达 40%~50%，不精确但速度快。

#### Redis 缓存总数

将总数缓存在 Redis 中，每次新增/删除时同步更新 Redis，查询总数直接读 Redis。

- 优点：快
- 缺点：缓存与数据库的一致性问题

#### 数据库单独表记录总数

使用触发器，在 INSERT、DELETE 时将数量更新到一张专门记录总数的表中。

- 优点：快
- 缺点：触发器可能影响写入性能

#### 做上/下一页（不分页）

前端只支持上一页/下一页，后端通过主键 ID 定位：

```sql
SELECT xx FROM table WHERE id > #{lastId} AND xxx = yyy LIMIT 1000;
```

要求主键 ID 最好是自增数字类型。

#### ES 统计

数据全量同步到 ES，查询走 ES。

- 优点：快
- 缺点：ES 与关系型数据库适用场景不同

### 方案选择优先级

1. 不做分页，改做上下页滚动
2. 返回大概总数，不需精确
3. 数据库单独表存总数
4. Redis 缓存
5. 直接 `COUNT(*)`，可能超时
6. ES

### COUNT 写法区别

| 写法 | 行为 |
|:---|:---|
| `COUNT(*)` | 统计行数，已针对 InnoDB 优化，不统计 NULL 行 |
| `COUNT(1)` | 与 COUNT(*) 几乎无区别 |
| `COUNT(col)` | 统计 `col` 列非 NULL 的行数 |
