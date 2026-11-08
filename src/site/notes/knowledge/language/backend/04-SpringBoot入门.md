---
{"dg-publish":true,"permalink":"/knowledge/language/backend/04-SpringBoot入门/","title":"Spring Boot 入门","tags":["java","spring","springboot","web框架","ioc","aop","jpa","rest","微服务"],"dg-note-properties":{"category":"后端 / Web 框架","title":"Spring Boot 入门","date":"2026-11-06","tags":["java","spring","springboot","web框架","ioc","aop","jpa","rest","微服务"]}}
---


# Spring Boot 入门

> Java 生态的绝对主流。和 Flask/Gin 的「轻量框架」思路不同，Spring 是**全家桶**：IoC 容器、AOP、数据访问、Web、安全、微服务一应俱全。
> Spring Boot 的价值在于——把这些能力**开箱即用地组装好**，让你不用再写一堆 XML 配置。

返回 [[knowledge/language/backend/index\|🌐 后端开发]]

---

## 一、Spring 到底是什么

```
Spring Framework（核心）
├── IoC 容器      —— 对象的创建与依赖注入交给框架
├── AOP           —— 横切逻辑（日志、事务、权限）与业务解耦
├── Spring MVC    —— Web 层
├── Spring Data   —— 数据访问统一抽象
└── ...

Spring Boot（脚手架）
└── 在上面这些之上做「自动配置 + 内嵌服务器 + Starter 依赖」
```

**一句话**：Spring 是能力，Spring Boot 是「让你 5 分钟跑起来」的封装。

### 1.1 和 Flask / Gin 的核心差异

| 维度 | Flask / Gin | Spring Boot |
|:--|:--|:--|
| **哲学** | 微框架，按需装插件 | 全家桶，约定优于配置 |
| **依赖注入** | 无（自己 new） | **IoC 容器内置**，`@Autowired` 全自动 |
| **ORM** | 自己选（SQLAlchemy / GORM） | Spring Data JPA / MyBatis |
| **启动速度** | 毫秒级 | 秒级（JVM 启动成本） |
| **内存占用** | 几十 MB | 几百 MB 起 |
| **企业接受度** | 中 | **极高**（银行、保险、政企标配） |
| **学习曲线** | 平缓 | 陡（概念多：IoC/AOP/Bean 生命周期…） |

**选型建议**：做小工具、快速原型 → Flask/Gin；做企业级业务系统、团队协作、需要成熟生态 → Spring Boot。

---

## 二、环境与最小示例

**前提**：JDK 17+（Spring Boot 3.x 最低要求）、Maven 或 Gradle。

### 2.1 用 Spring Initializr 建项目

访问 [start.spring.io](https://start.spring.io)，选：

| 项 | 值 |
|:--|:--|
| Project | Maven |
| Language | Java |
| Spring Boot | 3.3.x |
| Packaging | Jar |
| Java | 17 或 21 |
| Dependencies | **Spring Web**（够跑第一个例子） |

下载解压后目录结构：

```
demo/
├── pom.xml
└── src/main/java/com/example/demo/
    ├── DemoApplication.java
    └── controller/
```

### 2.2 主类

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication          // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 2.3 第一个接口

```java
package com.example.demo.controller;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello Spring Boot";
    }

    @GetMapping("/ping")
    public Map<String, Object> ping() {
        return Map.of("message", "pong", "time", System.currentTimeMillis());
    }
}
```

运行：

```bash
./mvnw spring-boot:run
# 或先打包再跑
./mvnw clean package && java -jar target/demo-0.0.1-SNAPSHOT.jar
```

访问 `http://localhost:8080/api/hello` → `Hello Spring Boot`

> **注意**：Spring Boot 内嵌了 Tomcat，**不需要单独装服务器**——这是它当年最大的改进。

---

## 三、IoC 与依赖注入（Spring 的灵魂）

### 3.1 什么是 IoC

传统写法里，A 需要 B 就自己 `new B()`——耦合死了。IoC 反过来：**B 由容器创建并「注入」给 A**，A 只管用。

```java
// ❌ 传统：强耦合，无法替换、无法 mock
public class OrderService {
    private UserRepo repo = new UserRepo();
}

// ✅ Spring：容器负责创建和注入
@Service
public class OrderService {
    private final UserRepo repo;

    public OrderService(UserRepo repo) {   // 构造器注入（推荐）
        this.repo = repo;
    }
}
```

### 3.2 三种注入方式

| 方式 | 写法 | 评价 |
|:--|:--|:--|
| **构造器注入** | 构造函数参数 | ✅ **推荐**，依赖不可变、易测试 |
| Setter 注入 | `@Autowired` 在 setter 上 | 可选依赖时用 |
| 字段注入 | `@Autowired` 在字段上 | ❌ 不推荐，容易掩盖循环依赖 |

### 3.3 常用注解

| 注解 | 作用 |
|:--|:--|
| `@Component` | 通用组件，交给容器管理 |
| `@Service` | 业务层（本质同 `@Component`，语义更明确） |
| `@Repository` | 数据层，附带异常转换 |
| `@Controller` / `@RestController` | Web 层 |
| `@Configuration` | 配置类，里面可以写 `@Bean` |
| `@Bean` | 方法返回值注册为容器里的对象 |

```java
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper().configure(FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```

### 3.4 Bean 的作用域与生命周期

| 作用域 | 说明 |
|:--|:--|
| `singleton` | 默认，全容器一个实例 |
| `prototype` | 每次获取新建 |
| `request` / `session` | Web 请求级 / 会话级 |

生命周期钩子：`@PostConstruct`（初始化后）、`@PreDestroy`（销毁前）。

> ⚠️ **单例 Bean 默认非线程安全**。Spring 帮不了你，别在单例里存用户相关的成员变量。

---

## 四、路由与请求参数

### 4.1 路由映射

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping                      // GET /users
    public List<User> list() { ... }

    @GetMapping("/{id}")             // GET /users/1
    public User get(@PathVariable Long id) { ... }

    @PostMapping                     // POST /users
    public User create(@RequestBody User u) { ... }

    @PutMapping("/{id}")             // PUT /users/1
    public User update(@PathVariable Long id, @RequestBody User u) { ... }

    @DeleteMapping("/{id}")          // DELETE /users/1
    public void delete(@PathVariable Long id) { ... }
}
```

### 4.2 参数绑定四种方式

| 场景 | 注解 | 示例 |
|:--|:--|:--|
| 路径变量 | `@PathVariable` | `/users/{id}` |
| 查询参数 | `@RequestParam` | `?page=1&size=10` |
| 请求体 JSON | `@RequestBody` | POST 的 JSON body |
| 表单 / 头 | `@ModelAttribute` / `@RequestHeader` | — |

```java
@GetMapping("/search")
public Page<User> search(
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestHeader(value = "X-Token", required = false) String token) {
    ...
}
```

### 4.3 参数校验（JSR-380）

```java
public record CreateUserReq(
        @NotBlank(message = "用户名不能为空") String username,
        @Email(message = "邮箱格式错误") String email,
        @Min(value = 18, message = "年龄必须 ≥18") int age
) {}

