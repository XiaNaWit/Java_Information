# Spring 事务

## Spring 支持两种方式的事务管理

### 1、编程式事务管理

- TransactionTemplate

```java
@Autowired
private TransactionTemplate transactionTemplate;

public void testTransaction(){
    transactionTemplate.execute(new TransactionCallbackWithoutResult(){
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus status){
                try{
                    //...业务代码
                }catch(Exception e){
                    status.setRollbackOnly();
                }
        }
    });
}
```

- TransactionManager

```java
    @Autowired
private PlatformTransactionManager transactionManager;

public void testTransaction2(){
        TransactionStatus status=transactionManager.getTransaction(new DefaultTransactionDefinition());
        try{
            //...业务代码
            transactionManager.commit(status);
        }catch(Exception e){
            transactionManager.rollback(status);
        }
}
```

### 2、注解

- @Transactional

## 事务的传播性 Propagation

① PROPAGATION_REQUIRED：如果当前没有事务，就创建一个新事务，如果当前存在事务，就加入该事务，该设置是最常用的设置。

② PROPAGATION_SUPPORTS：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行。

③ PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就抛出异常。

④ PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务。

⑤ PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

⑥ PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。

⑦ PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则按REQUIRED属性执行。

## 事务边界

**什么是事务边界**：一次事务从「开启（或加入已有事务）」到「提交/回滚」的范围，就是这个事务的边界。`@Transactional` 是 AOP 实现的，所以边界天然等于「**代理方法的一次调用**」：

```text
调用方 → 代理对象（开启事务）→ 目标方法执行 → 正常返回（提交）
                                        └→ 抛出可回滚异常（回滚）
```

几个关键认知：

- 边界由**最外层经过代理的方法**决定；同类内部方法调用不经过代理，不会产生新边界（也不会让注解生效）。
- 事务内所有通过代理进入、传播行为为 `REQUIRED`（默认）的方法都**加入**这个边界，任何一个抛出可回滚异常，整个边界内的操作一起回滚。
- 边界的位置决定事务粒度：粒度越大，锁和连接持有时间越长；粒度越小，越容易出现「部分成功」。

**边界画错的三种典型**

| 问题 | 现象 | 正确做法 |
|:---|:---|:---|
| 边界过小（拆太细） | 「创建订单」「扣库存」各自 `@Transactional`，被 Controller 依次调用，第二步失败时第一步已提交 | 在一个事务方法里编排两个 DAO，子方法用默认 `REQUIRED` 加入同一边界 |
| 边界过大 | 事务里包了 RPC、发 MQ、文件 IO、大批量循环、`Thread.sleep`，形成长事务：行锁/间隙锁长期不释放、连接池被占满、undo log 膨胀、主从延迟 | 远程调用、消息发送移到事务外；先远程预占 → 再本地事务 → 失败补偿；强一致用事务消息/本地消息表 |
| 边界位置不对 | 注解加在 Controller/DAO 上，或加在 `private`、同类自调用的方法上（根本没进代理，等于没加） | 放在 Service 对外业务方法上，以「一个业务用例」为粒度 |

```java
@Service
public class OrderService {
    @Autowired private StockService stockService;

    // 边界：整个「下单」用例；任一环节失败，订单与库存一起回滚
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(OrderCmd cmd) {
        orderDao.insert(cmd.toOrder());     // 同一事务
        stockDao.deduct(cmd.getSkuId());    // 同一事务
        stockService.recordLog(cmd);        // 跨 Bean 调用，REQUIRED 加入当前边界
    }
}

// 反例：把边界切在调用方，两步各自提交，第二步失败无法回滚第一步
orderService.createOrder(cmd);
stockService.deduct(skuId);
```

> 需要「事务提交后再做」的动作（发消息、清缓存、通知），不要直接写在事务里，用 `TransactionSynchronizationManager.registerSynchronization(...)` 的 `afterCommit` 回调，或使用 `@TransactionalEventListener(phase = AFTER_COMMIT)`。

## spring事务失效的场景

`@Transactional` 生效需要同时满足：**Bean 由 Spring 管理 + 调用经过代理 + 方法可被代理（public、非 final/static）+ 异常能触发回滚 + 有可用的事务管理器和支持事务的存储引擎**。缺一即失效或行为不符合预期。

### 1. 常见失效场景速查

