# JDK 版本区别

Java 自 2017 年改为**每半年发布一个版本**的节奏，LTS（长期支持）版本每两年发布一次。当前主要的 LTS 版本为 Java 8、11、17、21。

---

## Java 8（2014-03 LTS）

### Lambda 表达式

允许将函数作为方法的参数传递。

```java
// 语法: (parameters) -> expression 或 (parameters) -> { statements; }
Arrays.asList("jay", "Eason", "SHE").forEach(s -> System.out.print(s));
```

### 函数式接口

使用 `@FunctionalInterface` 注解，接口中只有一个抽象方法（如 Runnable、Comparator）。

### 默认方法与静态方法

接口中可包含实现了的方法（default），也可包含静态方法（static）。

```java
public interface ISingerService {
    default void sing() {
        System.out.printLn("唱歌");
    }
    void writeSong();
}
```

### 方法引用

通过 `::` 直接引用已有类或实例的方法。

```java
Consumer<String> consumer = System.out::println;
```

### Stream API

| 操作 | 说明 |
|:---|:---|
| filter | 筛选 |
| map | 流映射 |
| reduce | 将流中元素组合 |
| collect | 返回集合 |
| sorted | 排序 |
| flatMap | 流扁平化转换 |
| limit | 返回指定个数的流 |
| distinct | 去重 |

### Optional

用于优雅处理 null，避免 NullPointerException。

### Date / Time API

`LocalDate`、`LocalDateTime`、`Instant`、`Duration`、`Period` 等不可变且线程安全。

### 重复注解

通过 `@Repeatable` 定义可在同一位置多次使用的注解。

### Base64

JDK 内置 Base64 编码/解码支持。

### JVM 元空间

使用元空间（Metaspace）代替永久代（PermGen），`-XX:MetaspaceSize` 和 `-XX:MaxMetaspaceSize` 控制大小。

---

## Java 9（2017-09）

### 模块化系统（JPMS）

`module-info.java` 定义模块，显式声明依赖和导出包，解决「类路径地狱」。

```java
module com.example {
    requires java.sql;
    exports com.example.api;
}
```

### JShell

交互式 REPL 工具，可直接运行 Java 代码片段。

### 集合工厂方法

`List.of()`、`Set.of()`、`Map.of()` 创建不可变集合。

### 接口私有方法

接口中允许定义 private 和 private static 方法。

### try-with-resources 改进

可在 try 中使用已存在的 final/effectively-final 变量。

### 钻石操作符扩展

允许匿名内部类使用 `<>` 钻石操作符。

---

## Java 10（2018-03）

### 局部变量类型推断（var）

```java
var list = new ArrayList<String>();
var map = new HashMap<String, Integer>();
```

仅用于局部变量，不改变 Java 静态类型本质。

### 时间基准版本发布

JDK 10 起采用严格的时间驱动发布模式，每 6 个月一个版本。

---

## Java 11（2018-09 LTS）

### 字符串增强

`isBlank()`、`strip()`、`stripLeading()`、`stripTrailing()`、`lines()`、`repeat(n)`。

### Lambda 局部变量语法

```java
var map = new HashMap<String, Object>();
map.forEach((var k, var v) -> System.out.println(k + ":" + v));
```

### 标准化 HTTP Client

支持 HTTP/1.1、HTTP/2、WebSocket，完全重写，支持同步和异步。

### 单文件编译运行

```bash
java HelloWorld.java    # 无需先 javac 编译
```

### ZGC（实验性）

可伸缩低延迟垃圾收集器，GC 停顿不超过 10ms，可处理几百 MB 到几个 TB 的堆。

### 移除项

移除 Java EE 和 CORBA 模块，移除 CGI、Finalizer 等过时功能。

---

## Java 12（2019-03）

### Switch 表达式（预览）

允许 switch 作为表达式返回值。

```java
var result = switch (day) {
    case MONDAY, FRIDAY -> "work";
    case SATURDAY, SUNDAY -> "rest";
    default -> "unknown";
};
```

### Shenandoah GC

低暂停时间 GC，与运行线程并发进行垃圾回收（Red Hat 贡献）。

### 微基准测试套件

JDK 源码中引入了 Java Microbenchmark Harness（JMH）测试。

---

## Java 13（2019-09）

### 文本块（预览）

```java
var json = """
    {
        "name": "jay",
        "age": 18
    }
    """;
```

### Switch 表达式（第二次预览）

引入 yield 返回值机制（替代 break 返回值）。

### Socket API 重实现

底层用更简单、更现代的方式重写了 java.net.Socket 和 ServerSocket。

---

## Java 14（2020-03）

### Records（预览）

```java
public record Point(int x, int y) { }
```

自动生成构造器、equals、hashCode、toString、getter 方法，不可变数据载体。

### Pattern Matching for instanceof（预览）

```java
if (obj instanceof String s) {
    System.out.println(s.length());
}
```

### 文本块（第二次预览）

### Helpful NullPointerExceptions

精确指出哪个变量为 null，而非仅打印行号。

### jpackage 打包工具

将 Java 应用打包成原生安装包（exe、dmg、deb）。

---

## Java 15（2020-09）

### 文本块（正式）

文本块功能正式发布。

### Sealed Classes（预览）

限制哪些类可以继承密封类。

### ZGC（生产可用）

ZGC 从实验性转为正式功能。

### Hidden Classes

