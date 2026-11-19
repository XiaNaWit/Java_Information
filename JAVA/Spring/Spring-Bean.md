# Spring Bean

## Spring bean的生命周期

Spring Bean的生命周期分为四个阶段和多个扩展点。扩展点又可以分为影响多个Bean和影响单个Bean。整理如下：

四个阶段

- 实例化 Instantiation
  - Spring 容器根据 bean 的定义创建一个 bean 实例。这可以通过调用构造函数或使用工厂方法实现。
- 属性赋值 Populate
    - Spring 容器将配置的属性值注入到 bean 中，这个过程可以通过依赖注入来完成，包括简单类型属性和引用其他 bean。
- 初始化 Initialization
    - 在 bean 属性设置完成后，Spring 调用初始化方法，确保 bean 在使用之前处于有效状态，且具备运行所需的所有配置和资源。可以通过以下方式实现：
        - @PostConstruct 注解的方法。
        - 实现 InitializingBean 接口，重写 afterPropertiesSet() 方法。
        - 在配置文件中指定初始化方法。
- 销毁 Destruction
    - 当 bean 的生命周期结束时，Spring 容器负责调用销毁方法。可以通过以下方式实现：
        - @PreDestroy 注解的方法。
        - 实现 DisposableBean 接口，重写 destroy() 方法。
        - 在配置文件中指定销毁方法。

多个扩展点

- 影响多个Bean
    - BeanPostProcessor(作用于初始化阶段的前后)
    - InstantiationAwareBeanPostProcessor(作用于实例化阶段的前后)
- 影响单个Bean
    - Aware(Aware类型的接口的作用就是让我们能够拿到Spring容器中的一些资源)
        - Aware Group1
            - BeanNameAware
            - BeanClassLoaderAware
            - BeanFactoryAware
        - Aware Group2
            - EnvironmentAware
            - EmbeddedValueResolverAware(实现该接口能够获取Spring EL解析器，用户的自定义注解需要支持spel表达式的时候可以使用)
            - ApplicationContextAware(ResourceLoaderAware\ApplicationEventPublisherAware\MessageSourceAware)
    - 生命周期(实例化和属性赋值都是Spring帮助我们做的，能够自己实现的有初始化和销毁两个生命周期阶段)
        - InitializingBean
        - DisposableBean

## bean的作用域

Spring Bean 中所说的作用域，在配置文件中即是“scope”

在面向对象程序设计中作用域一般指对象或变量之间的可见范围。

而在Spring容器中是指其创建的Bean对象相对于其他Bean对象的请求可见范围。

在Spring 容器当中，一共提供了5种作用域类型，在配置文件中，通过属性scope来设置bean的作用域范围

### singleton

```xml

<bean id="userInfo" class="cn.lovepi.UserInfo" scope="singleton"></bean>
```

当Bean的作用域为singleton的时候,Spring容器中只会存在一个共享的Bean实例，所有对Bean的请求只要id与bean的定义相匹配，则只会返回bean的同一实例。单一实例会被存储在单例缓存中，为Spring的缺省作用域。

### prototype

```xml

<bean id="userInfo" class="cn.lovepi.UserInfo" scope=" prototype "></bean>
```

每次对该Bean请求的时候，Spring IoC都会创建一个新的作用域。

对于有状态的Bean应该使用prototype，对于无状态的Bean则使用singleton

### request

```xml

<bean id="userInfo" class="cn.lovepi.UserInfo" scope=" request "></bean>
```

Request作用域针对的是每次的Http请求，Spring容器会根据相关的Bean的

定义来创建一个全新的Bean实例。而且该Bean只在当前request内是有效的。

### session

```xml

<bean id="userInfo" class="cn.lovepi.UserInfo" scope=" session "></bean>
```

针对http session起作用，Spring容器会根据该Bean的定义来创建一个全新的Bean的实例。而且该Bean只在当前http session内是有效的。

### global session

```xml

<bean id="userInfo" class="cn.lovepi.UserInfo" scope="globalSession"></bean>
```

类似标准的http session作用域，不过仅仅在基于portlet的web应用当中才有意义。Portlet规范定义了全局的Session的概念。他被所有构成某个portlet外部应用中的各种不同的portlet所共享。在global
session作用域中所定义的bean被限定于全局的portlet session的生命周期范围之内。

## 循环依赖问题

### [三级缓存](#三级缓存)

Spring 解决循环依赖的核心就是提前暴露对象，而提前暴露的对象就是放置于第二级缓存中。下表是三级缓存的说明：

|名称|    描述|
| --- | --- |
|singletonObjects |    一级缓存，存放完整的 Bean。|
|earlySingletonObjects    |二级缓存，存放提前暴露的Bean，Bean 是不完整的，未完成属性注入和执行 init 方法。|
|singletonFactories    |三级缓存，存放的是 Bean 工厂，主要是生产 Bean，存放到二级缓存中。|

所有被Spring 管理的 Bean，最终都会存放在 singletonObjects 中，这里面存放的 Bean 是经历了所有生命周期的（除了销毁的生命周期），完整的，可以给用户使用的。

earlySingletonObjects 存放的是已经被实例化，但是还没有注入属性和执行 init 方法的 Bean。

singletonFactories 存放的是生产 Bean 的工厂。

Bean 都已经实例化了，为什么还需要一个生产 Bean 的工厂呢？这里实际上是跟 AOP 有关，如果项目中不需要为 Bean 进行代理，那么这个 Bean 工厂就会直接返回一开始实例化的对象，如果需要使用 AOP 进行代理，那么这个工厂就会发挥重要的作用了，这也是本文需要重点关注的问题之一。

### [三级缓存的作用](#三级缓存的作用)

