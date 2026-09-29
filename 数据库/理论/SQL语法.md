# SQL 语法基础

本文介绍 SQL 的增删改查（CRUD）语法以及各类通用函数。语法以 MySQL 为主，同时标注其他数据库的差异。

## 目录

- [一、DDL：数据定义](#一ddl数据定义)
  - [建表](#建表)
  - [修改表结构](#修改表结构)
  - [删除表](#删除表)
- [二、DML：增删改](#二dml增删改)
  - [INSERT：插入](#insert插入)
  - [UPDATE：更新](#update更新)
  - [DELETE：删除](#delete删除)
- [三、DQL：查询](#三dql查询)
  - [基础查询](#基础查询)
  - [WHERE 条件](#where-条件)
  - [ORDER BY 排序](#order-by-排序)
  - [LIMIT 分页](#limit-分页)
  - [GROUP BY 分组](#group-by-分组)
  - [JOIN 连接](#join-连接)
  - [子查询](#子查询)
  - [UNION 合并](#union-合并)
- [四、通用函数](#四通用函数)
  - [字符串函数](#字符串函数)
  - [数值函数](#数值函数)
  - [日期时间函数](#日期时间函数)
  - [流程控制函数](#流程控制函数)
  - [聚合函数](#聚合函数)
  - [窗口函数](#窗口函数)
- [五、常见面试点](#五常见面试点)

## 一、DDL：数据定义

### 建表

```sql
CREATE TABLE user (
    id          BIGINT       NOT NULL AUTO_INCREMENT COMMENT '主键',
    username    VARCHAR(50)  NOT NULL                COMMENT '用户名',
    age         INT          DEFAULT 0               COMMENT '年龄',
    email       VARCHAR(100)                         COMMENT '邮箱',
    status      TINYINT      DEFAULT 1               COMMENT '状态：1正常 0禁用',
    create_time DATETIME     DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    UNIQUE KEY uk_username (username),
    KEY idx_age (age)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 COMMENT = '用户表';
```

**常用数据类型**：

| 类别 | 类型 | 说明 |
|:---|:---|:---|
| 整数 | `TINYINT` / `SMALLINT` / `INT` / `BIGINT` | 分别占 1/2/4/8 字节 |
| 小数 | `DECIMAL(m,d)` | 精确小数，**金额必用** |
| 浮点 | `FLOAT` / `DOUBLE` | 有精度问题，不适合金额 |
| 字符串 | `CHAR(n)` / `VARCHAR(n)` | 定长 / 变长 |
| 大文本 | `TEXT` / `LONGTEXT` | 不建议与主表放一起 |
| 日期 | `DATE` / `TIME` / `DATETIME` / `TIMESTAMP` | `TIMESTAMP` 受时区影响，范围到 2038 |
| 布尔 | 无原生 `BOOLEAN` | 用 `TINYINT(1)` 代替 |

### 修改表结构

```sql
-- 添加列
ALTER TABLE user ADD COLUMN phone VARCHAR(20) COMMENT '手机号';

-- 修改列类型
ALTER TABLE user MODIFY COLUMN phone VARCHAR(30);

-- 修改列名和类型
ALTER TABLE user CHANGE COLUMN phone mobile VARCHAR(30);

-- 删除列
ALTER TABLE user DROP COLUMN mobile;

-- 添加索引
ALTER TABLE user ADD INDEX idx_email (email);

-- 删除索引
ALTER TABLE user DROP INDEX idx_email;

-- 重命名表
ALTER TABLE user RENAME TO users;

-- 修改表注释
ALTER TABLE user COMMENT = '用户信息表';
```

> **注意**：`ALTER TABLE` 在数据量大时可能锁表或耗时很长（取决于版本和操作类型）。生产环境变更建议用 `pt-online-schema-change` 或 `gh-ost` 等工具。`MySQL 8.0` 部分操作已支持 `ALGORITHM=INSTANT`（秒级完成）。

### 删除表

```sql
DROP TABLE user;                    -- 删除表（不可恢复）
DROP TABLE IF EXISTS user;          -- 存在才删，避免报错
TRUNCATE TABLE user;                -- 清空数据，保留表结构
```

| 操作 | 删除内容 | 能否回滚 | 速度 | 自增 ID |
|:---|:---|:---|:---|:---|
| `DELETE` | 满足条件的行 | ✅ 可回滚 | 慢（逐行） | 不重置 |
| `TRUNCATE` | 全部数据 | ❌ 不可回滚 | 快（重建表） | **重置为 1** |
| `DROP` | 表结构 + 数据 | ❌ 不可回滚 | 快 | — |

## 二、DML：增删改

### INSERT：插入

```sql
-- 单条插入
INSERT INTO user (username, age, email) VALUES ('张三', 18, 'zs@test.com');

-- 批量插入（推荐，比多条单插快很多）
INSERT INTO user (username, age) VALUES
    ('张三', 18),
    ('李四', 20),
    ('王五', 22);

-- 插入并忽略重复（唯一键冲突时不报错）
INSERT IGNORE INTO user (username, age) VALUES ('张三', 18);

-- 唯一键冲突时更新
INSERT INTO user (username, age) VALUES ('张三', 19)
ON DUPLICATE KEY UPDATE age = VALUES(age);

-- 从查询结果插入
INSERT INTO user_backup (username, age)
SELECT username, age FROM user WHERE age > 18;
```

> **性能提示**：批量插入时，`VALUES` 列表一次 500~1000 条较合适。太大可能触发 `max_allowed_packet` 限制。

### UPDATE：更新

```sql
-- 更新指定行
UPDATE user SET age = 19 WHERE id = 1;

-- 更新多个字段
UPDATE user SET age = 19, email = 'new@test.com' WHERE id = 1;

-- 基于原值更新
UPDATE user SET age = age + 1 WHERE id = 1;

-- 关联更新（MySQL 多表更新）
UPDATE user u
JOIN order o ON u.id = o.user_id
SET u.status = 2
WHERE o.amount > 1000;
```

> ⚠️ **务必带 WHERE 条件**！忘记 `WHERE` 会更新全表。建议先用 `SELECT` 验证条件，或开启 `sql_safe_updates`。

### DELETE：删除

```sql
-- 删除指定行
DELETE FROM user WHERE id = 1;

-- 批量删除
DELETE FROM user WHERE age < 18;

-- 分批删除大量数据（避免大事务）
DELETE FROM user WHERE create_time < '2020-01-01' LIMIT 1000;
-- 循环执行，直到影响行数为 0

-- 关联删除
DELETE u FROM user u
JOIN order o ON u.id = o.user_id
WHERE o.amount = 0;
```

> **生产建议**：大批量删除不要一条 SQL 删几百万行——会产生大事务、长时间持锁、主从延迟。应分批（`LIMIT`）+ 循环 + 每次短暂 sleep。

## 三、DQL：查询

### 基础查询

```sql
-- 查询所有列（不推荐，明确列出字段更好）
SELECT * FROM user;

-- 指定列
SELECT id, username, age FROM user;

-- 别名
SELECT username AS name, age AS user_age FROM user;

-- 去重
SELECT DISTINCT age FROM user;

-- 计算列
SELECT username, age * 2 AS double_age FROM user;
```

### WHERE 条件

```sql
-- 比较运算
SELECT * FROM user WHERE age > 18;
SELECT * FROM user WHERE age >= 18 AND age <= 30;
SELECT * FROM user WHERE age BETWEEN 18 AND 30;   -- 等价于上面

-- 逻辑运算
SELECT * FROM user WHERE age > 18 AND status = 1;
SELECT * FROM user WHERE age < 18 OR age > 60;
SELECT * FROM user WHERE NOT status = 1;

-- 模糊查询
SELECT * FROM user WHERE username LIKE '张%';      -- 以张开头
SELECT * FROM user WHERE username LIKE '%张%';     -- 包含张（索引失效）
SELECT * FROM user WHERE username LIKE '_三';      -- _ 匹配单个字符

-- 范围查询
SELECT * FROM user WHERE id IN (1, 2, 3);
SELECT * FROM user WHERE id NOT IN (1, 2, 3);

-- NULL 判断（不能用 = NULL！）
SELECT * FROM user WHERE email IS NULL;
SELECT * FROM user WHERE email IS NOT NULL;

-- EXISTS
SELECT * FROM user u WHERE EXISTS (
    SELECT 1 FROM order o WHERE o.user_id = u.id
);
```

> ⚠️ **易错点**：`= NULL` 永远为 false（NULL 表示未知），必须用 `IS NULL`。

**运算符优先级**：`NOT` > `AND` > `OR`。建议用括号明确优先级：

```sql
-- 有歧义
WHERE a = 1 OR b = 2 AND c = 3     -- 实际是 a=1 OR (b=2 AND c=3)

-- 明确
WHERE (a = 1 OR b = 2) AND c = 3
```

### ORDER BY 排序

```sql
-- 升序（默认）
SELECT * FROM user ORDER BY age ASC;

-- 降序
SELECT * FROM user ORDER BY age DESC;

-- 多字段排序（先按 age 降序，age 相同时按 id 升序）
SELECT * FROM user ORDER BY age DESC, id ASC;

-- 按别名排序
SELECT username, age * 2 AS d FROM user ORDER BY d DESC;

-- NULL 值排序
SELECT * FROM user ORDER BY email IS NULL, email;   -- NULL 排最后
```

### LIMIT 分页

```sql
-- 取前 10 条
SELECT * FROM user LIMIT 10;

-- 跳过 20 条，取 10 条（第 3 页，每页 10 条）
SELECT * FROM user LIMIT 20, 10;
-- 等价于
SELECT * FROM user LIMIT 10 OFFSET 20;
```

> ⚠️ **深分页性能问题**：`LIMIT 1000000, 10` 会先扫描并丢弃前 100 万行，极慢。优化方案见下方「面试点」。

### GROUP BY 分组

```sql
-- 按年龄分组统计人数
SELECT age, COUNT(*) AS cnt
FROM user
GROUP BY age;

-- 分组后筛选（HAVING 作用于分组结果）
SELECT age, COUNT(*) AS cnt
FROM user
GROUP BY age
HAVING cnt > 5;

-- 多字段分组
SELECT age, status, COUNT(*)
FROM user
GROUP BY age, status;
```

**WHERE 与 HAVING 的区别**：

| 对比项 | WHERE | HAVING |
|:---|:---|:---|
| 执行时机 | 分组**前**过滤 | 分组**后**过滤 |
| 能否用聚合函数 | ❌ 不能 | ✅ 可以 |
| 示例 | `WHERE age > 18` | `HAVING COUNT(*) > 5` |

**执行顺序**（重要）：

```text
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

### JOIN 连接

```sql
-- 准备两张表做示例
-- user: id, username
-- order: id, user_id, amount
```

| JOIN 类型 | 说明 | 结果 |
|:---|:---|:---|
| `INNER JOIN` | 只返回两表**都匹配**的行 | 交集 |
| `LEFT JOIN` | 返回**左表全部** + 右表匹配（无匹配为 NULL） | 左表全量 |
| `RIGHT JOIN` | 返回**右表全部** + 左表匹配 | 右表全量 |
| `FULL OUTER JOIN` | 返回两表全部（MySQL 不支持，用 UNION 模拟） | 并集 |
| `CROSS JOIN` | 笛卡尔积 | 全组合 |

```sql
-- 内连接：只查有订单的用户
SELECT u.username, o.amount
FROM user u
INNER JOIN order o ON u.id = o.user_id;

-- 左连接：查所有用户（含无订单的，订单字段为 NULL）
SELECT u.username, o.amount
FROM user u
LEFT JOIN order o ON u.id = o.user_id;

-- 查「没有订单的用户」（左连接 + 右表为 NULL）
SELECT u.username
FROM user u
LEFT JOIN order o ON u.id = o.user_id
WHERE o.id IS NULL;
```

> **提示**：`LEFT JOIN ... WHERE 右表字段 IS NULL` 是查找「左表有而右表没有」的经典写法。

### 子查询

```sql
-- 标量子查询（返回单个值）
SELECT * FROM user WHERE age > (SELECT AVG(age) FROM user);

-- 列子查询（IN）
SELECT * FROM user WHERE id IN (
    SELECT user_id FROM order WHERE amount > 1000
);

-- 派生表（FROM 子句中的子查询）
SELECT t.age, t.cnt
FROM (
    SELECT age, COUNT(*) AS cnt FROM user GROUP BY age
) t
WHERE t.cnt > 5;

-- EXISTS 相关子查询
SELECT * FROM user u
WHERE EXISTS (SELECT 1 FROM order o WHERE o.user_id = u.id);
```

**IN 与 EXISTS 的选择**：

- 子查询结果集**小** → 用 `IN`
- 外层结果集**小** → 用 `EXISTS`
- 现代优化器通常会自动优化，差异不大

### UNION 合并

```sql
-- UNION：合并并去重
SELECT username FROM user
UNION
SELECT name FROM customer;

-- UNION ALL：合并但不去重（更快，推荐）
SELECT username FROM user
UNION ALL
SELECT name FROM customer;
```

> `UNION` 会做去重（隐含排序），性能较差。若确定无重复或允许重复，用 `UNION ALL`。

## 四、通用函数

### 字符串函数

| 函数 | 作用 | 示例 | 结果 |
|:---|:---|:---|:---|
| `CONCAT(s1, s2, ...)` | 拼接 | `CONCAT('a', 'b')` | `ab` |
| `CONCAT_WS(sep, s1, ...)` | 带分隔符拼接 | `CONCAT_WS('-', 'a', 'b')` | `a-b` |
| `LENGTH(s)` | 字节长度 | `LENGTH('中文')` | `6` |
| `CHAR_LENGTH(s)` | 字符长度 | `CHAR_LENGTH('中文')` | `2` |
| `UPPER(s)` / `LOWER(s)` | 大小写转换 | `UPPER('ab')` | `AB` |
| `TRIM(s)` | 去首尾空格 | `TRIM(' a ')` | `a` |
| `LTRIM(s)` / `RTRIM(s)` | 去左/右空格 | — | — |
| `SUBSTRING(s, pos, len)` | 截取子串 | `SUBSTRING('abcdef', 2, 3)` | `bcd` |
| `LEFT(s, n)` / `RIGHT(s, n)` | 取左/右 n 个字符 | `LEFT('abc', 2)` | `ab` |
| `REPLACE(s, from, to)` | 替换 | `REPLACE('a-b', '-', '+')` | `a+b` |
| `LOCATE(sub, s)` | 子串位置（从 1 开始） | `LOCATE('b', 'abc')` | `2` |
| `INSTR(s, sub)` | 同上（参数相反） | `INSTR('abc', 'b')` | `2` |
| `LPAD(s, len, pad)` | 左填充 | `LPAD('5', 3, '0')` | `005` |
| `RPAD(s, len, pad)` | 右填充 | `RPAD('5', 3, '0')` | `500` |
| `REVERSE(s)` | 反转 | `REVERSE('abc')` | `cba` |
| `REPEAT(s, n)` | 重复 n 次 | `REPEAT('a', 3)` | `aaa` |
| `FORMAT(n, d)` | 格式化数字 | `FORMAT(1234.567, 2)` | `1,234.57` |

```sql
-- 常用组合：截取手机号后 4 位
SELECT RIGHT(phone, 4) FROM user;

-- 脱敏：中间 4 位换成 ****
SELECT CONCAT(LEFT(phone, 3), '****', RIGHT(phone, 4)) FROM user;
```

### 数值函数

| 函数 | 作用 | 示例 | 结果 |
|:---|:---|:---|:---|
| `ABS(n)` | 绝对值 | `ABS(-5)` | `5` |
| `CEIL(n)` / `CEILING(n)` | 向上取整 | `CEIL(1.1)` | `2` |
| `FLOOR(n)` | 向下取整 | `FLOOR(1.9)` | `1` |
| `ROUND(n, d)` | 四舍五入 | `ROUND(1.567, 2)` | `1.57` |
| `TRUNCATE(n, d)` | 截断（不四舍五入） | `TRUNCATE(1.567, 2)` | `1.56` |
| `MOD(a, b)` | 取余 | `MOD(10, 3)` | `1` |
| `POW(a, b)` / `POWER(a, b)` | 幂运算 | `POW(2, 3)` | `8` |
| `SQRT(n)` | 平方根 | `SQRT(16)` | `4` |
| `RAND()` | 随机数 [0,1) | `RAND()` | `0.73...` |
| `SIGN(n)` | 符号 | `SIGN(-5)` | `-1` |

```sql
-- 随机取 10 条
SELECT * FROM user ORDER BY RAND() LIMIT 10;   -- ⚠️ 大表慎用，性能差

-- 金额保留两位小数
SELECT ROUND(amount, 2) FROM `order`;
```

### 日期时间函数

| 函数 | 作用 | 示例 | 结果 |
|:---|:---|:---|:---|
| `NOW()` | 当前日期时间 | `NOW()` | `2026-09-29 10:00:00` |
| `CURDATE()` / `CURTIME()` | 当前日期 / 时间 | `CURDATE()` | `2026-09-29` |
| `DATE(d)` | 取日期部分 | `DATE('2026-09-29 10:00')` | `2026-09-29` |
| `YEAR(d)` / `MONTH(d)` / `DAY(d)` | 取年 / 月 / 日 | `YEAR(NOW())` | `2026` |
| `HOUR(d)` / `MINUTE(d)` / `SECOND(d)` | 取时 / 分 / 秒 | — | — |
| `DAYOFWEEK(d)` | 星期几（1=周日） | — | — |
| `WEEKDAY(d)` | 星期几（0=周一） | — | — |
| `DATEDIFF(d1, d2)` | 日期差（天数） | `DATEDIFF('2026-10-01', '2026-09-29')` | `2` |
| `TIMEDIFF(t1, t2)` | 时间差 | — | — |
| `DATE_ADD(d, INTERVAL n unit)` | 日期加 | `DATE_ADD(NOW(), INTERVAL 1 DAY)` | 明天 |
| `DATE_SUB(d, INTERVAL n unit)` | 日期减 | `DATE_SUB(NOW(), INTERVAL 7 DAY)` | 7 天前 |
| `DATE_FORMAT(d, fmt)` | 格式化 | `DATE_FORMAT(NOW(), '%Y-%m-%d')` | `2026-09-29` |
| `STR_TO_DATE(s, fmt)` | 字符串转日期 | `STR_TO_DATE('2026-09-29', '%Y-%m-%d')` | 日期 |
| `UNIX_TIMESTAMP(d)` | 转时间戳 | — | — |
| `FROM_UNIXTIME(ts)` | 时间戳转日期 | — | — |
| `LAST_DAY(d)` | 当月最后一天 | `LAST_DAY('2026-02-15')` | `2026-02-28` |

**常用格式符**：

| 格式符 | 含义 | 示例 |
|:---|:---|:---|
| `%Y` | 4 位年 | 2026 |
| `%y` | 2 位年 | 26 |
| `%m` | 月（01-12） | 09 |
| `%d` | 日（01-31） | 29 |
| `%H` | 小时（00-23） | 14 |
| `%i` | 分钟 | 30 |
| `%s` | 秒 | 45 |
| `%W` | 星期名 | Monday |

```sql
-- 查询今天的记录
SELECT * FROM `order` WHERE DATE(create_time) = CURDATE();
-- ⚠️ 上面的 DATE(create_time) 会导致索引失效！

-- ✅ 优化：用范围查询，可走索引
SELECT * FROM `order`
WHERE create_time >= CURDATE()
  AND create_time < DATE_ADD(CURDATE(), INTERVAL 1 DAY);

-- 按天分组统计
SELECT DATE_FORMAT(create_time, '%Y-%m-%d') AS day, COUNT(*)
FROM `order`
GROUP BY day;
```

> ⚠️ **性能坑**：在索引列上使用函数（如 `DATE(create_time)`）会导致**索引失效**，应改为范围查询。

### 流程控制函数

| 函数 | 作用 |
|:---|:---|
| `IF(cond, t, f)` | 三元运算 |
| `IFNULL(v, default)` | 为 NULL 时取默认值 |
| `NULLIF(a, b)` | a = b 时返回 NULL，否则返回 a |
| `COALESCE(v1, v2, ...)` | 返回第一个非 NULL 值 |
| `CASE WHEN` | 多条件分支 |

```sql
-- IF：简单判断
SELECT username, IF(age >= 18, '成年', '未成年') AS type FROM user;

-- IFNULL：处理 NULL
SELECT username, IFNULL(email, '未填写') AS email FROM user;

-- COALESCE：取第一个非 NULL
SELECT COALESCE(phone, email, '无联系方式') FROM user;

-- CASE WHEN：多分支（行转列的经典用法）
SELECT
    SUM(CASE WHEN age < 18 THEN 1 ELSE 0 END) AS 未成年,
    SUM(CASE WHEN age >= 18 AND age < 60 THEN 1 ELSE 0 END) AS 成年,
    SUM(CASE WHEN age >= 60 THEN 1 ELSE 0 END) AS 老年
FROM user;

-- CASE 的完整写法
SELECT username,
    CASE status
        WHEN 1 THEN '正常'
        WHEN 0 THEN '禁用'
        ELSE '未知'
    END AS status_desc
FROM user;
```

### 聚合函数

| 函数 | 作用 | 说明 |
|:---|:---|:---|
| `COUNT(*)` | 统计行数 | **不忽略 NULL**，InnoDB 有优化 |
| `COUNT(col)` | 统计非 NULL 值 | **忽略 NULL** |
| `COUNT(1)` | 同 `COUNT(*)` | 性能基本一致 |
| `SUM(col)` | 求和 | 忽略 NULL |
| `AVG(col)` | 平均值 | 忽略 NULL（注意：分母是非 NULL 行数） |
| `MAX(col)` / `MIN(col)` | 最大 / 最小 | 忽略 NULL |
| `GROUP_CONCAT(col)` | 分组内拼接 | 默认逗号分隔 |

```sql
-- COUNT(*) vs COUNT(col) 的区别
SELECT COUNT(*) FROM user;          -- 统计总行数
SELECT COUNT(email) FROM user;      -- 只统计 email 非 NULL 的行
```

> **经典坑**：`AVG(col)` 忽略 NULL，如果某行 col 是 NULL，它不计入分母。若想让 NULL 算作 0，需用 `AVG(IFNULL(col, 0))`。

```sql
-- GROUP_CONCAT：把分组内的值拼成字符串
SELECT age, GROUP_CONCAT(username) AS names
FROM user
GROUP BY age;
-- 结果：18 | 张三,李四,王五

-- 可自定义分隔符和排序
SELECT GROUP_CONCAT(username ORDER BY id DESC SEPARATOR ' | ')
FROM user;
```

### 窗口函数

窗口函数（MySQL 8.0+、PostgreSQL、SQL Server 2012+ 支持）在**不合并行**的前提下做聚合计算。

**语法**：`函数() OVER (PARTITION BY ... ORDER BY ...)`

| 函数 | 作用 |
|:---|:---|
| `ROW_NUMBER()` | 连续序号（1,2,3,4） |
| `RANK()` | 排名，并列时跳号（1,2,2,4） |
| `DENSE_RANK()` | 排名，并列时不跳号（1,2,2,3） |
| `SUM() OVER()` | 累计求和 |
| `LAG(col, n)` / `LEAD(col, n)` | 取前 / 后 n 行 |
| `FIRST_VALUE()` / `LAST_VALUE()` | 分组内首 / 末值 |

```sql
-- 1. 每个年龄组内按年龄排名
SELECT username, age,
    ROW_NUMBER() OVER (PARTITION BY age ORDER BY id) AS rn
FROM user;

-- 2. 分组取 Top N（经典场景）
SELECT * FROM (
    SELECT username, age, amount,
        ROW_NUMBER() OVER (PARTITION BY age ORDER BY amount DESC) AS rn
    FROM user
) t
WHERE t.rn <= 3;      -- 每个年龄取金额前 3

-- 3. 三种排名函数的区别（假设分数为 100, 100, 90）
SELECT score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS rn,      -- 1, 2, 3
    RANK()       OVER (ORDER BY score DESC) AS rk,      -- 1, 1, 3
    DENSE_RANK() OVER (ORDER BY score DESC) AS drk      -- 1, 1, 2
FROM scores;

-- 4. 累计求和
SELECT day, amount,
    SUM(amount) OVER (ORDER BY day) AS running_total
FROM sales;

-- 5. 环比：与上一行比较
SELECT day, amount,
    LAG(amount, 1) OVER (ORDER BY day) AS prev_amount,
    amount - LAG(amount, 1) OVER (ORDER BY day) AS diff
FROM sales;
```

**窗口函数 vs GROUP BY**：

| 对比项 | GROUP BY | 窗口函数 |
|:---|:---|:---|
| 结果行数 | 分组后**减少** | **保持原行数** |
| 能否保留明细 | ❌ 不行 | ✅ 可以 |
| 典型用途 | 汇总统计 | 排名、累计、环比 |

## 五、常见面试点

**1. `COUNT(*)`、`COUNT(1)`、`COUNT(col)` 的区别？**

- `COUNT(*)` 和 `COUNT(1)`：统计所有行，**不忽略 NULL**，InnoDB 对二者有同样优化，性能无差异。
- `COUNT(col)`：只统计该列**非 NULL** 的行数，逻辑不同。
- 结论：需要统计行数时用 `COUNT(*)`，不要用 `COUNT(col)` 除非确实要排除 NULL。

**2. 深分页为什么慢？怎么优化？**

```sql
-- ❌ 慢：扫描 100 万行后丢弃，再取 10 行
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;

-- ✅ 方案一：延迟关联（先查主键，再回表）
SELECT u.* FROM user u
INNER JOIN (
    SELECT id FROM user ORDER BY id LIMIT 1000000, 10
) t ON u.id = t.id;

-- ✅ 方案二：游标分页（记录上一页最后一条的 id）
SELECT * FROM user WHERE id > 1000000 ORDER BY id LIMIT 10;
```

**3. `WHERE` 和 `HAVING` 的区别？**

- `WHERE` 在分组**前**过滤，不能用聚合函数。
- `HAVING` 在分组**后**过滤，可以用聚合函数。

**4. `DELETE`、`TRUNCATE`、`DROP` 的区别？**

| | DELETE | TRUNCATE | DROP |
|:---|:---|:---|:---|
| 类型 | DML | DDL | DDL |
| 删除内容 | 满足条件的行 | 全部数据 | 表结构 + 数据 |
| 可回滚 | ✅ | ❌ | ❌ |
| 速度 | 慢 | 快 | 快 |
| 自增重置 | 否 | **是** | — |

**5. 为什么索引列上使用函数会导致索引失效？**

因为 B+ 树索引按**列的原始值**排序。写成 `WHERE DATE(create_time) = '2026-09-29'` 时，数据库需要**对每一行先计算函数值**再比较，无法利用索引的有序性，只能全表扫描。

**解决**：改写为范围查询 `WHERE create_time >= '2026-09-29' AND create_time < '2026-09-30'`。

**6. `NULL` 的注意事项有哪些？**

- `= NULL` 永远为 false，必须用 `IS NULL` / `IS NOT NULL`
- 聚合函数 `SUM` / `AVG` / `MAX` / `MIN` 都**忽略 NULL**
- `COUNT(*)` 不忽略，`COUNT(col)` 忽略
- `NULL + 任何值 = NULL`
- `NULL` 参与排序时，MySQL 中 `ASC` 时排最前，`DESC` 时排最后（各数据库行为不同）
- 唯一索引允许多个 NULL（因为 NULL != NULL）

**7. `UNION` 和 `UNION ALL` 的区别？**

- `UNION` 去重（隐含排序），性能较差。
- `UNION ALL` 不去重，性能更好。
- 确定无重复或允许重复时，优先用 `UNION ALL`。

**8. `IN` 和 `EXISTS` 哪个快？**

- 子查询结果集**小** → 用 `IN`
- 外层表**小**、子查询表**大** → 用 `EXISTS`
- 现代优化器多数情况能自动转换，差异不明显。真正的优化点在于**是否走了索引**。

## 参考文章

- [MySQL 官方文档：函数](https://dev.mysql.com/doc/refman/8.0/en/functions.html)
- [MySQL 官方文档：SELECT 语法](https://dev.mysql.com/doc/refman/8.0/en/select.html)
- [PostgreSQL 官方文档：函数](https://www.postgresql.org/docs/current/functions.html)