| # | 场景 | 原因 | 解决 |
|:---|:---|:---|:---|
| 1 | `new` 出来的对象上调用事务方法 | 对象不在容器中，没有代理 | 用注入的 Bean，不要自己 `new` |
| 2 | 同类内部自调用（`this.a()`） | 未经过代理，注解被忽略 | 注入自身代理 / `AopContext.currentProxy()` / 拆分到另一个 Bean |
| 3 | 方法非 `public`（`private`/`protected`） | Spring 解析事务属性时只处理 `public` 方法，非 public 方法上即使有注解也会被跳过 | 改为 `public` |
| 4 | 方法/类是 `final`、`static` | CGLIB 无法继承重写、静态方法不属于对象 | 去掉 `final`/`static` |
| 5 | 受检异常未配置 `rollbackFor` | 默认只对 `RuntimeException` 和 `Error` 回滚 | `@Transactional(rollbackFor = Exception.class)` |
| 6 | 异常被 `catch` 吞掉 | 代理感知不到异常，正常提交 | catch 后重新抛出，或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| 7 | 传播行为不当 | `NOT_SUPPORTED`/`NEVER` 以非事务执行；`REQUIRES_NEW` 是独立边界 | 按业务选对 `propagation` |
| 8 | 表引擎不支持事务 | MyISAM 无事务 | 改用 InnoDB |
| 9 | 多数据源未指定事务管理器 | 用了默认管理器，操作的是另一个库 | `@Transactional(transactionManager = "xxxTxManager")` |
| 10 | 事务方法里开了新线程/线程池 | 事务上下文（连接）绑定在线程上，新线程不在事务内 | 事务结束后再异步，或把异步操作纳入独立事务 |
| 11 | `@Async` + `@Transactional` 混用 | 异步方法在别的线程执行，边界与调用方无关；代理叠加顺序还会导致注解丢失 | 异步方法内部自己开事务；不要指望调用方事务能覆盖它 |
| 12 | 切点/代理方式不匹配 | JDK 动态代理基于接口，接口方法未声明注解时可能拿不到注解 | 用 CGLIB（`proxyTargetClass = true`）或把注解写在接口上 |
| 13 | 未开启事务管理 | 未加 `@EnableTransactionManagement`（Spring Boot 自动配置已开） | 补上注解/自动配置 |

### 2. 重点：跨类调用会不会失效？

常见疑问：**A 类的 a() 加了 `@Transactional`，由 B 类调用 a()，事务会不会失效？**

**结论：只要 B 拿到的是 A 的代理对象（即 Spring 注入的 `aService`），a() 的事务就生效**。因为 B 调用的是代理对象的方法，代理在入口处开启事务。

```java
@Service
public class AService {
    @Transactional(rollbackFor = Exception.class)
    public void a() { ... }
}

@Service
public class BService {
    @Autowired private AService aService;   // 注入的是 A 的代理对象

    public void b() {
        aService.a();   // ✅ 走代理，事务生效
    }
}
```

需要区分「生效」和「边界合并」：默认传播行为 `REQUIRED` 下，如果 B.b() 自己也在事务里，A.a() 会**加入** B 的事务，二者是同一个边界——a() 内部抛异常会让整个事务回滚，B 捕获异常也无法"只回滚 a"。

以下情况跨类调用也会失效（或不符合预期）：

| 情形 | 说明 |
|:---|:---|
| `B` 里 `new AService()` | 拿到的是普通对象，没有代理 |
| 调用链是 `B → A.c() → this.a()` | `a()` 是自调用，未进代理（这是最常见坑） |
| A 是 `final` 类 / a 是 `final` 方法 | 无法生成代理 |
| A 通过 `@Bean` 方法内部 `new` 出来并直接持有原始引用 | 同上，取决于拿到的是否是代理 |
| `a()` 的传播行为是 `REQUIRES_NEW` | 事务生效，但**另开独立事务**：B 回滚不影响 a() 已提交的数据 |
| `a()` 被 `@Async` 标注 | 在另一线程执行，与 B 的事务无关，异常也不会回滚 B |

### 3. 反过来的情形：B 有注解、A 没有注解，A 抛异常会回滚吗？

常见疑问：**B.b() 上打了 `@Transactional`，B 里调用 A.a()，而 A.a() 上没有注解，A 中抛异常会导致回滚吗？**

**结论：会回滚。** 前提是异常最终**传播出 B 的事务边界**，且属于可回滚的异常类型（默认 `RuntimeException`/`Error`，或 B 上配了 `rollbackFor`）。

