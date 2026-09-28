# JDBC

## 目录

- [什么是 JDBC](#什么是-jdbc)
- [JDBC 的核心 API](#jdbc-的核心-api)
- [JDBC 执行六步骤](#jdbc-执行六步骤)
  - [完整代码示例](#完整代码示例)
  - [步骤详解](#步骤详解)
- [Statement 与 PreparedStatement](#statement-与-preparedstatement)
- [ResultSet 结果集](#resultset-结果集)
- [事务管理](#事务管理)
- [批处理](#批处理)
- [连接池](#连接池)
- [常见问题](#常见问题)
- [高频深挖面试题](#高频深挖面试题)
  - [1. PreparedStatement 防注入，靠的真是「预编译」吗？](#1-preparedstatement-防注入靠的真是预编译吗)
  - [2. 为什么 JDBC 4.0 之后可以不写 `Class.forName`？](#2-为什么-jdbc-40-之后可以不写-classforname)
  - [3. 连接池的 `close()` 为什么不会真的关连接？](#3-连接池的-close-为什么不会真的关连接)
  - [4. `maxLifetime` 为什么必须小于数据库的 `wait_timeout`？](#4-maxlifetime-为什么必须小于数据库的-wait_timeout)
  - [5. MySQL 的批处理为什么开了却没效果？](#5-mysql-的批处理为什么开了却没效果)
  - [6. 归还连接前为什么要恢复 `autoCommit`？](#6-归还连接前为什么要恢复-autocommit)
  - [7. 为什么说 JDBC 的异常设计不合理？](#7-为什么说-jdbc-的异常设计不合理)
  - [8. MyBatis 到底在 JDBC 之上做了什么？](#8-mybatis-到底在-jdbc-之上做了什么)
  - [9. `SELECT *` 到底慢在哪？](#9-select--到底慢在哪)
  - [10. 如果让你设计一个连接池，关键难点在哪？](#10-如果让你设计一个连接池关键难点在哪)

## 什么是 JDBC

JDBC（Java Database Connectivity）是 Java 提供的**用于执行 SQL 语句的 Java API**，位于 `java.sql` 和 `javax.sql` 包中。

它是一套**接口规范**，由 Java 官方定义，各数据库厂商提供具体实现（即数据库驱动）。开发者面向接口编程，换数据库只需更换驱动，代码基本不用改。

- **面向接口**：JDBC 只定义标准，具体实现由驱动完成
- **数据库无关**：同一套代码可访问 MySQL、Oracle、PostgreSQL 等
- **桥梁作用**：Java 程序与数据库之间的通信标准

```
Java 应用
   ↓  调用 java.sql.* 接口
JDBC API（规范）
   ↓  由驱动实现
JDBC Driver（如 mysql-connector-java）
   ↓  网络协议
数据库（MySQL / Oracle / PostgreSQL）
```

## JDBC 的核心 API

| 接口/类 | 作用 |
|:---|:---|
| `DriverManager` | 驱动管理器，负责注册驱动、获取 Connection |
| `Driver` | 驱动接口，各数据库厂商实现 |
| `Connection` | 数据库连接对象，代表一个会话 |
| `Statement` | 执行静态 SQL 语句 |
| `PreparedStatement` | 执行预编译 SQL，支持参数占位符 |
| `CallableStatement` | 执行存储过程 |
| `ResultSet` | 封装查询结果集，可遍历读取 |
| `SQLException` | JDBC 操作抛出的检查型异常 |
| `DataSource` | 数据源接口，连接池的基础 |

## JDBC 执行六步骤

1. **注册驱动**：加载数据库驱动类
2. **获取 Connection 连接**：通过 `DriverManager` 建立连接
3. **执行预编译**：创建 `Statement` / `PreparedStatement`
4. **执行 SQL**：调用 `executeQuery` / `executeUpdate`
5. **封装结果集**：遍历 `ResultSet` 映射为 Java 对象
6. **释放资源**：关闭 `ResultSet`、`Statement`、`Connection`

### 完整代码示例

```java
public void findStudent() {
    Connection conn = null;
    Statement stmt = null;
    ResultSet rs = null;

    try {
        // 1. 注册MySQL驱动
        Class.forName("com.mysql.cj.jdbc.Driver");
        // 连接数据库的基本信息
        String url = "jdbc:mysql://localhost:3306/db";
        String username = "root";
        String password = "password";
        // 2. 创建连接对象
        conn = DriverManager.getConnection(url, username, password);
        // 保存查询结果
        List<Student> stuList = new ArrayList<>();
        // 3. 创建statement，执行SQL
        stmt = conn.createStatement();
        // 4. 执行查询
        rs = stmt.executeQuery("select * from student");
        // 5. 取出结果封装为对象
        while (rs.next()) {
            Student student = new Student();
            student.setAge(rs.getInt("age"));
            student.setName(rs.getString("name"));
            stuList.add(student);
        }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        // 6. 关闭资源（顺序与打开相反）
        try {
            if (rs != null) {
                rs.close();
            }
            if (stmt != null) {
                stmt.close();
            }
            if (conn != null) {
                conn.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

### 步骤详解

#### 1. 注册驱动

```java
Class.forName("com.mysql.cj.jdbc.Driver");
```

本质是利用**反射**加载驱动类，触发其静态代码块，向 `DriverManager` 注册自己：

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    static {
        try {
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

> JDBC 4.0 之后可省略这一步，驱动会通过 SPI 机制（`META-INF/services/java.sql.Driver`）自动加载。

#### 2. 获取连接

```java
conn = DriverManager.getConnection(url, username, password);
```

URL 格式：`jdbc:子协议://主机:端口/数据库名?参数`

```text
jdbc:mysql://localhost:3306/db?useSSL=false&serverTimezone=Asia/Shanghai
```

#### 3. 创建 Statement

```java
stmt = conn.createStatement();
```

#### 4. 执行 SQL

| 方法 | 用途 | 返回值 |
|:---|:---|:---|
| `executeQuery(sql)` | 执行 SELECT | `ResultSet` |
| `executeUpdate(sql)` | 执行 INSERT/UPDATE/DELETE | 影响行数（int） |
| `execute(sql)` | 执行任意 SQL | boolean |

#### 5. 处理结果集

`ResultSet` 维护一个游标，初始位于第一行之前，需调用 `next()` 移动到下一行。

```java
while (rs.next()) {
    int age = rs.getInt("age");          // 按列名获取
    String name = rs.getString("name");  // 也可按索引 getString(1)
}
```

#### 6. 释放资源

**按打开顺序的逆序关闭**：`ResultSet` → `Statement` → `Connection`。推荐用 try-with-resources 自动关闭：

```java
try (Connection conn = DriverManager.getConnection(url, user, pwd);
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
        // 处理结果
    }
}
```

## Statement 与 PreparedStatement

| 对比项 | Statement | PreparedStatement |
|:---|:---|:---|
| SQL 形式 | 字符串拼接，静态 SQL | 带 `?` 占位符，预编译 SQL |
| SQL 注入 | **存在风险** | 天然防注入 |
| 执行效率 | 每次都要编译 | 预编译一次可重复执行 |
| 可读性 | 拼接混乱 | 参数与结构分离，清晰 |
| 适用场景 | 极少使用 | **推荐使用** |

```java
// Statement：拼接方式，存在注入风险
String sql = "select * from user where name = '" + name + "'";

// PreparedStatement：占位符方式，安全
String sql = "select * from user where name = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setString(1, name);
ResultSet rs = ps.executeQuery();
```

**为什么 PreparedStatement 能防注入？**

SQL 结构在预编译阶段就已确定，参数值只作为数据处理，不会被解析为 SQL 语法的一部分。即使传入 `' or '1'='1`，也只会被当作普通字符串。

## ResultSet 结果集

| 方法 | 说明 |
|:---|:---|
| `next()` | 移动到下一行，无更多行为返回 false |
| `getInt(String/ int)` | 获取 int 类型字段 |
| `getString(...)` | 获取 String 类型字段 |
| `getObject(...)` | 获取任意类型字段 |
| `wasNull()` | 判断上次读取的值是否为 SQL NULL |

**关于 NULL 的坑**：`getInt()` 遇到 NULL 会返回 0，无法区分「值就是 0」和「值为 NULL」。此时应改用 `getObject()` 或配合 `wasNull()`：

```java
int age = rs.getInt("age");
if (rs.wasNull()) {
    // 确实是 NULL，而非 0
}
```

## 事务管理

JDBC 默认**自动提交**（每执行一条 SQL 就提交一次）。要使用事务需手动控制：

```java
Connection conn = null;
try {
    conn = DriverManager.getConnection(url, user, pwd);
    conn.setAutoCommit(false);            // 1. 关闭自动提交

    // 2. 执行多条 SQL
    try (PreparedStatement ps1 = conn.prepareStatement("update account set balance = balance - ? where id = ?")) {
        ps1.setBigDecimal(1, amount);
        ps1.setInt(2, fromId);
        ps1.executeUpdate();
    }
    try (PreparedStatement ps2 = conn.prepareStatement("update account set balance = balance + ? where id = ?")) {
        ps2.setBigDecimal(1, amount);
        ps2.setInt(2, toId);
        ps2.executeUpdate();
    }

    conn.commit();                        // 3. 全部成功，提交
} catch (Exception e) {
    if (conn != null) {
        conn.rollback();                  // 4. 出现异常，回滚
    }
    throw e;
} finally {
    if (conn != null) {
        conn.setAutoCommit(true);         // 5. 恢复自动提交（连接池复用前提）
    }
}
```

**要点**：
- `setAutoCommit(false)` 开启事务
- `commit()` 提交、`rollback()` 回滚
- 归还连接到连接池前**必须恢复 `autoCommit=true`**，否则会污染后续使用
- 可设置保存点 `conn.setSavepoint()` 实现部分回滚

## 批处理

批量插入时，逐条执行效率极低。用 `addBatch()` + `executeBatch()` 可大幅提升性能。

```java
String sql = "insert into student(name, age) values(?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql)) {
    conn.setAutoCommit(false);
    for (int i = 0; i < 10000; i++) {
        ps.setString(1, "student" + i);
        ps.setInt(2, 20);
        ps.addBatch();                     // 1. 加入批次
        if (i % 1000 == 0) {
            ps.executeBatch();             // 2. 每 1000 条执行一次
            ps.clearBatch();
        }
    }
    ps.executeBatch();
    conn.commit();
}
```

> 需在 URL 上加 `rewriteBatchedStatements=true`（MySQL），否则批处理会被拆成单条发送，性能提升有限。

## 连接池

**为什么需要连接池？** 每次 `getConnection()` 都要建立 TCP 连接 + 认证，开销很大。连接池预先创建并复用连接，避免频繁创建销毁。

| 连接池 | 特点 |
|:---|:---|
| HikariCP | 性能极高，Spring Boot 2.x 默认 |
| Druid | 阿里开源，自带监控和 SQL 防火墙 |
| C3P0 | 老牌连接池，性能一般，逐渐被替代 |
| DBCP | Apache 出品，配置简单 |

```java
// HikariCP 示例
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/db");
config.setUsername("root");
config.setPassword("password");
config.setMaximumPoolSize(10);        // 最大连接数
config.setMinimumIdle(5);             // 最小空闲连接
config.setConnectionTimeout(30000);   // 获取连接超时

DataSource dataSource = new HikariDataSource(config);
Connection conn = dataSource.getConnection();   // 从池中借出
// ... 使用完毕后 close() 实际是归还到池中，而非真正关闭
```

**核心参数说明**：

| 参数 | 含义 |
|:---|:---|
| `maximumPoolSize` | 最大连接数，过大反而拖垮数据库 |
| `minimumIdle` | 最小空闲连接数 |
| `connectionTimeout` | 获取连接的最大等待时间 |
| `idleTimeout` | 空闲连接存活时间 |
| `maxLifetime` | 连接最大生命周期，应小于数据库的 wait_timeout |

## 常见问题

**1. 为什么 `Class.forName` 能注册驱动？**

`Class.forName` 会触发类的初始化，执行静态代码块，而驱动类的静态块中调用了 `DriverManager.registerDriver()`。

**2. JDBC 连接为什么必须关闭？**

连接是稀缺资源，不关闭会导致连接泄漏，最终耗尽数据库连接数（`Too many connections`）。用 try-with-resources 或在 finally 中关闭。

**3. `executeQuery` 和 `executeUpdate` 的区别？**

`executeQuery` 返回 `ResultSet`，用于查询；`executeUpdate` 返回影响行数（int），用于增删改。

**4. PreparedStatement 是万能的吗？**

不是。它只能用于参数位置（`?` 只能替换值，不能替换表名、列名）。需要动态拼接表名或列名时只能字符串拼接，此时必须做白名单校验防注入。

**5. 为什么生产环境不用原生 JDBC？**

- 样板代码多，重复繁琐（连接、关闭、异常处理）
- 结果集需手动映射为对象
- 事务、连接池管理都要自己实现

这些正是 MyBatis 等持久化框架要解决的问题，参见 [MyBatis](../../中间件/Mybatis/mybatis.md)。

## 高频深挖面试题


### 1. PreparedStatement 防注入，靠的真是「预编译」吗？

**问**：PreparedStatement 为什么能防止 SQL 注入？

**答**：因为它预编译了，参数不会参与 SQL 语法解析。

**追问**：那你告诉我，MySQL 默认情况下到底有没有做服务端预编译？

**深答**：这里有个普遍的误解。MySQL Connector/J 的 `useServerPrepStmts` **默认是 false**，也就是说默认情况下**服务端根本没做预编译**——驱动是在客户端把参数转义后拼进 SQL 发过去的。

所以严格来说：

| 场景 | 防注入靠什么 |
|:---|:---|
| `useServerPrepStmts=false`（默认） | 靠**客户端的转义**，不是预编译 |
| `useServerPrepStmts=true` | 靠**服务端预编译**（参数与结构分离） |

**但两种情况下都是安全的**，只是机制不同：默认模式下驱动会严格转义 `'`、`\` 等字符；开启服务端预编译后，参数走独立的协议通道，压根不进入 SQL 文本。

**真正要记住的是**：`Statement` 的问题不在于「没预编译」，而在于**它根本没有参数的概念**——靠字符串拼接，转义责任落在开发者身上，一句手写的拼接就漏了。

---

### 2. 为什么 JDBC 4.0 之后可以不写 `Class.forName`？

**问**：加载驱动为什么要写 `Class.forName("com.mysql.cj.jdbc.Driver")`？现在不写为什么也能跑？

**答**：因为 `Class.forName` 会触发类初始化，执行驱动静态块里的 `DriverManager.registerDriver()`。

**追问**：那 `ClassLoader.loadClass` 行不行？两者区别是什么？

**深答**：**不行**。区别就在「是否初始化」：

```java
Class.forName(name, true, loader);   // 会初始化 → 执行 static 块 → 注册成功
ClassLoader.loadClass(name);         // 只加载，不初始化 → static 块不执行 → 注册失败
```

`Class.forName(String)` 内部等价于第二个参数传 `true`，所以能注册。

**再追问**：那 JDBC 4.0 之后不写为什么也行？

**深答**：靠 **SPI 机制**。驱动 jar 包里有 `META-INF/services/java.sql.Driver` 文件，内容是实现类的全限定名。`ServiceLoader` 会扫描这个文件，用**反射**实例化驱动，自动完成注册。

**继续追问**：这里其实藏着双亲委派被破坏的问题，你能说说吗？

**深答**：能。`DriverManager` 在 `java.sql` 包下，由**启动类加载器**加载；而驱动在应用 classpath 下，需要**应用类加载器**加载。

按双亲委派，父加载器看不到子加载器的类，`DriverManager` 理应找不到驱动。JDBC 通过**线程上下文类加载器**（`Thread.currentThread().getContextClassLoader()`）拿到了应用类加载器，从而加载到驱动——这就是著名的「破坏双亲委派」案例。

---

### 3. 连接池的 `close()` 为什么不会真的关连接？

**问**：连接池里 `conn.close()` 后连接为什么不关闭？

**答**：因为连接池返回的不是原生 `Connection`，而是**代理对象**，`close()` 被重写成了「归还到池中」。

**追问**：具体怎么实现的？如果是你，怎么让用户调用 `close()` 时插进自己的逻辑？

**深答**：用**动态代理**（JDK `Proxy` 或 CGLIB）包装原生连接，拦截 `close()` 等关键方法：

```java
public class PooledConnection implements InvocationHandler {
    private final Connection realConn;
    private final ConnectionPool pool;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("close".equals(method.getName())) {
            pool.release(realConn);   // 归还，而非关闭
            return null;
        }
        return method.invoke(realConn, args);   // 其他方法直接转发
    }
}
```

HikariCP 的 `ProxyConnection` 就是这个思路。注意它还会拦截 `isClosed()`、`getAutoCommit()` 等做状态修正——因为用户看到的语义应该是「池化的连接」语义，而不是物理连接语义。

**再追问**：那有个坑——用户如果调用了 `close()` 又继续用这个对象呢？

**深答**：代理对象内部会维护一个 `closed` 标志位。归还后再调用任何方法就抛 `SQLException`，防止「已归还的连接被继续使用」这种隐蔽 bug。这也是为什么不建议手动持有 `Connection` 引用。

---

### 4. `maxLifetime` 为什么必须小于数据库的 `wait_timeout`？

**问**：连接池的 `maxLifetime` 参数有什么讲究？

**答**：它控制连接的最大存活时间，超过就销毁重建。

**追问**：那这个值怎么设？设成比数据库的 `wait_timeout` 大行不行？

**深答**：**绝对不行**，这是个线上高频坑。MySQL 的 `wait_timeout` 默认 28800 秒（8 小时），服务端会主动关闭「空闲超时」的连接。

如果 `maxLifetime` 设得比它大（或者压根不设），会出现：

```text
1. 一条连接长时间空闲，服务端已经悄悄把它关了
2. 连接池完全不知情，还认为这条连接可用
3. 应用借出这条「死连接」去执行 SQL
4. 报错：Communications link failure
```

**这个 bug 的恶心之处**：它**间歇性复现**，而且往往在低峰期之后流量回升时集中爆发，因为那些空闲连接刚被服务端清掉。排查时看到连接状态正常，实际早已失效。

**正确做法**：`maxLifetime` 设为明显小于 `wait_timeout` 的值（如 1 小时 vs 8 小时），让连接在被数据库关闭**之前**就由池主动回收重建。

**再追问**：那 `maxLifetime` 设太小会怎样？

**深答**：连接频繁重建，失去了池化的意义，增加建连开销。所以是「小于但不是远小于」，一般取 `wait_timeout` 的 1/2 甚至更小，同时要保证大于单次业务的最长执行时间——否则会出现「连接用着用着被池回收了」。

---

### 5. MySQL 的批处理为什么开了却没效果？

**问**：JDBC 批处理为什么快？怎么用？

**答**：用 `addBatch()` + `executeBatch()`，减少网络往返。

**追问**：我在 MySQL 上用了批处理，为什么性能没有提升？

**深答**：因为 **MySQL 驱动默认会把批处理拆成单条 SQL 逐条发送**，`addBatch` 只起到了「攒起来」的作用，网络往返一次没省。

必须显式开启参数：

```text
jdbc:mysql://localhost:3306/db?rewriteBatchedStatements=true
```

开启后驱动会把批量 INSERT 重写成多值形式：

```sql
-- 批处理前（N 次往返）
insert into t(name) values('a');
insert into t(name) values('b');

-- 开启 rewriteBatchedStatements 后（1 次往返）
insert into t(name) values('a'),('b');
```

性能差异可以达到**数十倍**，这是 MySQL 上最容易被忽略的优化点之一。

**再追问**：那批处理还有别的坑吗？

**深答**：有。一是 `executeBatch()` 返回的是**受影响行数数组**，不是总行数，需要自己累加；二是批量过大可能触发 `max_allowed_packet` 限制，需要分批（如每 1000 条执行一次）；三是如果某条失败，已执行的部分不会自动回滚，需要放在手动事务里配合处理。

---

### 6. 归还连接前为什么要恢复 `autoCommit`？

**问**：用连接池时，手动控制事务要注意什么？

**答**：记得 `setAutoCommit(false)` 开启事务，`commit()` 或 `rollback()` 结束事务。

**追问**：那事务结束后呢？直接归还连接行不行？

**深答**：**不行，必须恢复 `autoCommit(true)`**。原因是连接会被**复用**：

```java
conn.setAutoCommit(false);   // 开启事务
try {
    // 业务逻辑
    conn.commit();
} finally {
    conn.setAutoCommit(true);  // 关键点：归还前恢复
}
```

如果忘了恢复，下个使用者从池里拿到这条连接时，它的 `autoCommit` 还是 `false`。那么他执行的 SQL **不会自动提交**，会一直挂在事务里不生效——现象是「SQL 执行了但数据没变化」，或者「代码跑完了但没落库」。

**这个问题的隐蔽性在于**：它会不会出问题取决于「拿到的是哪条连接」。同一个业务，复用到了「干净」的连接就正常，复用到了「脏」连接就出错，表现为**偶发**、**莫名其妙的数据丢失**。

**再追问**：那连接池不管这个吗？

**深答**：主流连接池（HikariCP、Druid）在归还时会做**状态重置**，包括 autoCommit、readOnly、隔离级别等，所以通常不会出问题。但这属于「兜底」，不能依赖——自己显式恢复才是最稳妥的，而且在排查问题时能排除这条干扰项。

---

### 7. 为什么说 JDBC 的异常设计不合理？

**问**：你觉得 JDBC 有哪些设计上的缺陷？

**答**：`SQLException` 是检查型异常，到处都要 try-catch，代码很臃肿。

**追问**：除了「检查型异常」这点，还有什么更深的问题？

**深答**：更本质的问题是**异常分类的缺失**。JDBC 只抛一个 `SQLException`，但不同异常的处理策略完全不同：

| 异常性质 | 例子 | 正确处理 |
|:---|:---|:---|
| **瞬态**（可重试） | 死锁、连接超时、锁等待超时 | 重试可能成功 |
| **非瞬态**（重试无用） | SQL 语法错误、表不存在、字段超长 | 重试只会浪费资源 |

`SQLException` 只提供 `getErrorCode()`（厂商错误码）和 `getSQLState()`（标准状态码），要区分这两类，你得**自己解析错误码**。而不同数据库的错误码还不一样——MySQL 死锁是 1213，Oracle 是 60，代码里就得写一堆 `if`。

**追问**：那怎么解决这个问题？

**深答**：Spring 的 `DataAccessException` 体系就是答案。它把异常重新分类：

```text
DataAccessException
├── TransientDataAccessException      ← 瞬态，可重试
│   ├── DeadlockLoserDataAccessException
│   ├── QueryTimeoutException
│   └── ConcurrencyFailureException
└── NonTransientDataAccessException   ← 非瞬态，重试无用
    ├── BadSqlGrammarException
    ├── DataIntegrityViolationException
    └── ...
```

它通过 `SQLExceptionTranslator` 把各数据库的错误码统一翻译成这套分类。这样业务代码只需要 `catch (TransientDataAccessException e) { 重试 }`，就实现了**与数据库无关的重试逻辑**——这才是解决异常设计缺陷的正道。

---

### 8. MyBatis 到底在 JDBC 之上做了什么？

**问**：MyBatis 和 JDBC 是什么关系？

**答**：MyBatis 是对 JDBC 的封装，底层还是 JDBC。

**追问**：那它具体封装了哪些东西？如果让你手写一个简化版 MyBatis，你要解决哪几件事？

**深答**：本质上要解决四件事，每一件都对应 JDBC 的一个痛点：

| 要解决的问题 | JDBC 的做法 | MyBatis 的做法 |
|:---|:---|:---|
| **连接从哪来** | 手动 `getConnection` + `close` | 交给 `DataSource` / 事务管理器 |
| **参数怎么绑** | 手动 `setXxx(index, value)` | 反射读对象字段，自动绑定 |
| **结果怎么映射** | 手动 `while(rs.next())` + 逐字段 set | 反射 + `ResultMap` 自动映射 |
| **SQL 放哪** | 硬编码在 Java 里 | 抽离到 XML / 注解 |

**追问**：这里面技术上最难的是哪个？

**深答**：**参数绑定和结果映射**，因为要用到反射。

参数绑定的关键在 `ParamNameResolver`——它要解析方法参数名（默认情况下 Java 反射拿不到参数名，需要 `-parameters` 编译参数或 `@Param` 注解），把参数组织成一个 `Map`，再按 `#{name}` 里的名字取值。

结果映射更复杂，要处理：字段名与列名不一致（下划线转驼峰）、嵌套对象、集合类型、类型转换器（`TypeHandler`）。这也是为什么 MyBatis 有 `ResultMap` 这个看起来很繁琐的机制——它是在用配置的复杂度换取映射的灵活性。

**再追问**：那 MyBatis 的核心执行流程呢？

**深答**：`SqlSession` → `Executor` → `StatementHandler` → `ParameterHandler` / `ResultSetHandler`。

其中 `Executor` 负责缓存和事务，`StatementHandler` 负责创建 `PreparedStatement` 并执行。所以最终**一定**会走到 JDBC 的 `PreparedStatement.execute()`——这一点是绕不开的。

---

### 9. `SELECT *` 到底慢在哪？

**问**：为什么开发规范都禁止 `SELECT *`？

**答**：会查出不必要的数据，浪费带宽。

**追问**：如果这张表字段都挺小的，就多查了几个 int，影响大吗？

**深答**：影响**很大**，但原因不是带宽——真正的问题在**索引**。

假设有这样一个联合索引 `idx_abc(a, b, c)`，执行：

```sql
-- 可以走覆盖索引，不回表
SELECT a, b, c FROM t WHERE a = 1;

-- 必须回表，因为要返回所有字段
SELECT * FROM t WHERE a = 1;
```

覆盖索引意味着**只需要扫描索引就能拿到全部数据**，无需回表。回表则是「先在索引里找到主键，再拿主键去主键索引里取整行」——每行一次随机 IO。

数据量一大，覆盖索引和回表的差距可以是**数量级**的。

**追问**：还有别的影响吗？

**深答**：还有三个：

- **大字段拖累**：如果表里有 `TEXT`/`BLOB`，即使这些字段没用到也会被读出，网络和内存开销剧增。
- **影响索引选择**：`ORDER BY` 场景下，优化器可能因为要返回所有字段而放弃使用覆盖索引。
- **变更风险**：表新增字段后，`SELECT *` 的结果集结构会变化，可能导致映射报错或行为异常。

**一句话总结**：`SELECT *` 最大的代价是**失去了使用覆盖索引的机会**，而覆盖索引恰恰是 MySQL 最重要的优化手段之一。

---

### 10. 如果让你设计一个连接池，关键难点在哪？

**问**：让你实现一个简易连接池，你会怎么设计？

**答**：用阻塞队列存连接，`getConnection` 时 `take()`，归还时 `offer()` 回去。

**追问**：那用户调用 `close()` 的时候，怎么让它「归还」而不是真关闭？

**深答**：这是**最关键的一点**——必须用**动态代理**包装 `Connection`，拦截 `close()` 改成归还：

```java
if ("close".equals(method.getName())) {
    pool.release(realConn);
    return null;
}
```

如果直接返回原生 `Connection`，用户一调 `close()` 物理连接就断了，池化就没意义了。

**再追问**：还有哪些难点？

**深答**：主要有四个：

| 难点 | 问题 | 解决思路 |
|:---|:---|:---|
| **连接有效性** | 池里的连接可能已被服务端关闭 | 借出前 `isValid()` 校验，或用 `maxLifetime` 强制淘汰 |
| **并发安全** | 多线程同时借还 | 用 `BlockingQueue` / `Semaphore` 保证原子性 |
| **超时控制** | 池耗尽时不能让请求无限等待 | `poll(timeout)` 超时抛异常，避免线程堆积 |
| **泄漏检测** | 用户借了不还 | 记录借出时间与堆栈，超时未归还时告警或强制回收 |

**继续追问**：池大小怎么定？越大越好吗？

**深答**：**不是**，这是最常见的误区。连接数过大会导致：

- 数据库端每个连接都占用内存和线程，反而拖垮数据库。
- 应用侧线程上下文切换和锁竞争加剧。

经验公式是 `连接数 ≈ CPU核数 × 2 + 磁盘数`，但真正靠谱的做法是**压测**——从较小值开始逐步加压，观察 QPS 和 RT，找到拐点。HikariCP 作者的观点是：**连接数的瓶颈往往在磁盘 IO 和数据库本身，而不是 CPU**，所以盲目加连接通常没有收益。

**最后一问**：那 HikariCP 为什么比别的池快？

**深答**：几个关键点：一是用了自定义的 `FastList` 替代 `ArrayList` 做连接容器（省去了范围检查）；二是用 `ConcurrentBag` 做连接池容器，它是**无锁**的（基于 `ThreadLocal` + 原子操作），在高并发下避免了锁竞争；三是字节码精简，减少不必要的抽象层。核心思想就是**减少锁竞争和对象分配**。

## 参考文章

- [Oracle JDBC 官方文档](https://docs.oracle.com/javase/tutorial/jdbc/)
- [MySQL Connector/J 文档](https://dev.mysql.com/doc/connector-j/en/)