@PostMapping
public User create(@Valid @RequestBody CreateUserReq req) { ... }
```

校验失败会抛 `MethodArgumentNotValidException`，配一个全局异常处理器统一返回即可（见第七章）。

---

## 五、响应与统一返回体

### 5.1 返回类型

```java
@RestController   // = @Controller + @ResponseBody，方法返回值直接序列化为 JSON

@GetMapping("/a")
public String text() { return "hello"; }              // text/plain

@GetMapping("/b")
public Map<String, Object> json() {                    // application/json
    return Map.of("code", 0, "data", List.of());
}

@GetMapping("/c")
public ResponseEntity<User> withStatus() {
    return ResponseEntity.status(201).body(user);      // 自定义状态码
}
```

### 5.2 统一返回结构（实战必备）

```java
public record R<T>(int code, String msg, T data) {
    public static <T> R<T> ok(T data) { return new R<>(0, "ok", data); }
    public static <T> R<T> fail(int code, String msg) { return new R<>(code, msg, null); }
}
```

所有 Controller 统一返回 `R<T>`，前端只用处理一种结构。

---

## 六、数据访问：Spring Data JPA

### 6.1 实体与仓库

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    private String email;
    // getter/setter 省略
}
```

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // 方法名即查询：findByUsername → SELECT * WHERE username = ?
    Optional<User> findByUsername(String username);

    List<User> findByEmailContainingAndAgeGreaterThan(String kw, int age);

    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailCustom(@QueryParam("email") String email);
}
```

### 6.2 主要能力

| 能力 | 说明 |
|:--|:--|
| **方法名派生查询** | `findByXxxAndYyy` 自动生成 SQL |
| **分页排序** | `Pageable` 参数直接用，返回 `Page<T>` |
| **事务** | `@Transactional` 注解，默认回滚运行时异常 |
| **审计** | `@CreatedDate` / `@LastModifiedDate` 自动填时间 |

```java
@GetMapping
public Page<User> list(@RequestParam int page, @RequestParam int size) {
    return repo.findAll(PageRequest.of(page, size, Sort.by("id").descending()));
}
```

### 6.3 JPA 还是 MyBatis

| 维度 | Spring Data JPA | MyBatis |
|:--|:--|:--|
| **SQL 控制** | 自动生成，复杂查询要 JPQL | 手写 SQL，完全可控 |
| **上手速度** | 快 | 稍慢（要写 XML/注解） |
| **复杂报表** | 吃力 | 擅长 |
| **国内主流** | 中 | **高**（尤其甲方项目） |

> 很多团队是「JPA 做 CRUD + MyBatis 做报表」混用。

---

## 七、全局异常处理与拦截器

### 7.1 统一异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public R<Void> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .findFirst().orElse("参数错误");
        return R.fail(400, msg);
    }

    @ExceptionHandler(BusinessException.class)
    public R<Void> handleBiz(BusinessException e) {
        return R.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public R<Void> handleAll(Exception e) {
        log.error("未处理异常", e);
        return R.fail(500, "服务内部错误");
    }
}
```

