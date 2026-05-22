# SQL Server 基础

## 概述

SQL Server 是微软开发的关系型数据库管理系统（RDBMS），主要运行在 Windows 平台（2017+ 也支持 Linux/Docker）。以其与企业生态（.NET、Azure、Active Directory）的深度集成著称。

## 核心特性

| 特性 | 说明 |
|:---|:---|
| 闭源商业软件 | 提供 Express（免费）/ Standard / Enterprise 等版本 |
| ACID 兼容 | 完整支持事务、MVCC |
| T-SQL | Transact-SQL，在 SQL 标准上扩展了编程能力（存储过程、函数、触发器） |
| 高可用 | Always On 可用性组、故障转移集群、日志传送、数据库镜像 |
| 集成服务 | SSIS（数据集成）、SSRS（报表服务）、SSAS（分析服务） |
| 安全 | 行级安全（RLS）、动态数据掩码、始终加密（Always Encrypted） |

## 与 MySQL 的主要区别

| 对比维度 | SQL Server | MySQL |
|:---|:---|:---|
| 开发商 | 微软 | Oracle |
| 平台 | Windows / Linux / Docker | Windows / Linux / macOS |
| SQL 方言 | T-SQL | MySQL SQL |
| 索引类型 | 聚集/非聚集、列存储索引、全文索引、空间索引 | B+Tree、Hash、Full-text |
| 事务隔离级别快照 | 支持 SNAPSHOT 隔离和 READ COMMITTED SNAPSHOT | 通过 MVCC 实现类似功能 |
| 窗口函数 | 完整支持（2005+） | 8.0+ 支持 |
| 序列（Sequence） | 原生支持 | 不支持（可用自增模拟） |
| 公共表表达式（CTE） | 支持（含递归 CTE） | 8.0+ 支持 |
| 触发器 | 支持 INSTEAD OF / AFTER 触发器 | 支持 BEFORE / AFTER 触发器 |

## 架构

```
SQL Server 架构：
客户端 → 关系引擎（Relational Engine）
         ├─ 命令解析器（Parser）
         ├─ 查询优化器（Optimizer）
         └─ 查询执行器（Executor）
              ↓
         存储引擎（Storage Engine）
         ├─ 访问方法（Access Methods）
         ├─ 锁管理器（Lock Manager）
         ├─ 缓冲区管理器（Buffer Manager）
         └─ 事务管理器（Transaction Manager）
              ↓
         文件系统（数据文件 .mdf / 日志文件 .ldf）
```

- **数据文件**：主文件 `.mdf`，辅助文件 `.ndf`
- **日志文件**：`.ldf`，记录事务日志
- **页面（Page）**：8KB 基本存储单元
- **区（Extent）**：8 个连续页面（64KB）

## 事务与隔离级别

SQL Server 支持六种隔离级别（比标准多两种）：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|:---|:---:|:---:|:---:|:---|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 不加锁 |
| READ COMMITTED（默认） | 不可能 | 可能 | 可能 | 共享锁 |
| READ COMMITTED SNAPSHOT | 不可能 | 不可能 | 可能 | 行版本控制 |
| REPEATABLE READ | 不可能 | 不可能 | 可能 | 保持共享锁到事务结束 |
| SNAPSHOT | 不可能 | 不可能 | 不可能 | 行版本控制 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 | 范围锁 |

> **SNAPSHOT** 和 **READ COMMITTED SNAPSHOT** 是 SQL Server 特有的基于行版本控制的隔离级别。

## 索引

| 索引类型 | 说明 |
|:---|:---|
| **聚集索引（Clustered）** | 决定数据物理存储顺序，每个表只能有一个 |
| **非聚集索引（Nonclustered）** | 独立于物理存储，可建多个，含键值和行定位器 |
| **唯一索引（Unique）** | 确保索引键值唯一 |
| **全文索引（Full-Text）** | 用于全文搜索 |
| **列存储索引（Columnstore）** | 列式存储，适合 OLAP/数据仓库 |
| **空间索引（Spatial）** | 用于地理空间数据 |
| **筛选索引（Filtered）** | 只索引满足条件的数据行 |

## 常用命令

```sql
-- 连接（SQLCMD）
sqlcmd -S server_name -U username -P password -d dbname

-- 常用系统命令
SELECT @@VERSION;                     -- 查看版本
SELECT DB_NAME();                     -- 当前数据库名
EXEC sp_help 'table_name';            -- 查看表结构
EXEC sp_who2;                         -- 查看当前连接
EXEC sp_spaceused 'table_name';       -- 查看表空间使用

-- 创建数据库
CREATE DATABASE dbname;

-- 创建登录用户
CREATE LOGIN login_name WITH PASSWORD = 'password';

-- 创建数据库用户
USE dbname;
CREATE USER username FOR LOGIN login_name;

-- 授权
GRANT SELECT, INSERT, UPDATE ON schema.table TO username;

-- 备份
BACKUP DATABASE dbname TO DISK = 'C:\backup\dbname.bak';

-- 还原
RESTORE DATABASE dbname FROM DISK = 'C:\backup\dbname.bak';
```

## 高可用方案

| 方案 | 说明 | RPO | RTO |
|:---|:---|:---:|:---:|
| **Always On 可用性组** | 一组数据库自动故障转移，支持读写分离 | 0 | 秒级 |
| **故障转移集群** | 基于 Windows 集群的实例级高可用 | 0 | 分钟级 |
| **日志传送** | 从主服务器自动发送事务日志到备用服务器 | 分钟级 | 分钟级 |
| **数据库镜像** | 已弃用，推荐使用 Always On | 0 | 秒级 |

## 优势场景

- .NET / C# / Entity Framework 生态项目
- 企业级 Windows / Azure 环境
- 需要 SSIS/SSRS/SSAS 全套 BI 解决方案
- 大企业需要 Always On 高可用方案
- 需要与 Active Directory 集成进行 Windows 身份认证
