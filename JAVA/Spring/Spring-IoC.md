# Spring IoC

## IOC

### IOC是什么？

控制反转即IoC (Inversion of Control)，它把传统上由程序代码直接操控的对象的调用权交给容器，通过容器来实现对象组件的装配和管理。所谓的“控制反转”概念就是对组件对象控制权的转移，从程序代码本身转移到了外部容器。

Spring IOC 负责创建对象，管理对象（通过依赖注入（DI），装配对象，配置对象，并且管理这些对象的整个生命周期。

### 使用IOC的好处

- 不用自己组装，拿来就用。
- 享受单例的好处，效率高，不浪费空间
- 便于单元测试，方便切换mock组件
- 便于进行AOP操作，对于使用者是透明的
- 统一配置，便于修改

## BeanFactory 和 ApplicationContext有什么区别

BeanFactory和ApplicationContext是Spring的两大核心接口，都可以当做Spring的容器。其中ApplicationContext是BeanFactory的子接口。

### 依赖关系

BeanFactory：是Spring里面最底层的接口，包含了各种Bean的定义，读取bean配置文档，管理bean的加载、实例化，控制bean的生命周期，维护bean之间的依赖关系。

ApplicationContext接口作为BeanFactory的派生，除了提供BeanFactory所具有的功能外，还提供了更完整的框架功能：

- 继承MessageSource，因此支持国际化。
- 统一的资源文件访问方式。
- 提供在监听器中注册bean的事件。
- 同时加载多个配置文件。
- 载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层。

### 加载方式

BeanFactroy采用的是延迟加载形式来注入Bean的，即只有在使用到某个Bean时(调用getBean())
，才对该Bean进行加载实例化。这样，我们就不能发现一些存在的Spring的配置问题。如果Bean的某一个属性没有注入，BeanFacotry加载后，直至第一次使用调用getBean方法才会抛出异常。

ApplicationContext，它是在容器启动时，一次性创建了所有的Bean。这样，在容器启动时，我们就可以发现Spring中存在的配置错误，这样有利于检查所依赖属性是否注入。
ApplicationContext启动后预载入所有的单实例Bean，通过预载入单实例bean ,确保当你需要的时候，你就不用等待，因为它们已经创建好了。

相对于基本的BeanFactory，ApplicationContext 唯一的不足是占用内存空间。当应用程序配置Bean较多时，程序启动较慢。

### 创建方式

BeanFactory通常以编程的方式被创建，ApplicationContext还能以声明的方式创建，如使用ContextLoader。

### 注册方式

BeanFactory和ApplicationContext都支持BeanPostProcessor、BeanFactoryPostProcessor的使用，但两者之间的区别是：BeanFactory需要手动注册，而ApplicationContext则是自动注册。

## Spring IoC的初始化过程

### IOC粗略总结

1. 首先入口是xml或者注解或者其他形式，要实现beanDefinationReader接口，然后 读取的时候，会将他们解析为bean的定义信息；
2. beanFactory在加载bean信息实例化（底层用的反射）之前，spring加了一个接口beanFactoryPostProcessor,用作扩展用。
3. beanFactory内部实例化bean之后，在要初始化bean对象之前，增加了一个一系列aware接口，将他的容器，以及工厂都暴露出来供使用者做扩展用。
4. 经过一些列容器以及容器对象的注入之后，在初始化之前，spring又增加了一个接口 beanPostProcessor,该接口是可以作用于所有创建的bean,在初始化前后，均能通过重写该接口获取bean对象进行定制操作。
5. 经过实现初始化接口完成初始化功能。
6. 经过实现销毁接口disposableBean结束其生命。

### 重要的组件

- `BeanDefinition` 描述bean的属性的接口，例如bean的scope是单例还是多例，构造方法，有哪些property value，依赖等，相当于对这个bean的一份身份描述
    - Bean配置 --> BeanDefinition --> Bean对象
    -
    懒加载情况下，refresh只是把BeanDefinition注册到BeanFactory中，而不是把Bean注册到BeanFactory中。在调用上下文的getBean的时候才会去根据BeanDefinition生成具体的bean对象
- `BeanDefinitionMap`
- `BeanFactory`
    - spring的基础bean容器
    - 相当于存放所有bean的容器
    - 主要负责管理和提供各种 bean 的生命周期
