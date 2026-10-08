# 拦截器与过滤器

> 本文讲解 Spring MVC 拦截器（Interceptor）与 Servlet 过滤器（Filter）的区别、执行顺序与使用场景。

## 目录

- [一、先看一张图：请求处理链路](#一先看一张图请求处理链路)
- [二、核心区别对比](#二核心区别对比)
- [三、执行顺序](#三执行顺序)
- [四、代码示例](#四代码示例)
- [五、典型使用场景](#五典型使用场景)
- [六、常见问题](#六常见问题)
- [七、概念澄清：三种「拦截器」](#七概念澄清三种拦截器)
- [八、现代项目的配置组织方式](#八现代项目的配置组织方式)

## 一、先看一张图：请求处理链路

理解两者区别的关键，是搞清它们在**请求处理链路中的位置**：

```text
HTTP 请求
   ↓
┌──────────────────────────────────────────┐
│  Filter 过滤链（Servlet 规范）              │
│  Filter1 → Filter2 → ...                  │
│  ┌────────────────────────────────────┐   │
│  │  DispatcherServlet（Spring MVC）    │   │
│  │  ┌──────────────────────────────┐  │   │
│  │  │ Interceptor1.preHandle       │  │   │
│  │  │ Interceptor2.preHandle       │  │   │
│  │  │ ┌──────────────────────────┐ │  │   │
│  │  │ │   Handler（Controller）   │ │  │   │
│  │  │ └──────────────────────────┘ │  │   │
│  │  │ Interceptor2.postHandle      │  │   │
│  │  │ Interceptor1.postHandle      │  │   │
│  │  │ Interceptor2.afterCompletion │  │   │
│  │  │ Interceptor1.afterCompletion │  │   │
│  │  └──────────────────────────────┘  │   │
│  └────────────────────────────────────┘   │
│  Filter2 → Filter1（返回时反向）            │
└──────────────────────────────────────────┘
   ↓
HTTP 响应
```

**一句话概括**：

- **Filter 在 DispatcherServlet 之前**，是 Servlet 容器级别的，能拿到原始的 `ServletRequest`。
- **Interceptor 在 DispatcherServlet 内部**，是 Spring MVC 级别的，属于 `HandlerExecutionChain` 的一部分，能拿到即将执行的 `HandlerMethod`。

## 二、核心区别对比

| 对比维度 | Filter（过滤器） | Interceptor（拦截器） |
|:---|:---|:---|
| **规范归属** | **Servlet 规范**（J2EE） | **Spring MVC 框架** |
| **依赖容器** | 依赖 Servlet 容器（Tomcat） | 依赖 Spring 容器 |
| **接口** | `javax.servlet.Filter` | `org.springframework.web.servlet.HandlerInterceptor` |
| **处理对象** | `ServletRequest` / `ServletResponse` | `HttpServletRequest` / `HttpServletResponse` / `HandlerMethod` |
| **触发时机** | DispatcherServlet **之前** | DispatcherServlet **内部**、Handler **之前/后** |
| **能否获取 Bean** | ❌ 默认不能（需特殊处理） | ✅ 可以直接 `@Autowired` |
| **能否获取请求的 Handler** | ❌ 不知道最终处理请求的方法 | ✅ 可拿到 `HandlerMethod` |
| **粒度** | **粗**（按 URL 匹配） | **细**（可按方法、注解匹配） |
| **能否中断请求** | ✅ 可以（不调用 `chain.doFilter`） | ✅ 可以（`preHandle` 返回 false） |
| **典型用途** | 编码、跨域、请求日志、敏感词过滤 | 登录校验、权限控制、日志、参数预处理 |
| **配置方式** | `@WebFilter` / `FilterRegistrationBean` | `WebMvcConfigurer#addInterceptors` |

**最容易混淆的一点**：**Filter 拿不到「即将执行的 Controller 方法」**，而 Interceptor 可以（通过 `HandlerMethod`）。所以「按注解做权限控制」这类需求只能用 Interceptor。

## 三、执行顺序

**完整执行流程**：

```text
Filter#doFilter 前置逻辑
  → Interceptor#preHandle
    → Handler（Controller 方法）
  ← Interceptor#postHandle        （Controller 返回但视图未渲染时）
  ← Interceptor#afterCompletion   （视图渲染完成后，可拿到异常）
Filter#doFilter 后置逻辑（chain.doFilter 之后的代码）
```

**多个时的顺序规则**：

| 类型 | 规则 |
|:---|:---|
| **多个 Filter** | `@Order` / `FilterRegistrationBean#setOrder` 数字小的先执行；**返回时顺序相反** |
| **多个 Interceptor** | 按注册顺序执行 `preHandle`；**`postHandle` 和 `afterCompletion` 逆序执行** |

举例：注册 I1、I2 两个拦截器

```text
I1.preHandle → I2.preHandle → Controller
→ I2.postHandle → I1.postHandle
→ I2.afterCompletion → I1.afterCompletion
```

**为什么 Interceptor 的前后顺序是「对称」的？** 类似栈的结构，先进后出。这样能保证「开启资源」和「释放资源」成对出现（如 I1 开启事务、I2 记录日志，则 I2 先收尾，I1 最后提交）。

**关于 preHandle 返回 false 的影响**：

- 返回 false 表示**中断请求**，后续拦截器的 `preHandle` 和 Controller 都不会执行。
- **但已执行成功的拦截器，其 `afterCompletion` 仍会被调用**（用于释放资源），`postHandle` 不会调用。

## 四、代码示例

### Filter

```java
@Slf4j
@Component
public class RequestLogFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        long start = System.currentTimeMillis();

        log.info("Filter 请求前：{} {}", req.getMethod(), req.getRequestURI());

        // 放行；不调用则请求被中断
        chain.doFilter(request, response);

        log.info("Filter 请求后：耗时 {}ms", System.currentTimeMillis() - start);
    }
}
```

**若需要在 Filter 中使用 Spring Bean**（因为 Filter 由 Servlet 容器创建，默认不受 Spring 管理）：

```java
// 方式一：把 Filter 注册为 Bean（上面的 @Component 就是）
// 方式二：改用 OncePerRequestFilter（Spring 提供，天然可注入 Bean）
@Component
public class AuthFilter extends OncePerRequestFilter {

    @Autowired
    private TokenService tokenService;    // ✅ 可以直接注入

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        String token = request.getHeader("Authorization");
        if (!tokenService.validate(token)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;   // 不调用 doFilter，中断请求
        }
        filterChain.doFilter(request, response);
    }
}
```

> **`OncePerRequestFilter`** 是 Spring 提供的 Filter 基类，它保证**在异步请求/转发时也只执行一次**（普通 Filter 可能被多次调用）。

### Interceptor

```java
@Slf4j
@Component
public class LoginInterceptor implements HandlerInterceptor {

    /** 在 Controller 方法执行前调用：返回 false 则中断请求 */
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        // 可以拿到即将执行的目标方法
        if (handler instanceof HandlerMethod handlerMethod) {
            log.info("即将执行：{}", handlerMethod.getMethod().getName());
        }

        String token = request.getHeader("Authorization");
        if (token == null || !valid(token)) {
            response.setStatus(401);
            return false;    // 中断
        }
        return true;         // 放行
    }

    /** Controller 返回后、视图渲染前调用：可修改 ModelAndView */
    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler,
                           ModelAndView modelAndView) throws Exception {
        log.info("Controller 执行完毕");
    }

    /** 视图渲染完成后调用：可拿到异常，用于资源清理 */
    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) throws Exception {
        log.info("请求处理完成，异常：{}", ex == null ? "无" : ex.getMessage());
    }
}
```

**注册拦截器**：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/**")                        // 拦截所有路径
                .excludePathPatterns("/login", "/register");   // 排除白名单
    }
}
```

**三个方法的执行时机小结**：

| 方法 | 执行时机 | 能否拿到异常 | 常见用途 |
|:---|:---|:---|:---|
| `preHandle` | Controller **之前** | — | 登录校验、权限判断 |
| `postHandle` | Controller **之后**、视图渲染前 | — | 修改 ModelAndView |
| `afterCompletion` | 视图渲染**完成后** | ✅ 可拿到 `Exception` | 资源清理、异常日志 |

## 五、典型使用场景

| 需求 | 推荐 | 原因 |
|:---|:---|:---|
| **请求/响应编码设置** | **Filter** | 需要在最外层统一处理，早于 Spring MVC |
| **CORS 跨域处理** | **Filter** | 跨域响应头要在 DispatcherServlet 之前写入 |
| **请求/响应体加解密** | **Filter** | 需要包装 `ServletRequest` / `ServletResponse` |
| **全站请求日志（含耗时）** | **Filter** | 能覆盖 DispatcherServlet 的完整耗时 |
| **登录 / Token 校验** | **Interceptor** | 能拿到 `HandlerMethod`，方便排除白名单 |
| **权限注解鉴权** | **Interceptor** | 可读取方法上的注解（Filter 做不到） |
| **请求参数预处理** | **Interceptor** | 在 `preHandle` 中处理更灵活 |
| **敏感词过滤** | **Filter** | 影响整个请求体，属于容器级处理 |

**判断口诀**：

> **要拿到「即将执行的 Controller 方法」或在 Spring 体系内工作 → Interceptor**
>
> **要处理原始请求/响应、包装 request、或需要覆盖整个 Servlet 链路 → Filter**

## 六、常见问题

**1. 为什么 Interceptor 能注入 Bean，Filter 默认不能？**

- **Interceptor** 由 `WebMvcConfigurer` 注册，实例由 **Spring 容器**管理，自然支持依赖注入。
- **Filter** 由 **Servlet 容器（Tomcat）** 创建和调用，不在 Spring 容器内，因此默认拿不到 Bean。

**解决**：把 Filter 声明为 `@Component`（或改用 `OncePerRequestFilter`），Spring Boot 会自动把它注册到 Servlet 容器，此时就能注入了。

**2. Filter 和 Interceptor 谁先执行？**

**Filter 先**。Filter 在 `DispatcherServlet` 之前，Interceptor 在 `DispatcherServlet` 内部。执行顺序：

```text
Filter#doFilter 前置 → Interceptor#preHandle → Controller
→ Interceptor#postHandle → Interceptor#afterCompletion → Filter 后置
```

**3. 如何在 Interceptor 中获取请求的 Controller 方法？**

`preHandle` 的第三个参数 `handler` 就是 `HandlerMethod`：

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                         Object handler) {
    if (handler instanceof HandlerMethod handlerMethod) {
        Method method = handlerMethod.getMethod();
        // 读取方法上的注解，做权限校验
        RequirePermission perm = method.getAnnotation(RequirePermission.class);
        if (perm != null && !check(perm.value())) {
            return false;
        }
    }
    return true;
}
```

**这是 Filter 做不到的**——Filter 阶段还没进行 Handler 映射，不知道最终会执行哪个方法。

**4. `postHandle` 和 `afterCompletion` 的区别？**

| 对比项 | `postHandle` | `afterCompletion` |
|:---|:---|:---|
| 时机 | Controller 返回后、视图渲染**前** | 视图渲染**后** |
| 能否拿异常 | ❌ | ✅ 可以 |
| 能否改 ModelAndView | ✅ | ❌ |
| Controller 抛异常时 | **不执行** | **仍执行** |

**5. 拦截器如何做全局异常处理？**

在 `afterCompletion` 中可以拿到异常，但更推荐用 **`@ControllerAdvice` + `@ExceptionHandler`**：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<?> handle(BusinessException e) {
        return Result.fail(e.getMessage());
    }
}
```

> `@ControllerAdvice` 本质上也是基于拦截器/AOP 机制实现的，但语义更清晰，且能返回统一的响应体。

**6. 两个 Filter 执行顺序怎么控制？**

```java
// 方式一：@Order 注解
@Order(1)
@Component
public class FilterA implements Filter { }

// 方式二：FilterRegistrationBean
@Bean
public FilterRegistrationBean<FilterB> filterB() {
    FilterRegistrationBean<FilterB> bean = new FilterRegistrationBean<>();
    bean.setFilter(new FilterB());
    bean.addUrlPatterns("/*");
    bean.setOrder(2);        // 数字小的先执行
    return bean;
}
```

> **注意**：`@Order` 对通过 `FilterRegistrationBean` 注册的 Filter 无效，对 `@Component` 注解的 Filter 有效（Spring Boot 场景）。

## 七、概念澄清：三种「拦截器」

**这是最容易混淆的地方**——Java 生态里叫「拦截器」的东西**不止一种**，它们拦截的对象完全不同。

| 名称 | 所属体系 | 拦截什么 | 核心接口 | 注册方式 |
|:---|:---|:---|:---|:---|
| **Spring MVC 拦截器** | Spring MVC | **HTTP 请求**（URL 级别） | `HandlerInterceptor` | `WebMvcConfigurer#addInterceptors` |
| **Servlet 过滤器** | Servlet 规范 | **HTTP 请求**（更外层） | `Filter` | `FilterRegistrationBean` / `@Component` |
| **MyBatis 插件拦截器** | MyBatis | **SQL 执行**（SQL 级别） | `Interceptor` / `InnerInterceptor` | `MybatisPlusInterceptor` Bean |

**关键区别**：

- 前两者拦截的是**请求**——决定「这个请求能不能进来、要不要放行」。
- 第三个拦截的是**SQL**——决定「这条 SQL 怎么改写、怎么执行」。

**如何快速判断**：

```text
需求是「登录校验、权限控制、接口日志」     → Spring MVC 拦截器
需求是「分页、多租户隔离、数据权限、SQL 日志」 → MyBatis 插件拦截器
需求是「编码、跨域、请求体加解密」          → Servlet 过滤器
```

> ⚠️ **实践提示**：团队口头讨论时说「加个拦截器」，**一定要先确认是哪种**。曾经有过把「数据权限」需求做成 MVC 拦截器的错误案例——MVC 拦截器在 Controller 层面工作，**根本拿不到最终生成的 SQL**，无法对 SQL 做精细化改写。

### MyBatis-Plus 插件拦截器长什么样

作为对比，看一下 MyBatis-Plus 的插件注册方式（与 MVC 拦截器完全不同）：

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    // 分页插件
    interceptor.addInnerInterceptor(paginationInnerInterceptor());
    // 乐观锁插件
    interceptor.addInnerInterceptor(optimisticLockerInnerInterceptor());
    return interceptor;
}
```

它注册的是**一个 Bean**，而不是往 `WebMvcConfigurer` 里塞。详细原理见 [MyBatis 插件机制](../../中间件/Mybatis/mybatis.md#mybatis-插件机制)。

## 八、现代项目的配置组织方式

在 Spring Boot 项目里，「注册」这件事**不是散落各处手工编码**，而是**框架约定好的标准化范式**。理解这个范式，比记住具体 API 更重要。

### 统一范式：`@Configuration` + 交给 Spring

无论哪个框架，注册入口都遵循同一套模式：

```text
① 定义组件（实现框架提供的接口）
        ↓
② 交给 Spring 托管（@Component 或 @Bean）
        ↓
③ 框架在启动时自动装配（约定好的扩展点）
```

**开发者的实际工作只有前两步**，第三步由框架完成——这就是「框架代理」的含义。

### 典型项目的配置类结构

实际项目里通常是一个 `config` 包，各管一摊：

```text
config/
├── WebMvcConfig.java          ← 拦截器、跨域、消息转换器、静态资源
├── MybatisPlusConfig.java     ← 分页插件、乐观锁、多租户、数据权限
├── SwaggerConfig.java         ← 接口文档
├── RedisConfig.java           ← 序列化方式
└── AsyncConfig.java           ← 线程池
```

**关键点**：这些配置类**互不干涉**，各自往自己框架的扩展点里注册组件。

| 框架 | 注册入口 | 注册什么 |
|:---|:---|:---|
| Spring MVC | `WebMvcConfigurer#addInterceptors` | `HandlerInterceptor` |
| Spring MVC | `WebMvcConfigurer#addCorsMappings` | 跨域配置 |
| MyBatis-Plus | `MybatisPlusInterceptor` Bean | `InnerInterceptor` 插件 |
| Servlet | `FilterRegistrationBean` | `Filter` |
| Spring | `@Bean` | 任意第三方组件 |

### 为什么 Spring MVC 拦截器需要「两步」（Bean + Config）？

这是初学者最常见的困惑：**为什么写个拦截器还要单独建个配置类？**

原因是职责分离：

| 步骤 | 职责 | 不做的后果 |
|:---|:---|:---|
| **加 `@Component`** | 让拦截器成为 Spring Bean | 无法用 `@Autowired` 注入依赖（如 `TokenService`） |
| **在 Config 注册** | 告诉 `HandlerMapping`：**哪些 URL 要挂这个拦截器** | 拦截器不会生效；或生效范围失控（默认拦全部） |

**「注册」的本质**是：Spring MVC 启动时收集所有 `WebMvcConfigurer`，把拦截器**装配进 `HandlerMapping`**；请求到达时，`HandlerMapping` 按 URL 匹配，**动态组装出 `HandlerExecutionChain`**（目标 Handler + 匹配到的拦截器列表）。

这也解释了为什么 `addPathPatterns` / `excludePathPatterns` 很重要——它们决定了拦截器的**生效边界**：

```java
registry.addInterceptor(loginInterceptor)
        .addPathPatterns("/**")                        // 拦截范围
        .excludePathPatterns("/login", "/register");   // 白名单
```

### 小结：从「写代码」到「框架生效」的完整链路

```text
开发者写拦截器类（实现 HandlerInterceptor）
        ↓ @Component 交给 Spring
Spring 容器创建 Bean
        ↓ 开发者写 WebMvcConfigurer 注册
Spring MVC 启动时收集配置，装配进 HandlerMapping
        ↓ 请求到达
HandlerMapping 匹配 URL，组装 HandlerExecutionChain
        ↓
执行 拦截器 → Controller → 拦截器
```

**记忆要点**：开发者负责「**写什么**」（拦截逻辑）和「**挂哪里**」（路径范围），框架负责「**什么时候调**」（装配与执行）。