- Spring 会先从一级缓存 singletonObjects 中尝试获取 Bean。
- 若是获取不到，而且对象正在建立中，就会尝试从二级缓存 earlySingletonObjects 中获取 Bean。
- 若还是获取不到，且允许从三级缓存 singletonFactories 中经过 singletonFactory 的 getObject() 方法获取 Bean 对象，就会尝试从三级缓存 singletonFactories 中获取 Bean。
- 若是在三级缓存中获取到了 Bean，会将该 Bean 存放到二级缓存中。

### [解决循环依赖](#解决循环依赖)
Spring 是如何通过上面介绍的三级缓存来解决循环依赖的呢？这里只用 A，B 形成的循环依赖来举例：

1. 实例化 A，此时 A 还未完成属性填充和初始化方法（@PostConstruct）的执行，A 只是一个半成品。
2. 为 A 创建一个 Bean 工厂，并放入到  singletonFactories 中。
3. 发现 A 需要注入 B 对象，但是一级、二级、三级缓存均为发现对象 B。
4. 实例化 B，此时 B 还未完成属性填充和初始化方法（@PostConstruct）的执行，B 只是一个半成品。
5. 为 B 创建一个 Bean 工厂，并放入到  singletonFactories 中。
6. 发现 B 需要注入 A 对象，此时在一级、二级未发现对象 A，但是在三级缓存中发现了对象 A，从三级缓存中得到对象 A，并将对象 A 放入二级缓存中，同时删除三级缓存中的对象 A。（注意，此时的 A 还是一个半成品，并没有完成属性填充和执行初始化方法）
7. 将对象 A 注入到对象 B 中。
8. 对象 B 完成属性填充，执行初始化方法，并放入到一级缓存中，同时删除二级缓存中的对象 B。（此时对象 B 已经是一个成品）
9. 对象 A 得到对象 B，将对象 B 注入到对象 A 中。（对象 A 得到的是一个完整的对象 B）
10. 对象 A 完成属性填充，执行初始化方法，并放入到一级缓存中，同时删除二级缓存中的对象 A。

### [为什么需要三级缓存，二级缓存能解决么](#为什么需要三级缓存，二级缓存能解决么)

[1、尝试使用两级缓存解决依赖冲突](#为什么需要三级缓存，二级缓存能解决么)

第三级缓存的目的是为了延迟代理对象的创建，因为如果没有依赖循环的话，那么就不需要为其提前创建代理，可以将它延迟到初始化完成之后再创建。

既然目的只是延迟的话，那么我们是不是可以不延迟创建，而是在实例化完成之后，就为其创建代理对象，这样我们就不需要第三级缓存了。因此，我们可以将 addSingletonFactory() 方法进行改造。

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");

    synchronized (this.singletonObjects) {
        // 判断一级缓存中不存在此对象
        if (!this.singletonObjects.containsKey(beanName)) { 
            // 直接从工厂中获取 Bean
            Object o = singletonFactory.getObject();

            // 添加至二级缓存中
            this.earlySingletonObjects.put(beanName, o);
            this.registeredSingletons.add(beanName);
        }
    }
}
```
这样的话，每次实例化完 Bean 之后就直接去创建代理对象，并添加到二级缓存中。

测试结果是完全正常的，Spring 的初始化时间应该也是不会有太大的影响，因为如果 Bean 本身不需要代理的话，是直接返回原始 Bean 的，并不需要走复杂的创建代理 Bean 的流程。

[2、三级缓存的意义](#为什么需要三级缓存，二级缓存能解决么)

测试证明，二级缓存也是可以解决循环依赖的。为什么 Spring 不选择二级缓存，而要额外多添加一层缓存，使用三级缓存呢？

如果 Spring 选择二级缓存来解决循环依赖的话，那么就意味着所有 Bean 都需要在实例化完成之后就立马为其创建代理，而 Spring 的设计原则是在 Bean 初始化完成之后才为其创建代理。

> 使用三级缓存而非二级缓存并不是因为只有三级缓存才能解决循环引用问题，其实二级缓存同样也能很好解决循环引用问题。使用三级而非二级缓存并非出于 IOC 的考虑，而是出于 AOP 的考虑，即若使用二级缓存，在 AOP 情形注入到其他 Bean的，不是最终的代理对象，而是原始对象。

## Spring框架中的单例bean是否线程安全

Spring框架中的单例bean是线程安全的吗？它是如何处理线程并发问题的?

不是，Spring框架中的单例bean不是线程安全的。

spring 中的 bean 默认是单例模式，spring 框架并没有对单例 bean 进行多线程的封装处理。实际上大部分 spring bean 是无状态的（比如 dao 类），在某种程度上来说 bean 也是安全的，但如果 bean
有状态的话（比如 view model ）就要开发者自己去保证线程安全了，最简单的就是改变 bean 的作用域，把“singleton”变更为“prototype”，这样请求 bean 相当于 new Bean()了， 保证线程安全了。

- 有状态就是有数据存储功能。
- 无状态就是不会保存数据。

Spring如何处理线程并发问题?

一般只有无状态的Bean才可以在多线程下共享，大部分是无状态的Bean。当存有状态的Bean的时候，spring一般是使用ThreadLocal进行处理，解决线程安全问题。

ThreadLocal和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。 同步机制采用了“时间换空间”的方式，仅提供一份变量，不同的线程获取锁，没获得锁的线程则需要排队。而ThreadLocal采用了“空间换时间”的方式。
ThreadLocal会为每一个线程提供一个独立的变量副本，从而隔离了多个线程对数据的访问冲突。因为每一个线程都拥有自己的变量副本，所以没有相同变量的访问冲突问题。所以在编写多线程代码时，可以把不安全的变量封装进ThreadLocal。