- `ApplicationContext`
    - BeanFactory 的子接口，在 BeanFactory 的基础上构建，是相对比较高级的 IoC 容器实现。包含 BeanFactory
      的所有功能，还提供了其他高级的特性，比如：事件发布、国际化信息支持、统一资源加载策略等。正常情况下，我们都是使用的 ApplicationContext
    - 相当于丰富了beanfactory的功能，这里理解为上下文就好
- `FactoryBean`
    - FactoryBean 是 Spring 提供的一个接口，允许开发者通过实现该接口来创建复杂的 bean 对象。
    - 用于自定义对象的创建过程
### 源码解析

首先抛开其他组件的启动，我们只需要引入spring-context就可以启动一个容器了

```xml

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
</dependency>
```

而在springboot出来之前最常见的加载bean的方式是读取配置文件

```java
public static void main(String[]args){
        ApplicationContext context=new ClassPathXmlApplicationContext("classpath:applicationfile.xml");
}
```

这里ApplicationContext是一个接口，主要的实现类有：

- ClassPathXmlApplicationContext 需要一个 xml 配置文件在系统中的路径
- FileSystemXmlApplicationContext 需要一个 xml 配置文件在系统中的路径
- AnnotationConfigApplicationContext 基于注解，大势所趋

下面的分析都基于 ClassPathXmlApplicationContext 进行分析，因为比较好理解点

在 resources 目录新建一个配置文件，文件名随意，通常叫 application.xml 或 application-xxx.xml就可以了,对应的类实现一个：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"
       default-autowire="byName">

    <bean id="messageService" class="com.javadoop.example.MessageServiceImpl"/>
