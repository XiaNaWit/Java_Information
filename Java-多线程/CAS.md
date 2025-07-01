

# 如何再不使用synchronized关键字的情况下保证线程安全
- ### 一、原子变量（Atomic Variables）：

  通过java.util.concurrent.atomic包中的原子类（如AtomicInteger、AtomicReference）实现无锁线程安全。这些类利用CAS（Compare-and-Swap）指令保证操作的原子性，例如：
    ```java
    private AtomicInteger counter = new AtomicInteger(0);
    counter.incrementAndGet();  // 原子递增操作
    ```
    其优势为：高性能，避免锁竞争；但是不适用于复杂操作的原子性保证

- ### 二、​​不可变对象（Immutable Objects）​：

  通过设计不可变类（如String、自定义final类）避免共享状态被修改。不可变对象天然线程安全，因为所有字段在构造后无法变更。示例如下：
  ```java
  public final class ImmutablePoint {
    private final int x;
    private final int y;
    public ImmutablePoint(int x, int y) { this.x = x; this.y = y; }
    // 无setter方法
    }
  ```
  ​​适用场景​​：配置参数、数据传输对象（DTO）

- ### 三、​​线程本地存储（ThreadLocal）​：

  通过ThreadLocal类为每个线程创建变量副本，避免共享状态，如：
  ```java
  private static ThreadLocal<SimpleDateFormat> dateFormat = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
  ```
  优势​​：适用于线程上下文隔离（如用户会话）；​​注意​​：需手动清理以避免内存泄漏

- ### 四、​​CAS（无锁算法 Compare-and-Swap)）​：

    使用AtomicReference等类手动实现CAS逻辑，例如线程安全单例：
    ```java
    public class Singleton {
        private static final AtomicReference<Singleton> INSTANCE = new AtomicReference<>();
        private Singleton() {}
        public static Singleton getInstance() {
            while (true) {
                Singleton instance = INSTANCE.get();
                if (instance != null) return instance;
                instance = new Singleton();
                if (INSTANCE.compareAndSet(null, instance)) return instance;
            }
        }
    }
    ```
    
    #### 原理
    - 实际上是使用了unsafe的CAS操作
    - 用期望值去与内存地址的值判断是否相等，相等则用更新值替换，否则CAS等待
    #### 参数
    compareAndSwapInt
    - this 对象
    - Offset 内存地址
    - expect 旧的期望值
    - update 新的值

    #### CAS的问题
    CAS操作是一种无锁算法，相对于使用锁机制，它能够更好地利用多核处理器的优势，提高并发性能。但是，CAS操作也存在一些问题，包括：

    - `ABA问题`：在CAS操作中，如果内存位置的当前值与预期原值相等，就会执行修改操作。但是，如果在修改之前，有其他线程对内存位置的值进行了两次修改，从而又恢复为了原来的值，此时CAS操作可能会认为没有被修改过，从而执行了修改操作。这种情况被称为ABA问题。

    - `自旋次数问题`：如果CAS操作失败，需要重试，这时会进入自旋状态，不断地进行CAS操作，直到操作成功为止。如果自旋次数太多，会浪费大量的CPU资源，降低系统的性能。

    - `只能保证单个变量的原子操作`：CAS操作只能保证单个变量的原子操作，对于多个变量的操作，需要加锁才能保证原子性。

    针对上述问题，可以采取一些措施来解决。例如，

    - 针对ABA问题，可以使用带有版本号的CAS操作，从而保证在执行修改操作前，内存位置的值没有被修改过；
    - 针对自旋次数问题，可以设置一个自旋次数的上限，超过上限时放弃自旋，转为使用锁机制；
    - 针对多个变量的原子操作，可以使用分段锁等方式来保证原子性。

- ### 五、​​并发工具类​

    利用java.util.concurrent包中的线程安全容器和工具：
​    
    - ​并发集合​​：如ConcurrentHashMap（分段锁）、CopyOnWriteArrayList（写时复制），适用于高并发读写场景。
    
    - ​同步工具​​：如CountDownLatch（计数器等待）、Semaphore（资源许可控制），简化多线程协作逻辑。



