# String

## 目录

- [String 的特性](#string-的特性)
- [为什么 String 设计成不可变](#为什么-string-设计成不可变)
- [储存数据](#储存数据)
- [字符常量和字符串常量的区别](#字符常量和字符串常量的区别)
- [String Pool（字符串常量池）](#string-pool字符串常量池)
- [String 拼接的编译优化](#string-拼接的编译优化)
- [StringBuilder 与 StringBuffer](#stringbuilder-与-stringbuffer)
- [String 的 hashCode()](#string-的-hashcode)
- [常用 API 方法](#常用-api-方法)
  - [1. 长度与判空](#1-长度与判空)
  - [2. 查找与判断](#2-查找与判断)
  - [3. 截取、替换、分割](#3-截取替换分割)
  - [4. 大小写与空白处理](#4-大小写与空白处理)
  - [5. 转换与格式化](#5-转换与格式化)
  - [6. 简单示例（综合）](#6-简单示例综合)
  - [7. 方法使用要点小结](#7-方法使用要点小结)
- [面试题](#面试题)
  - [一、不可变性与设计](#一不可变性与设计)
  - [二、常量池与内存](#二常量池与内存)
  - [三、字符串操作与性能](#三字符串操作与性能)
  - [四、编码与字符](#四编码与字符)
  - [五、常用方法相关](#五常用方法相关)

## String 的特性

String 有三个核心特性：

**1. 不可变性（Immutable）**

String 是典型的不可变对象：对它进行任何「修改」操作，实际都是**创建一个新对象**，再把引用指向新对象，原对象保持不变。

```java
String s = "abc";
s = s + "d";          // 并非修改原对象，而是新建 "abcd" 并让 s 指向它
                      // 原来的 "abc" 对象仍存在于常量池中
```

**2. 常量池优化**

字符串字面量创建后会在**字符串常量池**中缓存。下次创建相同内容时，直接返回池中的引用，避免重复创建。

```java
String s1 = "abc";
String s2 = "abc";
System.out.println(s1 == s2);   // true，指向同一个常量池对象
```

**3. final 修饰**

String 类被声明为 `final`，不可被继承。这既是**安全性**考虑（防止子类篡改 String 行为），也是**不可变的保障**（子类若可变就破坏了不可变性）。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    // ...
}
```

## 为什么 String 设计成不可变

不可变带来的好处可以归纳为四点：

| 好处 | 原因 |
|:---|:---|
| **线程安全** | 不可变对象天然线程安全，多线程共享无需同步 |
| **可缓存 hash 值** | hash 值不变，只需计算一次并缓存，适合做 HashMap 的 key |
| **常量池可行** | 只有不可变，多个引用共享同一对象才安全，String Pool 才成立 |
| **安全性** | 作为参数传递时不会被中途篡改（如网络连接的主机地址、文件路径） |

**安全性举例**：假设 String 可变，那么把一个主机地址传给网络连接方法后，另一个线程修改了这个 String，就会导致实际连接的主机与预期不一致——这是安全漏洞。

**不可变的实现保障**：

1. 类被 `final` 修饰，无法被继承重写
2. 内部存储数组被 `final` 修饰（引用不可变）
3. **不提供任何修改内部数组的方法**（这是最关键的，仅有前两条还不够）
4. 涉及返回内部的构造时做防御性拷贝

## 储存数据

**Java 8 及之前**：使用 `char` 数组存储，每个字符固定占 2 字节。

```java
private final char value[];
```

**Java 9 及之后**：改用 `byte` 数组 + `coder` 标识编码，称为「**紧凑字符串（Compact Strings）**」。

```java
private final byte[] value;
private final byte coder;    // LATIN1(0) 或 UTF16(1)
```

**为什么要改？** Java 8 中即使字符串全是 ASCII 字符（如英文、数字、URL），每个字符也要占 2 字节，浪费一半空间。Java 9 的优化是：

- 若字符串**只含 Latin-1 字符**（单字节可表示）→ 用 `byte[]` 每字符 1 字节，`coder = LATIN1`
- 若含**超出 Latin-1 的字符**（如中文）→ 仍用 UTF-16 每字符 2 字节，`coder = UTF16`

这样纯英文场景内存占用直接减半，是 JDK 9 的重要优化（JEP 254）。

## 字符常量和字符串常量的区别

| 对比项 | 字符常量 | 字符串常量 |
|:---|:---|:---|
| **形式** | 单引号，仅一个字符 `'a'` | 双引号，若干字符 `"abc"` |
| **类型** | 基本类型 `char` | 引用类型 `String` |
| **本质** | 一个 **16 位整型值**（Unicode 码点） | 一个**对象**（地址值指向堆/常量池） |
| **占内存** | 固定 **2 字节** | 至少 1 个字符，视长度而定 |
| **能否运算** | ✅ 可参与算术运算 | ❌ 不能（`"a" + "b"` 是拼接，不是加法） |

**易错点**：

```java
char c = 'a';
System.out.println(c + 1);        // 98，字符参与算术运算，取的是码点值

String s = "a";
System.out.println(s + 1);        // a1，字符串拼接

System.out.println('a' + 1);      // 98
System.out.println("a" + 1);      // a1
```

> **注意**：`'a' + 1` 是数值运算得到 98，而 `"a" + 1` 是字符串拼接得到 `"a1"`。这是面试常见的区分点。
## String Pool（字符串常量池）

### 三种常量池的区分（易混淆）

Java 中「常量池」这个概念实际涉及三个不同的东西：

| 名称 | 位置 | 内容 | 生命周期 |
|:---|:---|:---|:---|
| **class 文件常量池** | `.class` 文件中 | 编译期生成的字面量和符号引用 | 文件级 |
| **运行时常量池** | 方法区/元空间 | class 常量池加载后的运行时表示 | 类卸载前 |
| **全局字符串常量池** | JDK6 永久代 → JDK7+ **堆** | 字符串字面量（String Pool） | JVM 生命周期 |

我们平时说的「字符串常量池」指**第三个**。

### 什么是字符串常量池

JVM 为了提升性能和减少内存开销，避免字符串的重复创建，维护了一块特殊的内存空间——字符串池。

使用字符串时，先去池中查看是否已存在：存在则直接复用，不存在则创建并放入池中。

### 常量池位置的版本变化

| JDK 版本 | String Pool 位置 | 存储内容 |
|:---|:---|:---|
| JDK 6 | 永久代（方法区） | **对象**本身 |
| **JDK 7** | **堆** | **引用**（指向堆中的字符串对象） |
| JDK 8+ | 堆（永久代被元空间取代） | 引用 |

**JDK 7 为何要移到堆？** 永久代空间有限，在大量使用字符串的场景下（如 `intern()` 频繁调用）容易导致 `OutOfMemoryError: PermGen space`。移到堆中后可受 GC 管理，容量也更灵活。

### new String("abc") 会创建几个对象

**创建 2 个对象**（前提是池中还没有 `"abc"`）：

```java
String s = new String("abc");
```

1. **常量池检查阶段**：`"abc"` 是字面量，若池中不存在，则在 String Pool 创建一个对象
2. **new 阶段**：在**堆**中再创建一个新的 String 对象

所以是 **1 个池对象 + 1 个堆对象**。

> 如果池中**已存在** `"abc"`，则只创建 **1 个**堆对象。

**关键点**：`new String("abc")` 的堆对象与常量池对象是**两个不同的对象**：

```java
String s1 = new String("abc");
String s2 = "abc";
System.out.println(s1 == s2);        // false，堆对象 vs 池对象
System.out.println(s1.equals(s2));   // true，内容相同
```

### intern()

`intern()` 用于**手动将字符串放入常量池并返回池中引用**：

- 如果池中已存在值相等的字符串 → 直接返回池中引用
- 如果池中不存在 → 在池中创建（或记录引用）并返回

```java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);       // false，两个不同的堆对象

String s3 = s1.intern();
String s4 = s2.intern();
System.out.println(s3 == s4);       // true，都指向池中同一个对象

// 字面量形式会自动入池
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);       // true
```

**intern() 的实际应用**：用于处理大量重复字符串以节省内存（如解析日志、CSV 中的枚举值），但要注意：

- 池中的字符串**不会被 GC 回收**（JDK7+ 弱引用有例外），大量不同的 `intern()` 可能造成内存泄漏
- JDK6 时 `intern()` 性能较差（需在永久代分配），JDK7+ 移到堆后有明显改善

**经典面试题**：

```java
String s = new String("1") + new String("2");
s.intern();
String s2 = "12";
System.out.println(s == s2);   // JDK6: false;  JDK7+: true
```

**为什么 JDK7+ 是 true？** 因为 `"1" + "2"` 通过 `StringBuilder` 拼接后，结果 `"12"` 是**堆中的对象**。此时调用 `intern()`，池中并不存在 `"12"`，JDK7+ 会把**堆中该对象的引用**直接放入池中（而非复制一份对象），所以 `s2` 拿到的就是同一个引用。

## String 拼接的编译优化

### 变量拼接 → StringBuilder

```java
String str1 = "a";
String result = str1 + " a nice day";
```

编译后等价于：

```java
String result = new StringBuilder()
        .append(str1)
        .append(" a nice day")
        .toString();
```

**注意**：这是在**编译期**就确定的优化，所以**循环中拼接字符串极其低效**——每次循环都会 new 一个 StringBuilder：

```java
// ❌ 低效：每次循环都创建 StringBuilder
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;
}

// ✅ 正确：循环外创建，循环内复用
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
```

### final 变量的优化

如果拼接的变量被 `final` 修饰，它会在初始化时**作为常量加载到常量池**，拼接时直接替换为字面量值：

```java
final String str1 = "value";
String result = str1 + " a nice day";
// 编译后直接变成：String result = "value a nice day";
```

此时**不会有 StringBuilder 的创建开销**——这是 `final` 带来的一个实用优化。

### 常量折叠

纯字面量的拼接在**编译期**就直接合并，运行时**不会创建任何对象**：

```java
String s = "a" + "b" + "c";
// 编译后：String s = "abc";
```

**对比验证**：

```java
String a = "abc";
String b = "a" + "b" + "c";
System.out.println(a == b);   // true，编译期已折叠为同一个常量

String c = "a";
String d = c + "bc";
System.out.println(a == d);   // false，含变量的拼接在运行期完成
```

### 什么是字符串常量池？

java中常量池的概念主要有三个：`全局字符串常量池`，`class文件常量池`，`运行时常量池`。我们现在所说的就是`全局字符串常量池`

参考文献：[Java中几种常量池的区分](http://tangxman.github.io/2015/07/27/the-difference-of-java-string-pool/)。

jvm为了提升性能和减少内存开销，避免字符的重复创建，其维护了一块特殊的内存空间，即字符串池，当需要使用字符串时，先去字符串池中查看该字符串是否已经存在，如果存在，则可以直接使用，如果不存在，初始化，并将该字符串放入字符串常量池中。

字符串常量池的位置也是随着jdk版本的不同而位置不同。在jdk6中，常量池的位置在永久代（方法区）中，此时常量池中存储的是**对象**。在jdk7中，常量池的位置在堆中，此时，常量池存储的就是**引用**了。在jdk8中，永久代（方法区）被元空间取代了。


### new String("abc")会创建几个对象
- 使用这种方式一共会创建两个字符串对象（前提是 String Pool 中还没有 "abc" 字符串对象）。
- "abc" 属于字符串字面量，因此编译时期会在 String Pool 中创建一个字符串对象，指向这个 "abc" 字符串字面量；
- 而使用 new 的方式会在堆中创建一个字符串对象。
### intern()
字符串常量池（String Pool）保存着所有字符串字面量（literal strings），这些字面量在编译时期就确定。不仅如此，还可以使用 String 的 intern() 方法在运行过程中将字符串添加到 String Pool 中。当一个字符串调用 intern() 方法时，如果 String Pool 中已经存在一个字符串和该字符串值相等（使用 equals() 方法 进行确定），那么就会返回 String Pool 中字符串的引用；否则，就会在 String Pool 中添加一个新的字符串，并返回这个新字符串的引用。
```
下面示例中，s1 和 s2 采用 new String() 的方式新建了两个不同字符串，而 s3 和 s4 是通过 s1.intern() 方法取得一
个字符串引用。intern() 首先把 s1 引用的字符串放到 String Pool 中，然后返回这个字符串引用。因此 s3 和 s4 引用
的是同一个字符串。
String s1 = new String("aaa"); 
String s2 = new String("aaa"); 
System.out.println(s1 == s2); // false
 
String s3 = s1.intern(); 
String s4 = s1.intern(); 
System.out.println(s3 == s4); // true

如果是采用 "bbb" 这种字面量的形式创建字符串，会自动地将字符串放入 String Pool 中。
String s5 = "bbb"; 
String s6 = "bbb"; 
System.out.println(s5 == s6); // true
```
### str1 + " a nice day"
- 编译为 new StringBuilder().append(str1).append(" a nice day");
- 但是如果str1被final修饰，此变量会在初始化时加载到常量池，所以会直接变为str1的值"value"+"a nice day"
### "a" + "b" + "c"
编译优化不会创建对象

## StringBuilder 与 StringBuffer

### 三者的本质区别

| 对比项 | String | StringBuilder | StringBuffer |
|:---|:---|:---|:---|
| **可变性** | ❌ 不可变 | ✅ 可变 | ✅ 可变 |
| **线程安全** | ✅ 天然安全（不可变） | ❌ 不安全 | ✅ 安全（方法加 `synchronized`） |
| **性能** | 拼接时最慢 | **最快** | 较慢（同步开销） |
| **适用场景** | 少量操作、常量 | 单线程大量拼接 | 多线程大量拼接 |

**为什么可变？** `StringBuilder` / `StringBuffer` 内部的 `byte[]` 数组**没有被 `final` 修饰**，可以动态扩容修改，所以拼接时不需要创建新对象。

```java
// StringBuilder 内部结构（简化）
abstract class AbstractStringBuilder {
    byte[] value;      // 注意：没有 final
    int count;         // 已使用长度
}
```

### 底层扩容机制

`StringBuilder` 默认初始容量为 **16**，容量不足时扩容为 `(旧容量 << 1) + 2`，即**约 2 倍 + 2**：

```java
// AbstractStringBuilder 的扩容逻辑
private int newCapacity(int minCapacity) {
    int newCapacity = (value.length << 1) + 2;   // 2倍 + 2
    if (newCapacity - minCapacity < 0) {
        newCapacity = minCapacity;
    }
    return newCapacity;
}
```

**优化建议**：如果能预估长度，**创建时指定容量**可避免扩容：

```java
// 已知要拼接 1000 个字符
StringBuilder sb = new StringBuilder(1024);
```

### 如何选择

- **少量数据** → `String`（编译期优化后已足够）
- **单线程大量拼接** → `StringBuilder`（无锁，最快）
- **多线程大量拼接** → `StringBuffer`（有锁，安全）

> **实践提示**：多线程场景下，用 `StringBuilder` + 局部变量（每个线程自己的实例）通常比 `StringBuffer` 更好——避免了锁竞争。`StringBuffer` 的使用场景其实很少。

## String 的 hashCode()

String 遵守 `equals` 与 `hashCode` 的契约：`s.equals(s1)` 为 true，则 `s.hashCode() == s1.hashCode()`。

源码如下：

```java
public int hashCode() {
    int h = hash;                       // 默认 0
    if (h == 0 && value.length > 0) {   // 未计算过才计算
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];        // 多项式哈希，乘数 31
        }
        hash = h;                       // 缓存结果
    }
    return h;
}
```

**两个关键设计**：

**1. 缓存 hash 值**：hash 只在首次调用时计算，之后直接返回缓存的 `hash` 字段。这是「不可变」带来的好处——值不会变，缓存永远有效。

> 这也是 **String 适合做 HashMap key** 的原因：hash 计算一次即可，后续查找无需重复计算，相比普通对象更快。

**2. 乘数 31**：公式为 `h = 31 * h + val[i]`。

### 为什么要选 31 作为乘数

主要有两个原因：

| 原因 | 说明 |
|:---|:---|
| **31 是「不大不小」的质数** | 质数能减少哈希冲突。太小（如 2）会导致冲突多、信息丢失；太大（如 101）容易溢出且计算慢。37、41、43 也是不错的选择 |
| **31 可被 JVM 优化为位运算** | `31 * i = (i << 5) - i`，移位比乘法快，JVM 会自动做这个优化 |

**为什么「不大不小」很重要**：

- **太小**：如用 2，`h = 2*h + c`，高位信息容易溢出丢失，冲突率上升。
- **太大**：如用 101，虽然冲突少，但 `h` 更快超过 int 范围（溢出），且乘法开销更大。

**补充**：31 还被认为在数学上具有较好性质——当乘数为奇质数时，`31 * i` 的结果在低位不会出现规律性重复，有助于分散哈希值。

### 为什么不用乘数 1

如果乘数为 1，公式变成 `h = h + val[i]`，那么所有**字符相同但顺序不同**的字符串会得到相同的 hash 值：

```
"abc" → 97 + 98 + 99 = 294
"cba" → 99 + 98 + 97 = 294   // 哈希冲突！
```

固定大于 1 的乘数可以让**位置信息参与运算**，避免这种冲突。

## 常用 API 方法

String 的方法很多，按用途分类记忆更高效。**核心前提**：String 不可变，所有「修改」类方法都**返回新字符串**，原对象不变。

### 1. 长度与判空

| 方法 | 说明 | 版本 |
|:---|:---|:---|
| `length()` | 返回**字符数**（注意不是字节数） | 1.0 |
| `isEmpty()` | 是否为 `""`（长度 0） | 1.6 |
| `isBlank()` | 是否为空**或只含空白字符** | **11+** |
| `charAt(int i)` | 取索引 i 处的字符 | 1.0 |
| `toCharArray()` | 转为 `char[]` | 1.0 |
| `codePointCount(begin, end)` | 统计**码点**数量（emoji 等正确计数） | 1.5 |

```java
String s = "  ";
s.length();     // 2，有长度
s.isEmpty();    // false，不是空串
s.isBlank();    // true，全是空白 → 这才是「业务意义上的空」

String emoji = "😀";
emoji.length();                                  // 2（代理对占两个 char）
emoji.codePointCount(0, emoji.length());         // 1（真实字符数）
```

> ⚠️ **易错点**：判断「用户输入是否为空」应该用 `isBlank()` 而不是 `isEmpty()`——用户输入几个空格，`isEmpty()` 返回 false 但业务上应视为空。`isBlank()` 需要 **Java 11+**。

### 2. 查找与判断

| 方法 | 说明 | 返回 |
|:---|:---|:---|
| `indexOf(String/char)` | 首次出现的索引 | 找不到返回 **-1** |
| `indexOf(s, fromIndex)` | 从指定位置开始找 | -1 |
| `lastIndexOf(...)` | 最后一次出现的索引 | -1 |
| `contains(CharSequence)` | 是否包含子串 | boolean |
| `startsWith(String)` | 是否以某前缀开头 | boolean |
| `startsWith(String, offset)` | 从指定位置判断前缀 | boolean |
| `endsWith(String)` | 是否以某后缀结尾 | boolean |
| `matches(String regex)` | 是否**完全匹配**正则 | boolean |
| `equals(Object)` | 内容是否相等 | boolean |
| `equalsIgnoreCase(String)` | 忽略大小写比较 | boolean |
| `compareTo(String)` | 字典序比较 | 差值（int） |
| `compareToIgnoreCase(String)` | 忽略大小写字典序比较 | int |
| `contentEquals(CharSequence)` | 与任意字符序列比内容 | boolean |

```java
String s = "Hello World";

s.indexOf("o");            // 4（首次出现）
s.lastIndexOf("o");        // 7（最后出现）
s.indexOf("z");            // -1（不存在）

// ⚠️ matches 是完全匹配，不是包含！
"abc123".matches("\\d+");   // false，因为含字母
"123".matches("\\d+");      // true
"abc123".matches(".*\\d+.*"); // true，要「包含数字」得这样写
```

> ⚠️ **两个高频易错点**：
> 1. `indexOf` 找不到返回 **-1**，不是 0。判断时必须 `!= -1`，写成 `if (s.indexOf("x"))` 会逻辑错误（-1 在 Java 中不为 false）。
> 2. **`matches()` 是完全匹配**，不是包含匹配。判断「包含」要用 `contains()` 或 `.*xxx.*`。

**`compareTo()` 的返回值含义**：

```java
"a".compareTo("b");    // 负数（a 在 b 前）
"b".compareTo("a");    // 正数
"a".compareTo("a");    // 0

// 常用于排序
list.sort(String::compareTo);
```

### 3. 截取、替换、分割

| 方法 | 说明 |
|:---|:---|
| `substring(int begin)` | 从 begin 截到末尾 |
| `substring(int begin, int end)` | 截取 `[begin, end)`（**含头不含尾**） |
| `replace(char, char)` | 替换**所有**指定字符 |
| `replace(CharSequence, CharSequence)` | 替换**所有**子串 |
| `replaceAll(regex, replacement)` | **正则**替换全部 |
| `replaceFirst(regex, replacement)` | **正则**替换第一个 |
| `split(String regex)` | 按正则分割 |
| `split(String regex, int limit)` | 限制分割数量 |
| `concat(String)` | 拼接（等价于 `+`） |
| `String.join(delimiter, ...)` | 静态方法，用分隔符拼接 |
| `repeat(int n)` | 重复 n 次 | **11+** |

```java
String s = "Hello World";

s.substring(6);         // "World"
s.substring(0, 5);      // "Hello"，下标 0~4

s.replace('l', 'L');                    // "HeLLo WorLd"
s.replace("World", "Java");             // "Hello Java"
s.replaceAll("\\s+", "");               // "HelloWorld"（正则去空白）
s.replaceFirst("l", "L");               // "HeLlo World"

s.split(" ");                           // ["Hello", "World"]
"a,b,,c".split(",");                    // ["a", "b", "", "c"]
"a,b,,c".split(",", 2);                 // ["a", "b,,c"]，限制为 2 段

String.join("-", "a", "b", "c");        // "a-b-c"
"ab".repeat(3);                         // "ababab"
```

> ⚠️ **`split` 的坑**：
> 1. **参数是正则不是普通字符串**。按 `.` 分割必须转义：`split("\\.")`，直接写 `split(".")` 会得到空数组（因为 `.` 匹配任意字符）。
> 2. **末尾空串会被丢弃**：`"a,b,".split(",")` 返回 `["a", "b"]`（长度为 2，不是 3）。要保留需用 `split(",", -1)`。
> 3. **`replaceAll` 的替换串中 `$` 和 `\` 有特殊含义**，需转义，否则可能抛异常。

### 4. 大小写与空白处理

| 方法 | 说明 | 版本 |
|:---|:---|:---|
| `toUpperCase()` | 转大写 | 1.0 |
| `toLowerCase()` | 转小写 | 1.0 |
| `trim()` | 去首尾空白（**仅 ASCII** ≤ U+0020） | 1.0 |
| `strip()` | 去首尾空白（**Unicode** 感知） | **11+** |
| `stripLeading()` | 只去头部空白 | **11+** |
| `stripTrailing()` | 只去尾部空白 | **11+** |
| `indent(int n)` | 调整缩进 | **12+** |

```java
// trim() 和 strip() 对 Unicode 空白的行为差异
String s = "\u2000Hello\u2000";    // \u2000 是 Unicode 空格

s.trim().length();     // 7，trim 不认 \u2000，没去掉
s.strip().length();    // 5，strip 正确去掉了
```

> ⚠️ **`trim()` vs `strip()`**：
> - `trim()` 只处理 `\u0020` 及以下的字符，对全角空格（`\u3000`）、Unicode 空格（`\u2000`）**无效**。
> - `strip()` 基于 `Character.isWhitespace()`，**能处理所有 Unicode 空白**，是更正确的选择。
> - 处理**中文用户输入**（可能含全角空格）时，必须用 `strip()`。

### 5. 转换与格式化

| 方法 | 说明 | 版本 |
|:---|:---|:---|
| `String.valueOf(...)` | 基本类型/对象转字符串（**推荐，不会 NPE**） | 1.0 |
| `String.format(fmt, args)` | 格式化字符串 | 1.5 |
| `formatted(args)` | 实例方法版格式化 | **15+** |
| `getBytes()` | 转字节数组（用平台默认编码，**不推荐**） | 1.0 |
| `getBytes(Charset)` | 按指定编码转字节（**推荐**） | 1.6 |
| `toCharArray()` | 转 `char[]` | 1.0 |
| `intern()` | 放入字符串常量池并返回池中引用 | 1.0 |
| `chars()` | 转 `IntStream`（处理字符流） | **8+** |
| `codePoints()` | 转码点 `IntStream`（emoji 安全） | **8+** |
| `lines()` | 按行分割为 `Stream<String>` | **11+** |
| `transform(Function)` | 链式转换（函数式风格） | **12+** |

```java
// valueOf vs toString
Object obj = null;
String.valueOf(obj);     // "null"，安全
obj.toString();          // ❌ NullPointerException

// 格式化
String.format("姓名：%s，年龄：%d", "张三", 18);
"姓名：%s，年龄：%d".formatted("张三", 18);   // Java 15+ 更简洁

// 编码转换必须显式指定字符集
byte[] bytes = "中文".getBytes(StandardCharsets.UTF_8);   // ✅ 推荐
// "中文".getBytes();    // ❌ 依赖平台默认编码，跨平台可能乱码

// Stream 处理（Java 8+）
"Hello".chars().filter(Character::isUpperCase).count();    // 1

// 按行处理（Java 11+）
"a\nb\nc".lines().forEach(System.out::println);

// transform 链式调用（Java 12+）
String result = "  hello  "
        .transform(String::strip)
        .transform(String::toUpperCase);    // "HELLO"
```

> **重要提示**：`String.valueOf(obj)` 与 `obj.toString()` 的区别常被问到——前者对 `null` 返回字符串 `"null"` 而不抛异常，后者会 NPE。日志打印对象时应优先用 `String.valueOf()`。

### 6. 简单示例（综合）

```java
String s = " Hello World ";

s.length();                          // 13
s.trim();                            // "Hello World"
s.strip();                           // "Hello World"
s.contains("World");                 // true
s.indexOf("World");                  // 7
s.substring(1, 6);                   // "Hello"
s.replace("World", "Java");          // " Hello Java "
s.split(" ");                        // ["", "Hello", "World"]
String.join("-", "a", "b");          // "a-b"
String.format("%s-%d", "id", 1);     // "id-1"
```

### 7. 方法使用要点小结

| 要点 | 说明 |
|:---|:---|
| **`==` vs `equals()`** | `==` 比较引用地址，比较内容必须用 `equals()` |
| **频繁拼接用 Builder** | 循环中不要用 `+`，改用 `StringBuilder` / `StringBuffer` |
| **判空用 `isBlank()`** | 比 `isEmpty()` 更符合业务语义（Java 11+） |
| **去空白用 `strip()`** | 比 `trim()` 更通用，能处理 Unicode 空白（Java 11+） |
| **`matches()` 是完全匹配** | 判断包含用 `contains()` |
| **`indexOf` 找不到返回 -1** | 必须用 `!= -1` 判断 |
| **`split` 参数是正则** | 特殊字符需转义，末尾空串会被丢弃 |
| **编码要显式指定** | `getBytes(StandardCharsets.UTF_8)`，别用无参版本 |
| **`valueOf` 优于 `toString`** | 对 null 安全，不会 NPE |
| **所有方法都返回新对象** | String 不可变，调用后原对象不变 |

## 面试题

### 一、不可变性与设计

**1. String 为什么设计成不可变？**

四个方面：

- **线程安全**：不可变对象天然线程安全，无需同步。
- **可缓存 hash**：hash 值不变，计算一次可缓存，适合做 HashMap 的 key。
- **String Pool 的前提**：只有不可变，多个引用共享同一对象才安全。
- **安全性**：作为参数（如数据库 URL、文件路径、网络主机名）时不会被中途篡改。

**2. String 是 final 的，那不可变性只靠 final 保证吗？**

不是，`final` 只是保障之一。完整保障有三点：

1. 类被 `final` 修饰 → 防止子类重写方法破坏不可变
2. 内部数组 `private final` → 引用不可改
3. **不提供任何修改内部数组的方法** → 这是最关键的一点

仅有 `final` 保证不了不可变（`final` 只是引用不变，数组内容仍然可改）：

```java
final char[] arr = {'a', 'b'};
arr[0] = 'x';      // 合法！final 修饰的是引用，不是数组内容
```

**3. 既然 String 不可变，那 `s += "x"` 为什么能编译通过？**

因为这不是「修改」，而是**创建新对象并改变引用**：

```java
String s = "a";
s += "x";
// 等价于 s = new StringBuilder().append(s).append("x").toString();
// s 指向了新的 String 对象，原来的 "a" 没有被修改
```

**4. 为什么不直接用 `char[]` 而要用 String？**

- 安全性：`char[]` 可变，传递时可能被篡改
- 方便性：String 提供了大量 API 且重写了 `equals`/`hashCode`
- 性能：String 有常量池和 hash 缓存优化

> 反过来，**密码等敏感信息应该用 `char[]` 而非 String**——因为 String 不可变，密码会长期驻留在内存中（可能被 dump 到堆快照），且常量池中的副本不会被 GC 回收；而 `char[]` 用完可以立即清零。

### 二、常量池与内存

**5. `new String("abc")` 创建了几个对象？**

- 若常量池中**没有** `"abc"` → 创建 **2 个**（1 个池对象 + 1 个堆对象）
- 若常量池中**已有** `"abc"` → 创建 **1 个**（只创建堆对象）

**6. 下面的输出是什么？**

```java
String s1 = "abc";
String s2 = new String("abc");
String s3 = s2.intern();
System.out.println(s1 == s2);           // false
System.out.println(s1 == s3);           // true
System.out.println(s2 == s3);           // false
```

- `s1 == s2`：池对象 vs 堆对象，false
- `s1 == s3`：`intern()` 返回池中引用，与 s1 相同，true
- `s2 == s3`：堆对象 vs 池对象，false

**7. String Pool 在 JDK 各版本的位置？**

| 版本 | 位置 | 存储 |
|:---|:---|:---|
| JDK 6 | 永久代（方法区） | 对象本身 |
| JDK 7+ | **堆** | 引用 |

移到堆的原因：永久代空间有限，大量 `intern()` 容易 OOM。

**8. 下面两个分别创建几个对象？**

```java
String a = "a" + "b" + "c";    // 0 个（编译期折叠为 "abc"），或说最多 1 个池对象
String b = new String("abc");  // 最多 2 个
```

**9. `intern()` 有什么风险？**

- 常量池中的引用长期存活，不会被 GC（除非用 `-XX:+CMSClassUnloadingEnabled` 等）
- 大量**不同内容**的 `intern()` 会导致池持续膨胀 → 内存泄漏
- 仅适合「**大量重复**字符串」的去重场景

### 三、字符串操作与性能

**10. 为什么在循环中用 `+=` 拼接字符串性能差？**

因为每次 `+=` 都会在编译后变成 `new StringBuilder()` + `append` + `toString()`，即**每次循环都创建新对象**：

```java
// 10000 次循环 → 创建 10000 个 StringBuilder + 10000 个 String
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;
}
```

**正确做法**：循环外创建 `StringBuilder`，循环内 `append`。

**11. `final` 修饰的 String 拼接为什么更快？**

```java
final String a = "value";
String r = a + " abc";
// 编译期直接优化为：String r = "value abc";
```

因为 `final` 变量在编译期就是常量，拼接时直接替换为字面量值，**不创建 StringBuilder**。

**12. StringBuilder 和 StringBuffer 怎么选？**

- 单线程 → `StringBuilder`（无锁，更快）
- 多线程共享同一个 builder → `StringBuffer`
- 但更推荐的做法是：多线程下各用各的 `StringBuilder`（局部变量），避免锁竞争

> `StringBuilder` 是 JDK 5 引入的，作为 `StringBuffer` 的非同步版本。

**13. StringBuilder 的扩容机制？**

默认容量 16，不足时扩容为 `旧容量 * 2 + 2`，并做数组拷贝。能预估长度时，创建时指定容量可避免扩容开销。

**14. `String` 和 `StringBuilder` 的 `equals()` 行为一样吗？**

不一样，这是个常见陷阱：

```java
StringBuilder sb1 = new StringBuilder("abc");
StringBuilder sb2 = new StringBuilder("abc");
System.out.println(sb1.equals(sb2));   // false！

String s1 = "abc";
String s2 = "abc";
System.out.println(s1.equals(s2));     // true
```

**原因**：`String` 重写了 `equals()` 比较内容，而 **`StringBuilder` 没有重写**，用的是 `Object.equals()`（比较引用），所以两个内容相同的实例返回 false。

> 要比较 `StringBuilder` 的内容，需要转成 String：`sb1.toString().equals(sb2.toString())`。

### 四、编码与字符

**15. `char` 能存下所有字符吗？**

**不能**。`char` 是 16 位（2 字节），只能表示 **BMP（基本多语言平面）** 范围内的字符（U+0000 ~ U+FFFF）。

对于超出 BMP 的字符（如 emoji、部分生僻汉字），需要用**代理对（Surrogate Pair）**，即两个 `char` 表示一个字符：

```java
String emoji = "😀";
System.out.println(emoji.length());            // 2，占用两个 char
System.out.println(emoji.codePointCount(0, emoji.length()));  // 1，实际是一个字符
```

**实践建议**：处理可能含 emoji 的字符串时，用 `codePointCount()` 而非 `length()` 计算字符数。

**16. 字符常量 `'a'` 和字符串常量 `"a"` 的区别？**

| | `'a'` | `"a"` |
|:---|:---|:---|
| 类型 | `char`（基本类型） | `String`（引用类型） |
| 本质 | 16 位整型值（97） | 对象 |
| 内存 | 2 字节 | 对象头 + 数组等，远大于 2 字节 |
| 运算 | `'a' + 1 = 98` | `"a" + 1 = "a1"` |

**17. Java 9 为什么把 `char[]` 改成 `byte[]`？**

为了节省内存（JEP 254 紧凑字符串）：

- 纯 Latin-1 字符（英文、数字）→ `byte[]` 每字符 **1 字节**，`coder = LATIN1`
- 含其他字符（中文等）→ 仍用 **2 字节**，`coder = UTF16`

这样纯英文场景内存**直接减半**，而绝大多数字符串都是纯英文（URL、key、日志等）。

**18. 为什么密码要用 `char[]` 而不是 `String`？**

- `String` 不可变，密码会长期存在于内存中，直到被 GC 回收，期间可能被堆快照（heap dump）捕获
- `char[]` 是**可变**的，用完后可以**主动清零**，缩短敏感信息在内存中的存活时间

```java
char[] password = readPassword();
try {
    // 使用密码
} finally {
    Arrays.fill(password, '0');   // 主动清零
}
```

### 五、常用方法相关

**19. `trim()` 和 `strip()` 有什么区别？**

| 对比项 | `trim()` | `strip()` |
|:---|:---|:---|
| 版本 | 1.0 | **Java 11+** |
| 判断依据 | 字符 ≤ `U+0020`（ASCII） | `Character.isWhitespace()`（Unicode） |
| 全角空格 `\u3000` | ❌ 不去除 | ✅ 去除 |
| Unicode 空格 `\u2000` | ❌ 不去除 | ✅ 去除 |

```java
String s = "\u3000abc\u3000";   // 全角空格
s.trim().equals(s);    // true，trim 没去掉
s.strip().length();    // 3，strip 正确去除
```

**处理中文用户输入时应该用 `strip()`**——中文输入法容易打出全角空格。

**20. `isEmpty()` 和 `isBlank()` 有什么区别？**

- `isEmpty()`：长度是否为 0，即 `length() == 0`
- `isBlank()`：是否为空**或只含空白字符**（Java 11+）

```java
String s = "   ";
s.isEmpty();   // false，有三个字符
s.isBlank();   // true，业务上应视为空
```

> **实践建议**：校验用户输入非空，应该用 `isBlank()`。用 `isEmpty()` 会放过「只输入空格」的情况。

**21. `indexOf()` 找不到时返回什么？**

返回 **-1**（不是 0 也不是 null）。

```java
String s = "abc";
System.out.println(s.indexOf("z"));   // -1

// ❌ 错误写法
if (s.indexOf("z") != 0) { }   // 逻辑错误！-1 != 0 也是 true

// ✅ 正确写法
if (s.indexOf("z") != -1) { }
if (s.contains("z")) { }       // 更清晰
```

**22. `matches()` 和 `contains()` 有什么区别？**

- `matches(regex)`：**完全匹配**整个字符串
- `contains(s)`：判断是否**包含**子串

```java
"abc123".matches("\\d+");         // false，因为前面有字母
"abc123".contains("123");         // true

// 想用正则做「包含」判断，要自己加 .*
"abc123".matches(".*\\d+.*");     // true
```

> 这是很常见的认知错误——很多人以为 `matches` 是「包含」。

**23. `split()` 有什么坑？**

三个坑：

```java
// 坑 1：参数是正则，特殊字符要转义
"a.b.c".split(".");        // ❌ 结果为空数组（. 匹配任意字符）
"a.b.c".split("\\.");      // ✅ ["a", "b", "c"]

// 坑 2：末尾空串会被丢弃
"a,b,".split(",");         // ["a", "b"]，长度为 2
"a,b,".split(",", -1);     // ["a", "b", ""]，长度为 3，用 -1 保留空串

// 坑 3：split 性能一般，高频调用可考虑手写或用 StringUtils
```

**24. `String.valueOf()` 和 `toString()` 有什么区别？**

```java
Object obj = null;

String.valueOf(obj);    // "null"，安全返回字符串
obj.toString();         // ❌ NullPointerException

String.valueOf(123);    // "123"，支持基本类型
```

`String.valueOf()` 内部对 `null` 做了判断，**对 null 安全**。日志打印、字符串拼接场景推荐用它。

**25. `getBytes()` 为什么要指定字符集？**

```java
byte[] b1 = "中文".getBytes();                              // ❌ 用平台默认编码
byte[] b2 = "中文".getBytes(StandardCharsets.UTF_8);        // ✅ 明确指定
```

无参的 `getBytes()` 使用**平台默认字符集**——Windows 中文环境可能是 GBK，Linux 可能是 UTF-8。同一份代码在不同环境得到不同字节，导致**跨平台乱码**。

**必须显式指定字符集**，推荐 `StandardCharsets.UTF_8`。

**26. `length()`、`length`、`size()` 分别用在哪？**

| 写法 | 适用对象 | 说明 |
|:---|:---|:---|
| `length`（无括号） | **数组** | `arr.length` |
| `length()` | **String** | `str.length()` |
| `size()` | **集合**（List/Set/Map） | `list.size()` |

```java
int[] arr = {1, 2, 3};
arr.length;                  // 数组用 length

String s = "abc";
s.length();                  // String 用 length()

List<String> list = new ArrayList<>();
list.size();                 // 集合用 size()
```

> 这是 Java 中被吐槽最多的不一致设计之一，面试常考。

**27. 如何判断一个字符串是否包含 emoji 或统计真实字符数？**

```java
String emoji = "😀";

emoji.length();                                    // 2，代理对占两个 char
emoji.codePointCount(0, emoji.length());           // 1，真实字符数

// 判断是否含非 BMP 字符（如 emoji）
boolean hasEmoji = emoji.codePoints().anyMatch(cp -> cp > 0xFFFF);

// 按真实字符遍历
emoji.codePoints().forEach(cp -> 
    System.out.println(new String(Character.toChars(cp))));
```

> 涉及用户昵称、评论等可能含 emoji 的场景，**不能用 `length()` 判断长度**，否则会把一个 emoji 当两个字符，导致校验和截断出错。

# 参考文章
- https://mp.weixin.qq.com/s?__biz=MzI2OTQ4OTQ1NQ==&mid=2247483956&idx=1&sn=1c19164967621fa5449a7830d006c8f9&scene=19#wechat_redirect