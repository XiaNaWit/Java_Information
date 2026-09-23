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

## 参考文章

- [Oracle JDBC 官方文档](https://docs.oracle.com/javase/tutorial/jdbc/)
- [MySQL Connector/J 文档](https://dev.mysql.com/doc/connector-j/en/)
