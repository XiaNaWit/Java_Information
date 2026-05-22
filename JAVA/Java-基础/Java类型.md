# Java 类型

## 基本类型

| 类型 | 大小 | 默认值 | 备注 |
|:---|:---:|:---:|:---|
| byte | 1字节 | 0 | -128 ~ 127 |
| short | 2字节 | 0 | -32768 ~ 32767 |
| int | 4字节 | 0 | Java 默认整数类型 |
| long | 8字节 | 0L | 赋值需加 L |
| float | 4字节 | 0.0f | 赋值需加 f |
| double | 8字节 | 0.0d | Java 默认小数类型 |
| boolean | JVM 实现决定 | false | JVM 中常用 int 表示 |
| char | 2字节（Unicode） | '\u0000' | 0~65535 |

> `boolean` 在 JVM 中通常用 `int`（4字节）表示，boolean 数组用 `byte[]`。

### float / double 判等

用误差范围比较，而非 `==`：

```java
// float: Math.abs(a - b) < 1e-6
// double: Math.abs(a - b) < 1e-15
```

高精度要求使用 `BigDecimal`。

## 包装类型

### 缓存范围

| 类型 | 缓存范围 |
|:---|:---|
| Integer | [-128, 127]（可通过 JVM 参数调整上限） |
| Short | [-128, 127] |
| Byte | [-128, 127]（全部缓存） |
| Long | [-128, 127] |
| Float / Double | **无缓存**（浮点数） |
| Boolean | TRUE / FALSE（全部缓存） |
| Character | [0, 127] |

### 自动装箱与拆箱

```java
Integer i = 10;                      // 装箱：Integer.valueOf(10)
int n = i;                           // 拆箱：i.intValue()
```

**陷阱**：包装类型参与算术运算时会自动拆箱，值为 null 时抛出 NPE。

```java
Integer a = null;
int b = a + 1;  // NullPointerException
```

### new Integer() vs Integer.valueOf()

```java
Integer a = new Integer(100);    // 始终创建新对象
Integer b = Integer.valueOf(100);  // 使用缓存
Integer c = 100;                   // 等价于 valueOf，使用缓存
```

`valueOf()` 在缓存范围内不创建新对象：

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

### 包装类型判等

包装类型用 `==` 比较的是引用地址，建议用 `equals()` 比较值。注意缓存范围内的 `==` 可能返回 true，但这是依赖实现的，不应靠此编程。

## BigDecimal

用于高精度浮点运算，推荐用 `String` 构造以避免精度丢失：

```java
BigDecimal a = new BigDecimal("0.1");   // 正确
BigDecimal b = new BigDecimal(0.1);     // 错误！0.1 无法精确表示为 double
```
