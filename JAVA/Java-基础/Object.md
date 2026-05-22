# Object

Object 是所有类的根类。

## equals

- 基本类型：`==` 比较值
- 引用类型：`==` 比较引用地址，`equals()` 比较等价性
- Object 默认的 `equals()` 与 `==` 相同，子类通常需重写

## hashCode

- 等价的对象散列值一定相同，散列值相同的对象不一定等价
- 重写 `equals()` **必须**重写 `hashCode()`，否则无法与散列集合（HashMap、HashSet 等）正常运作
- 重写 `hashCode()` 的通用约定：`equals()` 认定为相同的对象必须返回相同的哈希值

### 重写 equals 的一般步骤

1. 检查是否为同一个对象引用（`==`）→ true 则返回 true
2. 检查传入对象是否为 null → true 则返回 false
3. 检查是否属于同一类（`getClass()` 或 `instanceof`）→ false 则返回 false
4. 转型后比较关键字段

## toString

返回对象的字符串表示，默认格式为 `类名@哈希值的十六进制`，建议重写。

## clone

| 拷贝方式 | 行为 |
|:---|:---|
| **浅拷贝** | 拷贝基本类型值，引用类型只拷贝引用（新旧对象共享引用对象） |
| **深拷贝** | 拷贝基本类型值，引用类型创建新对象 |

实现克隆需实现 `Cloneable` 标记接口（否则调用 `super.clone()` 会抛 `CloneNotSupportedException`）。

```java
@Override
protected Object clone() throws CloneNotSupportedException {
    return super.clone();  // 浅拷贝
}
```

## wait / notify / notifyAll

必须在 `synchronized` 块中调用，否则抛 `IllegalMonitorStateException`。

| 方法 | 说明 |
|:---|:---|
| `wait()` | 释放锁并挂起当前线程，等待被唤醒 |
| `wait(long timeout)` | 超时等待（毫秒） |
| `notify()` | 唤醒一个等待该对象的线程 |
| `notifyAll()` | 唤醒所有等待该对象的线程 |

## finalize

GC 回收对象前调用，**Java 9 起已弃用**。不应依赖它释放资源，应使用 `try-with-resources` 或 `Cleaner`。

## 为什么重写 equals 必须重写 hashCode

如果不重写 hashCode，两个 `equals()` 相等的对象可能具有不同的哈希值，导致 HashMap 等散列集合将这两个对象放在不同的桶中，从而出现逻辑错误：

```java
// 假设仅重写 equals 未重写 hashCode
Map<Person, String> map = new HashMap<>();
map.put(new Person("Tom"), "value1");
map.get(new Person("Tom"));   // → null（因为 hashCode 不同，定位到不同桶）
```