**原因**：事务是否纳管，取决于**数据库连接从哪里来**，而不是方法上有没有注解。B 的代理在进入方法时开启事务，并把 `Connection` 通过 `TransactionSynchronizationManager` 绑定到当前线程（`ThreadLocal`）；A.a() 虽然没有事务属性，但它运行在同一个线程里，A 中的 DAO 通过 `DataSourceUtils.getConnection()` 拿到的是**同一个已绑定连接、同一个事务**。A 抛出的异常一路上抛到 B 的代理，代理统一执行 rollback，A 已执行的 SQL 也会一并撤销。

```java
@Service
public class BService {
    @Autowired private AService aService;

    @Transactional(rollbackFor = Exception.class)   // 事务边界在这里
    public void b() {
        orderDao.insert(...);      // 同一事务，会回滚
        aService.a();              // 无注解，但共用同一连接
    }
}

@Service
public class AService {
    // 注意：这里没有 @Transactional
    public void a() {
        stockDao.deduct(...);      // b() 回滚时，这条也会回滚
        throw new RuntimeException("扣减失败");
    }
}
```

可以用一句话记忆：**没注解的方法不会「开启」事务，但会「加入」已有事务**。

这时不会回滚的情形（大多与注解无关，而是边界、连接、异常被拦截了）：

| 情形 | 结果 | 原因 |
|:---|:---|:---|
| B 里 `try { a() } catch (...) { }` 吞掉异常并正常返回 | 提交，A 的操作也不回滚 | 代理感知不到异常，除非调用 `setRollbackOnly()` |
| A 抛的是受检异常，B 未配 `rollbackFor` | 提交 | 默认只回滚 `RuntimeException`/`Error` |
| A 中自行 `DriverManager.getConnection` / 用另一个 `DataSource` / 自行 `new JdbcTemplate` 且未走 Spring 连接 | A 的操作不回滚 | 不在同一连接上，天然脱离当前事务 |
| A 中 `@Async` 或另起线程/线程池执行 | A 的操作不回滚 | 事务上下文绑定线程，新线程拿不到 |
| A 使用未与 Spring 事务集成的 `SqlSession`/ORM 会话 | A 的操作不回滚 | 会话没有交给 Spring 管理 |
| B 的传播行为是 `NOT_SUPPORTED`/`NEVER` | 都不回滚 | B 自身就以非事务方式执行 |
| B 的调用本身没走代理（自调用、`new` 出来的对象） | 都不回滚 | 根本没有开启事务 |

> 反过来还要注意「没注解」的真正风险：**A.a() 被独立调用（外层没有事务）时，它没有任何事务保障**——方法内多条 SQL 各自自动提交，中途失败会留下中间状态。若希望它在被单独调用时也有事务，就必须自己加 `@Transactional`（默认 `REQUIRED`，有外层事务时会自动加入，不会重复开启）。
>
> 在无注解的 A 中如果想强制回滚外层事务，可调用 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`；若当前不在事务中，会抛 `NoTransactionException`。

### 4. 自调用的正确处理方式

```java
@Service
public class MyService {
    @Autowired private MyService self;          // 注入自身代理（注意循环依赖，Spring Boot 2.6+ 默认禁止，需允许）

    public void outerMethod() {
        innerMethod();                                  // ❌ 未走代理，注解失效
        this.innerMethod();                             // ❌ 同上
        self.innerMethod();                             // ✅ 走代理
    }

    @Transactional(rollbackFor = Exception.class)
    public void innerMethod() { ... }
}
```

或者在启动类/配置类上开启 `@EnableAspectJAutoProxy(exposeProxy = true)` 后用 `((MyService) AopContext.currentProxy()).innerMethod()`；更推荐的做法是**把事务方法拆到另一个 Bean**，结构更清晰。

原写法 `this.innerMethod()` 不会生效，注意别被注释误导。

### 5. 异常与回滚规则

```java
@Transactional(rollbackFor = Exception.class)   // 受检异常也回滚
public void transactionalMethod() {
    try {
        // 可能抛异常的业务代码
    } catch (Exception e) {
        // ❌ 吞掉异常，事务会提交；需要标记回滚或重新抛出
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

### 6. 怎么确认事务真的生效

- 打印注入对象类型：`aService.getClass()` 是否形如 `AService$$EnhancerBySpringCGLIB$$...`，是则说明是代理；
- 打开事务日志：`logging.level.org.springframework.transaction=DEBUG`，正常会看到 `Creating new transaction` 或 `Participating in existing transaction`，以及 `Initiating transaction commit/rollback`；
- 断点查看 `DataSourceUtils.getConnection(dataSource)` 或 `TransactionSynchronizationManager.isActualTransactionActive()` 的返回值。