</beans>
```

main

```java
public class App {
    public static void main(String[] args) {
        // 用我们的配置文件来启动一个 ApplicationContext
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");

        System.out.println("context 启动成功");

        // 从 context 中取出我们的 Bean，而不是用 new MessageServiceImpl() 这种方式
        MessageService messageService = context.getBean(MessageService.class);
        // 这句将输出: hello world
        System.out.println(messageService.getMessage());
    }
}
```

在构造方法中的refresh()方法是启动加载整个容器的关键方法

方法在springboot容器启动时也会加载,方法为

- org.springframework.boot.SpringApplication#run
- org.springframework.boot.SpringApplication#refreshContext

```java
@Override
public void refresh()throws BeansException,IllegalStateException{
    // 1. 首先是一个synchronized加锁，当然要加锁，不然你先调一次refresh()然后这次还没处理完又调一次，就会乱套了；
    synchronized (this.startupShutdownMonitor){
        // 2. 这个方法是做准备工作的，记录容器的启动时间、标记“已启动”状态、处理配置文件中的占位符，可以点进去看看，这里就不多说了。
        prepareRefresh();

        // 3. 这个就很重要了，这一步是把配置文件解析成一个个Bean，并且注册到BeanFactory中，注意这里只是注册进去，并没有初始化。先继续往下看，等会展开这个方法详细解读
        ConfigurableListableBeanFactory beanFactory=obtainFreshBeanFactory();

        //4. 这个方法的作用是：设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean，这里都是spring里面的特殊处理，然后继续往下看
        prepareBeanFactory(beanFactory);

        try{
            // 5. 方法是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化，具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类，来完成一些其他的操作。
            postProcessBeanFactory(beanFactory);
    
            // 6. 接下来是这个方法是调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法；
            invokeBeanFactoryPostProcessors(beanFactory);
    
            // 7. 然后这个方法注册 BeanPostProcessor 的实现类，和上面的BeanFactoryPostProcessor 是有区别的，这个方法调用的其实是PostProcessorRegistrationDelegate类的registerBeanPostProcessors方法；
            // 这个类里面有个内部类BeanPostProcessorChecker，BeanPostProcessorChecker里面有两个方法postProcessBeforeInitialization和postProcessAfterInitialization，这两个方法分别在 Bean 初始化之前和初始化之后得到执行。
            // 然后回到refresh()方法中继续往下看
            registerBeanPostProcessors(beanFactory);
    
            // 8. 方法是初始化当前 ApplicationContext 的 MessageSource，国际化处理，继续往下
            initMessageSource();
    
            // 9. 方法初始化当前 ApplicationContext 的事件广播器继续往下
            initApplicationEventMulticaster();
    
            // 10. 方法初始化一些特殊的 Bean（在初始化 singleton beans 之前）；继续往下
            onRefresh();
    
            // 11. 方法注册事件监听器，监听器需要实现 ApplicationListener 接口；继续往下
            registerListeners();
    
            // 12. 重点到了 初始化所有的 singleton beans（单例bean），懒加载（non-lazy-init）的除外，这个方法也是等会细说
            finishBeanFactoryInitialization(beanFactory);
    
            // 13. 方法是最后一步，广播事件，ApplicationContext 初始化完成
            finishRefresh();
        } catch(BeansException ex){
            if(logger.isWarnEnabled()){
                logger.warn("Exception encountered during context initialization - "+"cancelling refresh attempt: "+ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
            destroyBeans();
    
            // Reset 'active' flag.
            cancelRefresh(ex);
    
            // Propagate exception to caller.
            throw ex;
        } finally{
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

## @Autowired原理
@Autowired 是 Spring 框架中用于自动装配依赖的一种注解。它简化了依赖注入的过程，允许 Spring 通过类型或名称自动查找并注入所需的 Bean

1. 组件扫描：

- Spring 启动时，会根据配置（如 @ComponentScan）扫描指定的包，查找被 @Component、@Service、@Repository、@Controller 等注解标记的类，并将其注册为 Spring 的 Bean。

2. 依赖解析：

- 当 Spring 容器创建一个 Bean 时，如果该 Bean 的属性标注了 @Autowired，Spring 将对这些属性进行依赖解析。
- Spring 会查找容器中匹配的 Bean，依据优先顺序：
    - 根据类型匹配（默认方式）。
    - 如果有多个同类型的 Bean，可以使用 @Qualifier 指定具体的 Bean 名称。

3. 注入：

一旦找到匹配的 Bean，Spring 会通过反射将其注入到标注了 @Autowired 的属性中。

4. 生命周期管理：

Spring 还会管理这些 Bean 的生命周期，包括初始化和销毁等操作。

## 使用@Autowired注解自动装配的过程是怎样的？

使用@Autowired注解来自动装配指定的bean。在使用@Autowired注解之前需要在Spring配置文件进行配置，<context:annotation-config />。

在启动spring
IoC时，容器自动装载了一个AutowiredAnnotationBeanPostProcessor后置处理器，当容器扫描到@Autowied、@Resource或@Inject时，就会在IoC容器自动查找需要的bean，并装配给该对象的属性。在使用@Autowired时，首先在容器中查询对应类型的bean：

- 如果查询结果刚好为一个，就将该bean装配给@Autowired指定的数据；
- 如果查询的结果不止一个，那么@Autowired会根据名称来查找；
- 如果上述查找的结果为空，那么会抛出异常。解决方法是，使用required=false。

## @Autowired和@Resource之间的区别

@Autowired可用于：构造函数、成员变量、Setter方法

@Autowired和@Resource之间的区别

- @Autowired默认是按照类型装配注入的，默认情况下它要求依赖对象必须存在（可以设置它required属性为false）。
- @Resource默认是按照名称来装配注入的，只有当找不到与名称匹配的bean才会按照类型来装配注入。

## Spring中BeanFactory与FactoryBean的区别

### BeanFactory

BeanFactory是一个接口，它是Spring中工厂的顶层规范，是SpringIoc容器的核心接口，它定义了getBean()、containsBean()等管理Bean的通用方法。Spring的容器都是它的具体实现如：

- DefaultListableBeanFactory
- XmlBeanFactory
- ApplicationContext

这些实现类又从不同的维度分别有不同的扩展。

### FactoryBean

首先它是一个Bean，但又不仅仅是一个Bean。它是一个能生产或修饰对象生成的工厂Bean，类似于设计模式中的工厂模式和装饰器模式。它能在需要的时候生产一个对象，且不仅仅限于它自身，它能返回任何Bean的实例。

FactoryBean表现的是一个工厂的职责。 即一个Bean A如果实现了FactoryBean接口，那么A就变成了一个工厂，根据A的名称获取到的实际上是工厂调用getObject()
返回的对象，而不是A本身，如果要获取工厂A自身的实例，那么需要在名称前面加上'&'符号。

- getObject('name')返回工厂中的实例
- getObject('&name')返回工厂本身的实例

