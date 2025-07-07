# String

## String的特性
* 不变性：String 是只读字符串，是一个典型的 immutable 对象，对它进行任何操作，其实都是创建一个新的对象，再把引用指向该对象。不变模式的主要作用在于当一个对象需要被多线程共享并频繁访问时，可以保证数据的一致性；

* 常量池优化：String 对象创建之后，会在字符串常量池中进行缓存，如果下次创建同样的对象时，会直接返回缓存的引用；

* final：使用 final 来定义 String 类，表示 String 类不能被继承，提高了系统的安全性。

## 字符型常量和字符串常量的区别？

1. 形式上: 字符常量是单引号引起的一个字符，字符串常量是双引号引起的若干个字符；

2. 含义上: 字符常量相当于一个整型值( ASCII 值),可以参加表达式运算；字符串常量代表一个地址值(该字符串在内存中存放位置，相当于对象；

3. 占内存大小：字符常量只占2个字节；字符串常量占若干个字节(至少一个字符结束标志) (注意: char 在Java中占两个字节)。

## 储存数据
- Java8内部使用char数组储存数据
  - `private final char value[];`
- Java9后使用byte数组同时使用coder来标识使用哪种编码
  - `private final byte[] value;`
  - `private final byte coder;`
## string pool

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

## final
### 不可变的原因
在Java中，String类型被设计成final的主要原因是为了保证字符串的不可变性。这意味着一旦一个字符串被创建，它的值就不能再被改变。这种不可变性带来了很多好处，例如：

- 线程安全：由于字符串是不可变的，所以在多线程环境下，不需要担心并发访问的问题。

- 缓存哈希值：由于字符串的哈希值是不可变的，所以可以在哈希表等数据结构中缓存字符串的哈希值，提高性能。

- 安全性：不可变的字符串可以防止在处理字符串时，因为修改字符串而导致的潜在安全问题，例如SQL注入攻击。

- 简化代码：由于字符串是不可变的，所以在编写代码时，不需要担心字符串值被改变的情况，这使得代码更加简单和可读。

此外，String类型作为Java中最常用的类型之一，设计成final还可以提高字符串的性能，因为编译器可以对字符串的操作进行一些优化，例如重复使用相同的字符串对象、避免创建新的字符串对象等。

### 在使用 HashMap 的时候，用 String 做 key 有什么好处？

HashMap 内部实现是通过 key 的 hashcode 来确定 value 的存储位置，因为字符串是不可变的，所以当创建字符串时，它的 hashcode 被缓存下来，不需要再次计算，所以相比于其他对象更快。


## StringBuilder
- 可变 byte数组没有被final修饰
- 线程不安全
- 速度快
## StringBuffer
- 可变 byte数组没有被final修饰
- 线程安全
  - 方法被synchronized修饰
- 速度慢

大量字符串操作使用他们
  - 操作少量的数据: 适用 String
  - 单线程操作字符串缓冲区下操作大量数据: 适用 StringBuilder
  - 多线程操作字符串缓冲区下操作大量数据: 适用 StringBuffer

## String 的hashcode()方法
String也是遵守equals的标准的，也就是 s.equals(s1)为true，则s.hashCode()==s1.hashCode()也为true。此处并不关注eqauls方法，而是讲解 hashCode()方法，String.hashCode()有点意思，而且在面试中也可能被问到。先来看一下代码：
```java
public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;
            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```
### 为什么要选31作为乘数呢？
从网上的资料来看，一般有如下两个原因：

- 31是一个不大不小的质数，是作为 hashCode 乘子的优选质数之一。另外一些相近的质数，比如37、41、43等等，也都是不错的选择。那么为啥偏偏选中了31呢？请看第二个原因。
- 31可以被 JVM 优化，31 * i = (i << 5) - i。

# 参考文章
- https://mp.weixin.qq.com/s?__biz=MzI2OTQ4OTQ1NQ==&mid=2247483956&idx=1&sn=1c19164967621fa5449a7830d006c8f9&scene=19#wechat_redirect