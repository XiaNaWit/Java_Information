# PostgreSQL 基础

## 概述

PostgreSQL 是一个功能强大的开源对象-关系型数据库系统，拥有超过 30 年的活跃开发历史。以其高扩展性、SQL 标准兼容性和丰富的数据类型著称。

## 核心特性

| 特性 | 说明 |
|:---|:---|
| 开源免费 | PostgreSQL 许可证，类似 MIT 或 BSD，可自由使用修改 |
| ACID 兼容 | 完整支持事务、MVCC |
| 扩展性 | 支持自定义数据类型、函数、操作符、索引方法 |
| SQL 标准 | 高度兼容 SQL:2016 标准 |
| 并发控制 | 多版本并发控制（MVCC），读写互不阻塞 |
| 数据类型丰富 | 原生支持 JSON/JSONB、数组、范围类型、网络地址、几何类型等 |

## 与 MySQL 的主要区别

| 对比维度 | PostgreSQL | MySQL |
|:---|:---|:---|
| 存储引擎 | 单一存储引擎，无插件 | 插件式引擎（InnoDB/MyISAM 等） |
| SQL 标准 | 更严格遵循标准 | 有方言扩展 |
| 索引类型 | B-Tree、Hash、GiST、GIN、SP-GiST、BRIN | B+Tree、Hash、Full-text |
| JSON 支持 | JSONB（二进制，支持索引） | JSON（5.7+，8.0+ 支持 Multi-valued Index） |
| 并发模型 | 多进程架构 | 多线程架构 |
| 复制 | 流复制（物理/逻辑），同步/异步 | 主从复制、组复制 |
| 全文检索 | 内置全文检索（tsvector/tsquery） | 需 Full-text 索引 |
| 分析函数 | 原生支持（窗口函数、CTE、递归查询） | 8.0+ 支持 |
| 物化视图 | 原生支持 | 需第三方或触发器模拟 |

## 架构

```
PostgreSQL 架构（多进程）：
客户端 → Postmaster（主进程）→ 为每个连接 fork 一个后端进程
                                ↓
                         共享内存（Buffer Pool / WAL Buffer）
                                ↓
                         文件系统（数据文件 / WAL 日志）
```

- **Postmaster**：主守护进程，管理连接、启动后台进程
- **Backend Process**：每个客户端连接对应一个独立进程
- **WAL（Write-Ahead Logging）**：预写日志，用于恢复和复制
- **Checkpointer**：定期将脏页写入磁盘
- **Autovacuum**：自动清理过期数据、回收空间

## 事务与隔离级别

PostgreSQL 支持标准 SQL 的四种隔离级别：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|:---|:---:|:---:|:---:|
| READ UNCOMMITTED | 可能（实际同 READ COMMITTED） | 可能 | 可能 |
| READ COMMITTED（默认） | 不可能 | 可能 | 可能 |
| REPEATABLE READ | 不可能 | 不可能 | 可能（PG 对幻读有额外限制） |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 |

> PostgreSQL 的 REPEATABLE READ 级别不允许幻读（通过快照隔离实现），实际比 SQL 标准更严格。

## 索引

| 索引类型 | 适用场景 |
|:---|:---|
| **B-Tree** | 默认类型，适用于等值和范围查询 |
| **Hash** | 等值查询 |
| **GiST** | 全文检索、几何数据、自定义类型 |
| **GIN** | 倒排索引，适用于 JSONB、数组、全文检索 |
| **SP-GiST** | 分区树、四叉树等非平衡数据结构 |
| **BRIN** | 大数据量顺序数据，块级索引，空间占用极小 |

## 常用命令

```sql
-- 连接数据库
psql -h host -p 5432 -U username -d dbname

-- 常用 \ 命令
\l                    -- 列出所有数据库
\d                    -- 列出当前库所有表
\d table_name         -- 查看表结构
\di                   -- 列出索引
\du                   -- 列出用户
\c dbname             -- 切换数据库
\q                    -- 退出

-- 创建数据库
CREATE DATABASE dbname;

-- 创建用户
CREATE USER username WITH PASSWORD 'password';

-- 授权
GRANT ALL PRIVILEGES ON DATABASE dbname TO username;
GRANT SELECT, INSERT, UPDATE ON table_name TO username;
```

## 优势场景

- 需要复杂查询（CTE、递归查询、窗口函数）
- 地理空间数据处理（PostGIS 扩展）
- JSON/JSONB 文档存储
- 对数据完整性要求极高的金融/合规系统
- 需要自定义数据类型和函数