### 7.2 拦截器（Interceptor）

```java
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
        String token = req.getHeader("X-Token");
        if (token == null) {
            resp.setStatus(401);
            return false;       // 中断请求
        }
        return true;
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login");
    }
}
```

---

## 八、配置与 Profile

### 8.1 配置文件

`src/main/resources/application.yml`

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useSSL=false
    username: root
    password: ${DB_PASSWORD:changeme}   # 支持环境变量覆盖，冒号后是默认值
  jpa:
    hibernate:
      ddl-auto: update                   # 开发用 update，生产务必 none/validate
    show-sql: true

logging:
  level:
    com.example: debug
```

### 8.2 多环境 Profile

```yaml
# application-dev.yml / application-prod.yml
```

```bash
java -jar app.jar --spring.profiles.active=prod
# 或环境变量
export SPRING_PROFILES_ACTIVE=prod
```

| 场景 | 建议 |
|:--|:--|
| `ddl-auto` | 开发 `update`；**生产 `none`**，表结构交给迁移工具 |
| 密码 | 绝不写死在 yml，用环境变量或配置中心 |
| 日志级别 | 生产 `info`，排查时临时调 `debug` |

### 8.3 配置读取

```java
@Value("${server.port}")
private int port;

// 或者类型安全的方式
@ConfigurationProperties(prefix = "app.mail")
public record MailProps(String host, int port, String from) {}
```

---

## 九、AOP：横切逻辑的正确姿势

日志、事务、权限这类「每个方法都要来一遍」的逻辑，用 AOP 抽出去。

```java
@Aspect
@Component
public class LogAspect {

    @Around("@annotation(LogExecution)")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            log.info("{} 耗时 {}ms", pjp.getSignature(), System.currentTimeMillis() - start);
        }
    }
}
```

常见切点表达式：

| 表达式 | 匹配 |
|:--|:--|
| `execution(* com.example.service..*(..))` | service 包下所有方法 |
| `@annotation(org.springframework.transaction.annotation.Transactional)` | 带某注解的方法 |
| `within(@RestController *)` | 所有 Controller 类 |

---

## 十、打包与部署

### 10.1 打成可执行 Jar

```bash
./mvnw clean package
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

Spring Boot 的 Jar 是**fat jar**——依赖全打进去了，`java -jar` 直接跑。

### 10.2 Docker 部署

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/demo-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

```bash
docker build -t demo:1.0 .
docker run -d -p 8080:8080 --name demo demo:1.0
```

> 用 `-XX:MaxRAMPercentage` 而不是写死 `-Xmx`，容器内存调整时不用改镜像。

### 10.3 生产建议

| 项 | 建议 |
|:--|:--|
| **JVM 参数** | `-XX:MaxRAMPercentage=75 -XX:+UseG1GC` |
| **健康检查** | 引入 `spring-boot-starter-actuator`，暴露 `/actuator/health` |
| **优雅停机** | `server.shutdown=graceful` |
| **表结构管理** | 用 Flyway / Liquibase，**别用 `ddl-auto: update`** |
| **监控** | Micrometer + Prometheus |

---

## 十一、常见坑

| 坑 | 现象 | 解决 |
|:--|:--|:--|
| **循环依赖** | 启动报 `BeanCurrentlyInCreationException` | 重构拆分，别用 `@Lazy` 硬绕 |
| **单例存状态** | 并发下数据串号 | 单例 Bean 必须无状态 |
| **`@Transactional` 失效** | 事务没生效 | 同类内部调用不经过代理；`private` 方法无效 |
| **`ddl-auto: update` 上生产** | 表结构被悄悄改坏 | 生产用 `none` + Flyway |
| **端口被占** | `Port 8080 already in use` | `server.port=8081` |
| **JSON 序列化循环引用** | 实体互相引用导致无穷递归 | DTO 隔离，别直接返实体 |
| **配置读不到** | 属性注入为 null | 检查包扫描范围 / `@ConfigurationProperties` 是否注册 |
| **fat jar 启动慢** | 冷启动 10s+ | 正常现象；必要时上 GraalVM native image |

---

## 十二、一句话总结

> **Spring Boot = IoC 容器 + 自动配置 + 内嵌服务器 + Starter 依赖**。
> 学会 IoC 就懂了一半，剩下的都是「在正确的地方加正确的注解」。

---

## 相关文档

- [[knowledge/language/backend/01-Flask入门\|Flask 入门]] — Python 轻量框架对照
- [[knowledge/language/backend/02-Django入门\|Django 入门]] — Python 全家桶框架对照
- [[knowledge/language/backend/03-Gin入门\|Gin 入门]] — Go 框架对照
- `devops/practices/03-CI工程化实战.md` — Maven/Gradle 在 CI 里的用法
