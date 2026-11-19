# Spring AOP

## 什么是 AOP

OOP(Object-Oriented Programming)面向对象编程，允许开发者定义纵向的关系，但并适用于定义横向的关系，导致了大量代码的重复，而不利于各个模块的重用。

AOP(Aspect-Oriented Programming)
，一般称为面向切面编程，作为面向对象的一种补充，用于将那些与业务无关，但却对多个对象产生影响的公共行为和逻辑，抽取并封装为一个可重用的模块，这个模块被命名为“切面”（Aspect），减少系统中的重复代码，降低了模块间的耦合度，同时提高了系统的可维护性。可用于权限认证、日志、事务处理等。

## AOP原理

原理是在IOC过程中，创建bean实例时，最后都会对bean进行处理来实现增强，对于AOP来说就是创建代理类

- 底层是动态代理技术
    - JDK动态代理(基于接口)
    - CGLib动态代理(基于类)
    - 在Spring AOP中，如果使用的是单例，推荐使用CGLib代理

## JDK动态代理

(一）实现原理

JDK的动态代理是基于反射实现。JDK通过反射，生成一个代理类，这个代理类实现了原来那个类的全部接口，并对接口中定义的所有方法进行了代理。当我们通过代理对象执行原来那个类的方法时，代理类底层会通过反射机制，回调我们实现的InvocationHandler接口的invoke方法。并且这个代理类是Proxy类的子类（记住这个结论，后面测试要用）。这就是JDK动态代理大致的实现方式。

（二）优点

JDK动态代理是JDK原生的，不需要任何依赖即可使用；

通过反射机制生成代理类的速度要比CGLib操作字节码生成代理类的速度更快；

（三）缺点

如果要使用JDK动态代理，被代理的类必须实现了接口，否则无法代理；

JDK动态代理无法为没有在接口中定义的方法实现代理，假设我们有一个实现了接口的类，我们为它的一个不属于接口中的方法配置了切面，Spring仍然会使用JDK的动态代理，但是由于配置了切面的方法不属于接口，为这个方法配置的切面将不会被织入。

JDK动态代理执行代理方法时，需要通过反射机制进行回调，此时方法执行的效率比较低；

## CGLib动态代理

（一）实现原理

CGLib实现动态代理的原理是，底层采用了ASM字节码生成框架，直接对需要代理的类的字节码进行操作，生成这个类的一个子类，并重写了类的所有可以重写的方法，在重写的过程中，将我们定义的额外的逻辑（简单理解为Spring中的切面）织入到方法中，对方法进行了增强。而通过字节码操作生成的代理类，和我们自己编写并编译后的类没有太大区别。

（二）优点

使用CGLib代理的类，不需要实现接口，因为CGLib生成的代理类是直接继承自需要被代理的类；

CGLib生成的代理类是原来那个类的子类，这就意味着这个代理类可以为原来那个类中，所有能够被子类重写的方法进行代理；

CGLib生成的代理类，和我们自己编写并编译的类没有太大区别，对方法的调用和直接调用普通类的方式一致，所以CGLib执行代理方法的效率要高于JDK的动态代理；

（三）缺点

由于CGLib的代理类使用的是继承，这也就意味着如果需要被代理的类是一个final类，则无法使用CGLib代理；

由于CGLib实现代理方法的方式是重写父类的方法，所以无法对final方法，或者private方法进行代理，因为子类无法重写这些方法；

CGLib生成代理类的方式是通过操作字节码，这种方式生成代理类的速度要比JDK通过反射生成代理类的速度更慢；

## AOP术语

### 连接点(Join point)

能够被拦截的地方

### 切点(Poincut)

具体定位的连接点

### 增强/通知(Advice)

表示添加到切点的一段逻辑代码，并定位连接点的方位信息

### 织入(Weaving)

将增强/通知添加到目标类的具体连接点上的过程。

### 引入/引介(Introduction)

允许我们向现有的类添加新方法或属性。是一种特殊的增强！

### 切面(Aspect)

切面由切点和增强/通知组成，它既包括了横切逻辑的定义、也包括了连接点的定义

## Spring对AOP的支持

- 基于代理的经典SpringAOP：需要实现接口，手动创建代理
- 纯POJO切面：使用XML配置，aop命名空间
- @AspectJ注解驱动的切面：使用注解的方式，这是最简洁和最方便的！

## 怎么定义一个注解

### 引入依赖

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 定义注解

#### 元注解

java.lang.annotation 提供了四种元注解，专门注解其他的注解（在自定义注解的时候，需要使用到元注解）：

- @Documented – 注解是否将包含在JavaDoc中
- @Retention – 什么时候使用该注解
- @Target – 注解用于什么地方
- @Inherited – 是否允许子类继承该注解

##### @Retention
定义该注解的生命周期

- RetentionPolicy.SOURCE : 在编译阶段丢弃。这些注解在编译结束之后就不再有任何意义，所以它们不会写入字节码。@Override, @SuppressWarnings都属于这类注解。
- RetentionPolicy.CLASS : 在类加载的时候丢弃。在字节码文件的处理中有用。注解默认使用这种方式
- RetentionPolicy.RUNTIME : 始终不会丢弃，运行期也保留该注解，因此可以使用反射机制读取该注解的信息。我们自定义的注解通常使用这种方式。

##### @Target
表示该注解用于什么地方。默认值为任何元素，表示该注解用于什么地方。可用的ElementType 参数包括

- ElementType.CONSTRUCTOR: 用于描述构造器
- ElementType.FIELD: 成员变量、对象、属性（包括enum实例）
- ElementType.LOCAL_VARIABLE: 用于描述局部变量
- ElementType.METHOD: 用于描述方法
- ElementType.PACKAGE: 用于描述包
- ElementType.PARAMETER: 用于描述参数
- ElementType.TYPE: 用于描述类、接口(包括注解类型) 或enum声明

##### @Documented
一个简单的Annotations 标记注解，表示是否将注解信息添加在java文档中。

##### @Inherited
定义该注释和子类的关系

@Inherited 元注解是一个标记注解，@Inherited 阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited 修饰的annotation 类型被用于一个class，则这个annotation 将被用于该class
的子类。

### 示例

自定义一个检查是否登录的注解

```java

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CheckLogin {

}
```

实现

```java

@Aspect
@Component
@Slf4j
@Order(1)
public class CheckLoginAspect {

    @Autowired
    RedisTemplate redisTemplate;

    @Before("execution(* *..controller..*(..))")
    public void before(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        CheckLogin annotation = method.getAnnotation(CheckLogin.class);

        if (annotation == null) {
            //获取类上注解
            annotation = joinPoint.getTarget().getClass().getAnnotation(CheckLogin.class);
        }
        if (annotation != null) {
            //获取到请求的属性
            ServletRequestAttributes attributes =
                    (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            //获取到请求对象
            HttpServletRequest request = attributes.getRequest();
            String ssoToken = HttpUtil.getSsoToken(request);
            if (ssoToken != null) {
                String loginUserTokenKey = AuthRedisKeyUtil.getLoginUserTokenKey(ssoToken);
                if (redisTemplate.hasKey(loginUserTokenKey)) {
                    //通过
                } else {
                    throw new LoginException("登录已过期");
                }
            } else {
                throw new IllegalRequestException("非法请求");
            }
        }
    }
}
```

使用

```java

@RestController
public class AccountController {

    @CheckLogin
    @GetMapping("/query")
    public JSONObject queryRegulation(Integer pageNum, Integer pageSize) {
        //....业务逻辑
    }
}
```