用于框架和 JVM 语言，不支持外部直接使用。

### Edwards-Curve 数字签名算法

新增 EdDSA 签名算法支持。

---

## Java 16（2021-03）

### Records（正式）

### Pattern Matching for instanceof（正式）

### Unix-Domain Socket Channels

支持本机 Unix 域套接字通信。

### 弹性元空间

可主动将未使用的元空间内存返还给操作系统。

---

## Java 17（2021-09 LTS）

### 封闭类（Sealed Classes，正式）

```java
public sealed class Shape permits Circle, Square { }
```

### Pattern Matching for switch（预览）

```java
switch (obj) {
    case Integer i -> System.out.println("Integer: " + i);
    case String s -> System.out.println("String: " + s);
    default -> System.out.println("Other");
}
```

### 强封装（Strong Encapsulation）

默认禁止反射访问模块内部 API（需 `--add-opens` 手动开放）。

### 外部函数和内存 API（孵化器）

安全、高效地调用非 Java 代码（如 C 函数）和操作非堆内存。

### 增强的伪随机数生成器

新的接口和实现，提高随机数生成的灵活性和可扩展性。

### 移除项

永久删除 Applet，移除 RMI Activation。

---

## Java 18（2022-03）

### 简易 Web 服务器

`jwebserver` 命令行工具，提供静态文件服务。

### UTF-8 为默认编码

`Charset.defaultCharset()` 默认返回 UTF-8。

### 弃用 Finalization

建议使用 `try-with-resources` 和 `Cleaner` 替代 finalize()。

### 代码片段在 Javadoc

`@snippet` 标签支持在文档中嵌入代码片段。

---

## Java 19（2022-09）

### 虚拟线程（预览）

JDK 19 引入虚拟线程预览版，轻量级用户态线程，极大简化高并发编程。

### Record Patterns（预览）

```java
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + "," + y);
}
```

### Pattern Matching for switch（第三次预览）

### Linux/RISC-V 移植

---

## Java 20（2023-03）

### 虚拟线程（第二次预览）

改进 API，增强稳定性。

### Scoped Values（孵化器）

线程局部变量的替代方案，不可变，适合与虚拟线程配合。

### Record Patterns（第二次预览）

### Foreign Function & Memory API（第二次预览）

### 结构化并发（孵化器）

将多线程任务视为一个结构化单元管理。

---

## Java 21（2023-09 LTS）

### 虚拟线程（正式）

正式发布虚拟线程，作为 LTS 版本的核心特性。

### 结构化并发（正式）

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user = scope.fork(() -> findUser());
    Future<Integer> order = scope.fork(() -> fetchOrder());
    scope.join();
    // 处理结果
}
```

### 记录模式（正式）

```java
if (obj instanceof Point(int x, int y)) {
    // 使用 x, y
}
```

### Pattern Matching for switch（正式）

### 字符串模板（预览）

```java
String msg = STR."Hello \{name}!";
```

### 外部函数和内存 API（正式）

### 向量 API（正式）

利用向量化硬件加速批量计算。

### 分代 ZGC

ZGC 引入分代收集模式，降低 CPU 开销。

### 顺序集合

`SequencedCollection`、`SequencedSet`、`SequencedMap` 接口统一了集合的顺序操作。

---

## Java 22（2024-03）

### 语句在 super() 之前（预览）

```java
public class MyClass extends Base {
    MyClass() {
        var config = loadConfig();
        super(config);  // 之前不允许，Java 22 预览
    }
}
```

### Stream Gatherers（预览）

扩展 Stream API 的自定义中间操作。

### 隐式声明的类和实例 main 方法（预览）

更简洁的入门体验：

```java
void main() {
    System.out.println("Hello, World!");
}
```

### 字符串模板（第二次预览）

### 启动多文件源代码程序

允许直接运行由多个 .java 文件组成的程序。

### 区域描述符发布

`Region Pinning` 用于 G1 GC 减少延迟。

---

## Java 23（2024-09）

### Markdown in Javadoc 注释

文档注释支持 Markdown 语法。

### 模块导入声明（预览）

```java
import module java.base;
```

简化模块系统中导入的写法。

### 灵活构造器体（第二次预览）

完善 `super()` 之前允许写语句的功能。

### Stream Gatherers（第二次预览）

### 隐式声明的类（第二次预览）

### 弃用 32 位 Windows 移植

---

## Java 24（2025-03）

### Scoped Values（正式）

Scoped Values 从孵化到正式发布。

### 密钥封装机制 API（KEM）

量子安全加密传输的 API 支持。

### Stream Gatherers（正式）

Stream Gatherers 正式发布。

### 模块导入声明（第二次预览）

### 灵活构造器体（第三次预览）

### 量子抗性加密算法

引入 ML-KEM、ML-DSA、SLH-DSA 等量子安全算法。

---

## Java 25（2025-09 LTS）

### 灵活构造器体（正式）

允许在构造函数中先执行语句再调用 super()。

### 模块导入声明（正式）

简化模块导入语法正式发布。

### 值对象与原始类（预览，Valhalla 项目）

引入值类型，将不可变对象视为原始类型，消除对象头开销，提升性能。

### 结构化并发增强

StructuredTaskScope 引入更丰富的错误处理模型。

### ZGC 持续优化

进一步降低停顿时间，适配更大规模堆。

### G1 GC 增强

G1 在超大堆场景下的停顿时间进一步降低。
