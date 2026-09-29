# 数据库类型分类与 SQL 语法差异

本文梳理常见数据库的类型分类，并对四类主流数据库（MySQL、PostgreSQL、SQL Server、NoSQL）的**数据类型**和 **SQL 语法**做横向对比，便于迁移和排查方言差异。

## 目录

- [一、数据库类型分类](#一数据库类型分类)
  - [按数据模型分类](#按数据模型分类)
  - [按用途分类](#按用途分类)
  - [四类数据库总览](#四类数据库总览)
- [二、数据类型对比](#二数据类型对比)
  - [数值类型](#数值类型)
  - [字符串类型](#字符串类型)
  - [日期时间类型](#日期时间类型)
  - [布尔与特殊类型](#布尔与特殊类型)
- [三、SQL 语法差异](#三sql-语法差异)
  - [建表与自增主键](#建表与自增主键)
  - [分页查询](#分页查询)
  - [字符串拼接](#字符串拼接)
  - [NULL 处理](#null-处理)
  - [日期函数](#日期函数)
  - [LIMIT 与 TOP](#limit-与-top)
  - [引号与标识符](#引号与标识符)
  - [正则与模糊匹配](#正则与模糊匹配)
  - [UPSERT（插入或更新）](#upsert插入或更新)
  - [递归查询](#递归查询)
  - [窗口函数](#窗口函数)
  - [字符串聚合](#字符串聚合)
- [四、函数对照表](#四函数对照表)
- [五、迁移注意事项](#五迁移注意事项)

## 一、数据库类型分类

### 按数据模型分类

| 类型 | 数据模型 | 特点 | 代表 |
|:---|:---|:---|:---|
| **关系型（RDBMS）** | 二维表（行 + 列） | 强 Schema、ACID、支持 JOIN | MySQL、PostgreSQL、Oracle、SQL Server |
| **键值型（KV）** | Key-Value | 极高性能、结构简单 | Redis、Memcached |
| **文档型** | JSON/BSON 文档 | Schema 灵活、嵌套结构 | MongoDB、CouchDB |
| **列族型** | 列族（宽表） | 海量数据、稀疏存储 | HBase、Cassandra |
| **搜索引擎** | 倒排索引 | 全文检索、聚合分析 | Elasticsearch |
| **图数据库** | 节点 + 边 | 关系遍历高效 | Neo4j、Nebula |
| **时序型** | 时间戳 + 指标 | 高频写入、时间窗口聚合 | InfluxDB、TDengine |

### 按用途分类

| 分类 | 全称 | 特征 | 典型产品 |
|:---|:---|:---|:---|
| **OLTP** | 联机事务处理 | 高并发、短事务、行式存储、强调一致性 | MySQL、PostgreSQL、Oracle |
| **OLAP** | 联机分析处理 | 大数据量、复杂聚合、列式存储 | ClickHouse、Greenplum、Doris |
| **HTAP** | 混合事务分析处理 | 同时支持 TP 和 AP | TiDB、OceanBase、PolarDB |

### 四类数据库总览

| 对比维度 | MySQL | PostgreSQL | SQL Server | NoSQL（以 MongoDB 为例） |
|:---|:---|:---|:---|:---|
| **数据模型** | 关系型 | 对象-关系型 | 关系型 | 文档型 |
| **Schema** | 严格 | 严格（支持 JSONB 灵活扩展） | 严格 | 灵活/无 Schema |
| **方言名称** | MySQL SQL | PL/pgSQL | T-SQL | MongoDB Query Language |
| **自增主键** | `AUTO_INCREMENT` | `SERIAL` / `GENERATED` | `IDENTITY` | ObjectId（自动） |
| **分页** | `LIMIT n, m` | `LIMIT n OFFSET m` | `OFFSET n ROWS FETCH NEXT m` | `skip().limit()` |
| **默认隔离级别** | REPEATABLE READ | READ COMMITTED | READ COMMITTED | 快照隔离 |
| **事务支持** | ✅ | ✅ | ✅ | 4.0+ 多文档事务 |
| **JOIN** | ✅ | ✅ | ✅ | `$lookup`（能力有限） |
| **适用场景** | Web 应用、互联网业务 | 复杂查询、GIS、JSON | .NET 生态、企业内网 | 灵活 Schema、海量文档 |

## 二、数据类型对比

### 数值类型

| 通用概念 | MySQL | PostgreSQL | SQL Server | 说明 |
|:---|:---|:---|:---|:---|
| 微整数 | `TINYINT` | `SMALLINT` | `TINYINT` | 1 字节 |
| 小整数 | `SMALLINT` | `SMALLINT` | `SMALLINT` | 2 字节 |
| 整数 | `INT` / `INTEGER` | `INTEGER` / `INT4` | `INT` | 4 字节 |
| 长整数 | `BIGINT` | `BIGINT` / `INT8` | `BIGINT` | 8 字节 |
| 精确小数 | `DECIMAL(m,d)` | `NUMERIC(m,d)` | `DECIMAL(m,d)` | **金额必用** |
| 单精度浮点 | `FLOAT` | `REAL` / `FLOAT4` | `REAL` | 4 字节 |
| 双精度浮点 | `DOUBLE` | `DOUBLE PRECISION` / `FLOAT8` | `FLOAT` | 8 字节 |
| 自增整数 | `INT AUTO_INCREMENT` | `SERIAL` | `INT IDENTITY(1,1)` | — |
| 无符号整数 | `INT UNSIGNED` ✅ | ❌ 不支持 | ❌ 不支持 | **仅 MySQL 支持** |

> **重点差异**：**只有 MySQL 支持 `UNSIGNED`**（无符号整数），PostgreSQL 和 SQL Server 都不支持。迁移时需注意范围变化。

### 字符串类型

| 通用概念 | MySQL | PostgreSQL | SQL Server | 说明 |
|:---|:---|:---|:---|:---|
| 定长字符串 | `CHAR(n)` | `CHAR(n)` | `CHAR(n)` | 不足补空格 |
| 变长字符串 | `VARCHAR(n)` | `VARCHAR(n)` | `VARCHAR(n)` | n 为字符数 |
| 无长度限制 | `TEXT` | `TEXT` | `VARCHAR(MAX)` | SQL Server 用 MAX |
| 大文本 | `LONGTEXT` | `TEXT` | `NVARCHAR(MAX)` | — |
| 二进制 | `BLOB` | `BYTEA` | `VARBINARY(MAX)` | — |
| 枚举 | `ENUM('a','b')` ✅ | ❌（用 CHECK 约束） | ❌ | 仅 MySQL 原生支持 |
| UUID | 需 `CHAR(36)` | `UUID` 原生 | `UNIQUEIDENTIFIER` | — |

**关键差异**：

| 差异点 | 说明 |
|:---|:---|
| **字符长度语义** | MySQL 的 `VARCHAR(n)` 中 n 是**字符数**（5.0.3+）；SQL Server 的 `VARCHAR(n)` 是**字节数**，中文要用 `NVARCHAR` |
| **VARCHAR 上限** | MySQL 最多 65535 字节；PostgreSQL 无实际上限；SQL Server 最多 8000 字节（超出用 `MAX`） |
| **ENUM** | 只有 MySQL 原生支持，PostgreSQL 用 `CHECK` 约束或自定义类型模拟 |
| **Unicode** | MySQL 用 `utf8mb4` 字符集；SQL Server 需用 `NCHAR`/`NVARCHAR` 类型 |

> ⚠️ **SQL Server 最容易踩的坑**：`VARCHAR` 是单字节编码，存中文会乱码或丢失，必须用 `NVARCHAR`。且字符串字面量要加 `N` 前缀：`N'中文'`。

### 日期时间类型

| 通用概念 | MySQL | PostgreSQL | SQL Server | 说明 |
|:---|:---|:---|:---|:---|
| 日期 | `DATE` | `DATE` | `DATE` | 年月日 |
| 时间 | `TIME` | `TIME` | `TIME` | 时分秒 |
| 日期时间 | `DATETIME` | `TIMESTAMP` | `DATETIME` | 无时区 |
| 带时区 | ❌（用 `TIMESTAMP`） | `TIMESTAMPTZ` ✅ | `DATETIMEOFFSET` | PostgreSQL 支持最佳 |
| 时间戳 | `TIMESTAMP` | — | — | **受时区影响，范围到 2038** |
| 自动更新 | `ON UPDATE CURRENT_TIMESTAMP` | 需触发器 | 需触发器 | **仅 MySQL 原生支持** |

**关键差异**：

| 差异点 | 说明 |
|:---|:---|
| **`TIMESTAMP` 语义** | MySQL 的 `TIMESTAMP` 受时区影响且范围到 2038；PostgreSQL 的 `TIMESTAMP` 无时区（等价于 MySQL 的 `DATETIME`），语义**完全不同** |
| **`DATETIME` 范围** | MySQL `DATETIME` 范围 1000~9999 年；`TIMESTAMP` 仅 1970~2038 |
| **时区处理** | PostgreSQL 的 `TIMESTAMPTZ` 是最完善的；MySQL 需谨慎使用 `TIMESTAMP` |
| **获取当前时间** | MySQL `NOW()`；PostgreSQL `NOW()`；SQL Server `GETDATE()` |

### 布尔与特殊类型

| 通用概念 | MySQL | PostgreSQL | SQL Server | 说明 |
|:---|:---|:---|:---|:---|
| **布尔类型** | `TINYINT(1)` 模拟 | `BOOLEAN` 原生 ✅ | `BIT` | MySQL 无原生布尔 |
| **JSON** | `JSON`（5.7+） | `JSON` / `JSONB` ✅ | `NVARCHAR(MAX)` + 函数 | PostgreSQL 支持最佳 |
| **数组** | ❌ 不支持 | `INT[]` 原生 ✅ | ❌ | 仅 PostgreSQL |
| **范围类型** | ❌ | `INT4RANGE` 等 ✅ | ❌ | 仅 PostgreSQL |
| **几何类型** | 有限支持 | `POINT`、`POLYGON` 等 ✅ | `GEOMETRY` | PostgreSQL + PostGIS 最强 |
| **网络地址** | ❌ | `INET`、`CIDR` 原生 ✅ | ❌ | 仅 PostgreSQL |
| **枚举** | `ENUM` ✅ | 需自定义类型 | ❌ | — |

> **PostgreSQL 的类型优势**：原生支持**数组、范围、JSONB、网络地址、几何类型**，且允许**自定义类型**，这是它最大的特色之一（见 `PostgreSQL/PostgreSQL基础.md`）。

## 三、SQL 语法差异

### 建表与自增主键

```sql
-- MySQL
CREATE TABLE user (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    age  INT UNSIGNED
) ENGINE = InnoDB;

-- PostgreSQL
CREATE TABLE user (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(50),
    age  INT
);
-- 也可用标准的 GENERATED 语法（PG 10+）
-- id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY

-- SQL Server
CREATE TABLE [user] (
    id   INT IDENTITY(1,1) PRIMARY KEY,
    name NVARCHAR(50),
    age  INT
);
```

| 差异点 | 说明 |
|:---|:---|
| **自增语法** | MySQL `AUTO_INCREMENT`、PG `SERIAL`、SQL Server `IDENTITY(1,1)` |
| **获取自增 ID** | MySQL `LAST_INSERT_ID()`、PG `RETURNING id`、SQL Server `SCOPE_IDENTITY()` |
| **引擎声明** | 只有 MySQL 需 `ENGINE=InnoDB` |
| **表名引用** | SQL Server 用 `[user]`（`user` 是保留字），MySQL/PG 用反引号或双引号 |

```sql
-- 插入后获取自增 ID
-- PostgreSQL（推荐，一条语句搞定）
INSERT INTO user (name) VALUES ('张三') RETURNING id;

-- MySQL
INSERT INTO user (name) VALUES ('张三');
SELECT LAST_INSERT_ID();

-- SQL Server
INSERT INTO [user] (name) VALUES (N'张三');
SELECT SCOPE_IDENTITY();
```

### 分页查询

```sql
-- MySQL：LIMIT offset, count
SELECT * FROM user ORDER BY id LIMIT 20, 10;

-- PostgreSQL：LIMIT ... OFFSET
SELECT * FROM user ORDER BY id LIMIT 10 OFFSET 20;

-- SQL Server（2012+）：OFFSET FETCH
SELECT * FROM [user] ORDER BY id
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- SQL Server（2008 及之前）：ROW_NUMBER 模拟
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY id) AS rn FROM [user]
) t WHERE t.rn BETWEEN 21 AND 30;

-- Oracle
SELECT * FROM (
    SELECT t.*, ROWNUM rn FROM (SELECT * FROM user ORDER BY id) t
    WHERE ROWNUM <= 30
) WHERE rn > 20;
```

| 数据库 | 分页语法 | 注意 |
|:---|:---|:---|
| MySQL | `LIMIT offset, count` 或 `LIMIT count OFFSET offset` | offset 在前 |
| PostgreSQL | `LIMIT count OFFSET offset` | 不能写 `LIMIT offset, count` |
| SQL Server | `OFFSET n ROWS FETCH NEXT m ROWS ONLY` | **必须有 `ORDER BY`** |
| Oracle | `ROWNUM` / 12c+ 支持 `OFFSET FETCH` | — |

### 字符串拼接

```sql
-- MySQL：CONCAT 函数（注意：MySQL 中 || 默认是「或」！）
SELECT CONCAT(first_name, ' ', last_name) FROM user;
-- 可用 PIPES_AS_CONCAT 模式让 || 变成拼接

-- PostgreSQL：|| 运算符（也可用 CONCAT）
SELECT first_name || ' ' || last_name FROM user;
SELECT CONCAT(first_name, ' ', last_name) FROM user;

-- SQL Server：+ 运算符（也可用 CONCAT）
SELECT first_name + ' ' + last_name FROM [user];
SELECT CONCAT(first_name, ' ', last_name) FROM [user];
```

| 数据库 | 拼接方式 | 陷阱 |
|:---|:---|:---|
| MySQL | `CONCAT(a, b)` | **`\|\|` 默认是逻辑或，不是拼接** |
| PostgreSQL | `a \|\| b` | 标准 |
| SQL Server | `a + b` | **NULL 参与时整体为 NULL**（`CONCAT` 会跳过 NULL） |

> ⚠️ **重要差异**：同样的 `SELECT a || b` 在 MySQL 中是「逻辑或」，在 PostgreSQL 中是「字符串拼接」，行为完全不同！迁移时是最容易出错的地方之一。

### NULL 处理

```sql
-- 判断 NULL（各数据库一致）
SELECT * FROM user WHERE email IS NULL;

-- 替换 NULL
-- MySQL / SQL Server
SELECT IFNULL(email, '无') FROM user;          -- MySQL
SELECT ISNULL(email, '无') FROM [user];        -- SQL Server

-- PostgreSQL / 标准 SQL
SELECT COALESCE(email, '无') FROM user;

-- 通用写法（所有数据库都支持）
SELECT COALESCE(email, '无') FROM user;
```

| 函数 | MySQL | PostgreSQL | SQL Server | 标准 SQL |
|:---|:---|:---|:---|:---|
| NULL 替换 | `IFNULL` | `COALESCE` | `ISNULL` | `COALESCE` |
| 多值取首个非 NULL | `COALESCE` | `COALESCE` | `COALESCE` | `COALESCE` |
| NULL 判断 | `IS NULL` | `IS NULL` | `IS NULL` | `IS NULL` |

> **建议**：优先使用标准的 `COALESCE`，跨数据库兼容性最好。

### 日期函数

| 功能 | MySQL | PostgreSQL | SQL Server |
|:---|:---|:---|:---|
| 当前时间 | `NOW()` | `NOW()` | `GETDATE()` |
| 当前日期 | `CURDATE()` | `CURRENT_DATE` | `CAST(GETDATE() AS DATE)` |
| 格式化 | `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` | `FORMAT(d, 'yyyy-MM-dd')` |
| 字符串转日期 | `STR_TO_DATE(s, '%Y-%m-%d')` | `TO_DATE(s, 'YYYY-MM-DD')` | `CONVERT(DATE, s)` |
| 日期加 | `DATE_ADD(d, INTERVAL 1 DAY)` | `d + INTERVAL '1 day'` | `DATEADD(DAY, 1, d)` |
| 日期差 | `DATEDIFF(d1, d2)` | `d1 - d2`（返回 interval） | `DATEDIFF(DAY, d2, d1)` |
| 取年份 | `YEAR(d)` | `EXTRACT(YEAR FROM d)` | `YEAR(d)` |
| 当月最后一天 | `LAST_DAY(d)` | `(DATE_TRUNC('month', d) + INTERVAL '1 month' - INTERVAL '1 day')` | `EOMONTH(d)` |

> ⚠️ **`DATEDIFF` 参数顺序不同**：
> - MySQL：`DATEDIFF(日期1, 日期2)` → 日期1 - 日期2
> - SQL Server：`DATEDIFF(单位, 开始, 结束)` → 结束 - 开始

### LIMIT 与 TOP

```sql
-- MySQL / PostgreSQL：LIMIT
SELECT * FROM user LIMIT 10;

-- SQL Server：TOP
SELECT TOP 10 * FROM [user];

-- SQL Server 也支持 PERCENT
SELECT TOP 10 PERCENT * FROM [user];
```

> SQL Server 的 `TOP` 不能像 `LIMIT` 那样指定 offset，深分页必须用 `OFFSET FETCH`。

### 引号与标识符

| 数据库 | 字符串 | 标识符（表名/列名） |
|:---|:---|:---|
| MySQL | `'string'` | `` `backtick` `` |
| PostgreSQL | `'string'` | `"double quote"` |
| SQL Server | `'string'` | `[bracket]` 或 `"double quote"` |
| Oracle | `'string'` | `"double quote"` |

```sql
-- MySQL：表名是保留字时
SELECT * FROM `order`;

-- PostgreSQL
SELECT * FROM "order";

-- SQL Server
SELECT * FROM [order];
```

> **注意**：MySQL 中双引号默认是字符串（标准 SQL 中应是标识符），这与 PostgreSQL 不同。

### 正则与模糊匹配

```sql
-- MySQL：REGEXP（8.0+ 支持 REGEXP_LIKE）
SELECT * FROM user WHERE name REGEXP '^张';

-- PostgreSQL：~ 运算符（区分大小写）、~* 不区分
SELECT * FROM user WHERE name ~ '^张';
SELECT * FROM user WHERE name ~* 'abc';

-- SQL Server：LIKE + 通配符（无原生正则，需 CLR）
SELECT * FROM [user] WHERE name LIKE '张%';
```

| 操作 | MySQL | PostgreSQL | SQL Server |
|:---|:---|:---|:---|
| 模糊匹配 | `LIKE '%x%'` | `LIKE '%x%'` | `LIKE '%x%'` |
| 正则匹配 | `REGEXP '^x'` | `~ '^x'` | ❌ 需 CLR |
| 不区分大小写 | 默认取决于排序规则 | `~*` | 默认取决于排序规则 |

### UPSERT（插入或更新）

```sql
-- MySQL：ON DUPLICATE KEY UPDATE
INSERT INTO user (id, name, age) VALUES (1, '张三', 18)
ON DUPLICATE KEY UPDATE age = VALUES(age);
-- MySQL 8.0.20+ 推荐用别名写法
INSERT INTO user (id, name, age) VALUES (1, '张三', 18) AS new
ON DUPLICATE KEY UPDATE age = new.age;

-- PostgreSQL：ON CONFLICT
INSERT INTO user (id, name, age) VALUES (1, '张三', 18)
ON CONFLICT (id) DO UPDATE SET age = EXCLUDED.age;
-- 也可配置冲突时什么都不做
ON CONFLICT (id) DO NOTHING;

-- SQL Server：MERGE（也可用 IF EXISTS 判断）
MERGE INTO [user] AS target
USING (SELECT 1 AS id, N'张三' AS name, 18 AS age) AS source
ON target.id = source.id
WHEN MATCHED THEN UPDATE SET target.age = source.age
WHEN NOT MATCHED THEN INSERT (id, name, age) VALUES (source.id, source.name, source.age);
```

| 数据库 | 语法 | 特点 |
|:---|:---|:---|
| MySQL | `ON DUPLICATE KEY UPDATE` | 依赖唯一键/主键冲突 |
| PostgreSQL | `ON CONFLICT ... DO UPDATE/DO NOTHING` | 可指定冲突列，更明确 |
| SQL Server | `MERGE` | 功能最强但语法复杂，也有并发陷阱 |

### 递归查询

```sql
-- MySQL 8.0+ / PostgreSQL / SQL Server：标准 WITH RECURSIVE
WITH RECURSIVE org_tree AS (
    -- 锚点：根节点
    SELECT id, name, parent_id, 1 AS level
    FROM org WHERE parent_id IS NULL

    UNION ALL

    -- 递归：找子节点
    SELECT o.id, o.name, o.parent_id, t.level + 1
    FROM org o
    INNER JOIN org_tree t ON o.parent_id = t.id
)
SELECT * FROM org_tree;
```

| 数据库 | 支持情况 |
|:---|:---|
| MySQL | 8.0+ 支持 `WITH RECURSIVE`（5.7 不支持） |
| PostgreSQL | ✅ 支持 |
| SQL Server | ✅ 支持 |
| Oracle | `CONNECT BY` 专有语法 + 11g+ 支持标准写法 |

> **MySQL 5.7 的坑**：不支持 `WITH RECURSIVE`，只能用存储过程或应用层递归实现。

### 窗口函数

**支持情况**：

| 数据库 | 最低版本 | 说明 |
|:---|:---|:---|
| MySQL | **8.0+** | 5.7 完全不支持 |
| PostgreSQL | 8.4+ | 支持完善 |
| SQL Server | 2005+ | 支持完善 |
| Oracle | 8i+ | 支持完善 |

```sql
-- 通用语法（各数据库基本一致）
SELECT username, age,
    ROW_NUMBER() OVER (PARTITION BY age ORDER BY id) AS rn,
    RANK() OVER (ORDER BY age DESC) AS rk,
    SUM(amount) OVER (ORDER BY create_time) AS running_total
FROM user;
```

> **MySQL 5.7 用户的困境**：这是从 5.7 升级到 8.0 的**主要动力之一**。5.7 中只能用变量模拟，代码复杂且不可靠。

### 字符串聚合

```sql
-- MySQL：GROUP_CONCAT
SELECT age, GROUP_CONCAT(username SEPARATOR ',') FROM user GROUP BY age;

-- PostgreSQL：STRING_AGG（也支持 GROUP_CONCAT 风格的兼容函数）
SELECT age, STRING_AGG(username, ',') FROM user GROUP BY age;

-- SQL Server：STRING_AGG（2017+）或 STUFF + FOR XML（老版本）
SELECT age, STRING_AGG(username, ',') FROM [user] GROUP BY age;
```

| 数据库 | 函数 | 版本要求 |
|:---|:---|:---|
| MySQL | `GROUP_CONCAT` | 全版本 |
| PostgreSQL | `STRING_AGG` | 9.0+ |
| SQL Server | `STRING_AGG` | **2017+** |

## 四、函数对照表

| 功能 | MySQL | PostgreSQL | SQL Server |
|:---|:---|:---|:---|
| **字符串长度** | `LENGTH()` 字节 / `CHAR_LENGTH()` 字符 | `LENGTH()` 字符 / `OCTET_LENGTH()` 字节 | `LEN()` |
| **取子串** | `SUBSTRING(s, pos, len)` | `SUBSTRING(s FROM pos FOR len)` | `SUBSTRING(s, pos, len)` |
| **查找子串** | `LOCATE()` / `INSTR()` | `POSITION()` / `STRPOS()` | `CHARINDEX()` |
| **转大写** | `UPPER()` | `UPPER()` | `UPPER()` |
| **去空格** | `TRIM()` / `LTRIM()` / `RTRIM()` | `TRIM()` / `LTRIM()` / `RTRIM()` | `TRIM()` / `LTRIM()` / `RTRIM()` |
| **替换** | `REPLACE()` | `REPLACE()` | `REPLACE()` |
| **四舍五入** | `ROUND(n, d)` | `ROUND(n, d)` | `ROUND(n, d)` |
| **向上取整** | `CEIL()` / `CEILING()` | `CEIL()` | `CEILING()` |
| **取模** | `MOD(a, b)` | `MOD(a, b)` | `a % b` |
| **条件判断** | `IF(cond, t, f)` | `CASE WHEN` | `IIF(cond, t, f)` |
| **NULL 替换** | `IFNULL()` | `COALESCE()` | `ISNULL()` |
| **类型转换** | `CAST(x AS type)` / `CONVERT(x, type)` | `CAST(x AS type)` | `CAST(x AS type)` / `CONVERT(type, x)` |
| **当前时间** | `NOW()` | `NOW()` | `GETDATE()` |
| **限制行数** | `LIMIT n` | `LIMIT n` | `TOP n` |

## 五、迁移注意事项

从一种数据库迁移到另一种时，最容易踩坑的地方：

| 陷阱 | 说明 | 应对 |
|:---|:---|:---|
| **`\|\|` 语义不同** | MySQL 是逻辑或，PG 是拼接 | 统一用 `CONCAT` |
| **`VARCHAR` 长度语义** | MySQL 是字符数，SQL Server 是字节数 | SQL Server 必须用 `NVARCHAR` 存中文 |
| **`TIMESTAMP` 语义** | MySQL 受时区影响且到 2038；PG 无时区 | 注意时区转换逻辑 |
| **`NULL` 拼接** | SQL Server 的 `+` 遇 NULL 整体为 NULL | 用 `CONCAT` 或 `COALESCE` |
| **`UNSIGNED`** | 仅 MySQL 支持 | 迁移时评估取值范围是否越界 |
| **`ENUM`** | 仅 MySQL 支持 | 改用 `CHECK` 约束或关联表 |
| **`DATEDIFF` 参数顺序** | MySQL 和 SQL Server 相反 | 逐一核对 |
| **分页语法** | 各家不同，SQL Server 必须有 `ORDER BY` | 改写分页语句 |
| **`LIMIT` 位置** | MySQL/PG 在最后，SQL Server 用 `TOP` | 改写 |
| **窗口函数版本** | MySQL 需 8.0+，SQL Server 需 2005+ | 检查目标版本 |
| **保留字冲突** | `user`、`order`、`group` 等 | 用对应的标识符引号 |
| **字符串大小写** | Windows 下 MySQL/SQL Server 默认不敏感，Linux 下敏感 | 明确排序规则（collation） |

**迁移检查清单**：

- [ ] 数据类型逐一映射（特别注意数值范围、字符编码）
- [ ] 分页语句改写
- [ ] 字符串拼接方式改写
- [ ] 日期函数对照替换
- [ ] NULL 处理函数替换
- [ ] 自增主键与获取 ID 方式调整
- [ ] 窗口函数、递归查询等版本特性确认
- [ ] 保留字加引号
- [ ] 事务隔离级别差异评估
- [ ] 全量数据迁移 + 校验

## 参考文章

- [MySQL 官方文档](https://dev.mysql.com/doc/)
- [PostgreSQL 官方文档](https://www.postgresql.org/docs/)
- [SQL Server 官方文档](https://learn.microsoft.com/en-us/sql/)
- [MySQL 与 PostgreSQL 语法差异](https://www.postgresql.org/docs/current/features.html)
