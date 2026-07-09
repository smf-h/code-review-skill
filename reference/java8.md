# Java 8 / Legacy Stack Code Review Guide

面向仍运行在 **Java 8**（及相近遗留栈：Spring Boot 2.x、`javax.*`、Hibernate 5）上的代码审查。不要用 Java 17/21 特性（Records、文本块、虚拟线程、`ProblemDetail` 等）去要求这类 PR。

> 现代栈（Java 17/21 + Spring Boot 3）请用 [Java Guide](java.md)。

## 目录

- [适用范围](#适用范围)
- [Lambda 与函数式接口](#lambda-与函数式接口)
- [Stream API](#stream-api)
- [Optional](#optional)
- [Date/Time API (`java.time`)](#datetime-api-javatime)
- [并发：线程池与 CompletableFuture](#并发线程池与-completablefuture)
- [Spring Boot 2](#spring-boot-2)
- [JPA / Hibernate 5](#jpa--hibernate-5)
- [异常处理](#异常处理)
- [测试](#测试)
- [Review Checklist](#review-checklist)
- [References](#references)

---

## 适用范围

| 场景 | 加载指南 |
|------|----------|
| Java 8 / 11（未用现代语法）、Spring Boot 2.x、`javax.persistence` | **本文件** |
| Java 17/21、Spring Boot 3、`jakarta.*`、虚拟线程 | [java.md](java.md) |
| 正在从 Boot 2 → Boot 3 迁移 | 两边对照：本文件看遗留坑，`java.md` 看目标态 |

审查时先确认 `pom.xml` / `build.gradle` 的 `java.version`、Spring Boot 版本和 `javax` vs `jakarta` 包名，再选指南。

---

## Lambda 与函数式接口

### 保持短小，优先方法引用

```java
// ❌ Lambda 过长，难读、难测、难调试
users.stream().forEach(u -> {
    // 十几行业务逻辑...
});

// ✅ 抽成方法，或使用方法引用
users.forEach(this::processUser);
```

### 优先使用 JDK 已有函数式接口

```java
// ❌ 无必要的自定义函数式接口
@FunctionalInterface
interface UserCallback {
    void accept(User u);
}

// ✅ 用 Consumer / Function / Predicate / Supplier / BiFunction 等
void process(Consumer<User> callback) { ... }
```

### 捕获变量必须 effectively final

```java
// ❌ 修改被捕获变量 → 编译失败或被迫用数组/Atomic 糊弄
int sum = 0;
list.forEach(n -> sum += n); // 编译错误

// ✅ 用 Stream 归约，或显式可变累加器类型
int sum = list.stream().mapToInt(Integer::intValue).sum();
```

---

## Stream API

### 简单循环不要硬上 Stream

```java
// ❌ 副作用 + 无转换，Stream 无收益
items.stream().forEach(item -> process(item));

// ✅ for-each 更清晰
for (Item item : items) {
    process(item);
}
```

### 收集结果：Java 8 用 `Collectors`

```java
// ❌ Java 16+ 的 Stream.toList() —— Java 8 没有
list.stream().map(...).toList();

// ✅ Java 8
List<Dto> result = list.stream()
    .map(...)
    .collect(Collectors.toList());
```

注意：`Collectors.toList()` 不保证不可变；若需要不可变，用 `Collections.unmodifiableList(...)` 或 Guava/`List.copyOf`（后者需更高版本 JDK）。

### `Collectors.toMap` 的两个经典坑

```java
// ❌ value 为 null → NPE（内部走 Map.merge，禁止 null value）
Map<Long, String> map = users.stream()
    .collect(Collectors.toMap(User::getId, User::getNickname)); // nickname 可能 null

// ✅ 先过滤，或显式处理 null
Map<Long, String> map = users.stream()
    .filter(u -> u.getNickname() != null)
    .collect(Collectors.toMap(User::getId, User::getNickname));

// ❌ 重复 key → IllegalStateException
.collect(Collectors.toMap(User::getName, Function.identity()));

// ✅ 提供 merge function
.collect(Collectors.toMap(User::getName, Function.identity(), (a, b) -> a));
```

### 慎用 `parallelStream()`

```java
// ❌ 小集合 / I/O / 有共享可变状态 —— parallel 往往更慢或更危险
list.parallelStream().forEach(sharedList::add); // 竞态

// ❌ 在 parallel 里用有副作用的 forEach 写非并发集合
map.entrySet().parallelStream().forEach(e -> result.put(e.getKey(), e.getValue()));

// ✅ CPU 密集 + 无共享可变状态 + 数据量足够大时才考虑
// ✅ 收集用 collect(toMap/toConcurrentMap)，不要用 forEach 往外部 Map 塞
Map<K, V> result = list.parallelStream()
    .collect(Collectors.toConcurrentMap(Item::getKey, Item::getValue, (a, b) -> a));
```

并行流默认用 `ForkJoinPool.commonPool()`，会与同进程其他 parallel/CF 任务抢线程。

### 不要修改 Stream 源；避免嵌套 `forEach`

```java
// ❌ 管道执行中修改源 → ConcurrentModificationException
list.stream().peek(list::add).count();

// ❌ 嵌套 forEach 可读性差，且难做短路
a.forEach(x -> b.forEach(y -> ...));

// ✅ 用 flatMap / 常规循环表达笛卡尔或关联逻辑
```

### 原始流避免装箱

```java
// ❌ Stream<Integer> 装箱开销
int sum = list.stream().map(Order::getAmount).reduce(0, Integer::sum);

// ✅ IntStream / LongStream / DoubleStream
int sum = list.stream().mapToInt(Order::getAmount).sum();
```

---

## Optional

**设计意图**：作为**返回值**表达「可能没有」，不是通用 null 替代品。

```java
// ❌ 字段 / 参数 / 集合元素用 Optional（序列化、反射、API 噪音）
class User {
    private Optional<String> email;
}
void send(Optional<String> email) { ... }
Optional<List<Order>> findOrders(); // 空列表已能表达「没有」

// ✅ 仅作返回值；集合返回空集合
public Optional<User> findById(Long id) { ... }
public List<Order> findOrders(Long userId) { ... } // 没有则 emptyList
```

### 禁止 `isPresent()` + `get()` 当 null 检查用

```java
// ❌ 比 null 检查更啰嗦，且 get() 仍可能炸
if (userOpt.isPresent()) {
    return userOpt.get().getName();
}
return "Unknown";

// ✅ 函数式链式（Java 8 可用）
return userOpt.map(User::getName).orElse("Unknown");
```

### `orElse` vs `orElseGet`

```java
// ❌ orElse 参数会立刻求值（即使 Optional 有值）
return findUser(id).orElse(loadDefaultFromDb()); // 总是打 DB

// ✅ 昂贵默认值用 orElseGet
return findUser(id).orElseGet(this::loadDefaultFromDb);

// ✅ 必须有值时（Java 8）
return findUser(id).orElseThrow(() -> new UserNotFoundException(id));
// 注意：无参 orElseThrow() 是 Java 10+，Java 8 必须传 Supplier
```

### `of` vs `ofNullable`；嵌套用 `flatMap`

```java
// ❌ of(null) → 立刻 NPE
Optional.of(possiblyNull);

// ✅
Optional.ofNullable(possiblyNull);

// ❌ map 返回 Optional 造成 Optional<Optional<T>>
optional.map(this::findOther); // findOther 返回 Optional

// ✅
optional.flatMap(this::findOther);
```

Java 8 **没有** `Optional.stream()` / `ifPresentOrElse` / `or`（这些是 Java 9+）。过滤 Optional 集合时用：

```java
list.stream()
    .map(this::findUser)
    .filter(Optional::isPresent)
    .map(Optional::get) // 此处已 filter，可接受；或抽 helper
    .collect(Collectors.toList());
```

---

## Date/Time API (`java.time`)

遗留系统最常见的生产事故来源之一：继续用 `Date` / `Calendar` / `SimpleDateFormat`。

```java
// ❌ SimpleDateFormat 线程不安全，静态共享会错乱
private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd");

// ✅ DateTimeFormatter 不可变、线程安全
private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

### 选对类型

| 类型 | 用途 |
|------|------|
| `Instant` | 机器时间戳、审计、跨服务事件（UTC 时间线） |
| `LocalDate` | 只有日期（生日、营业日） |
| `LocalDateTime` | **无时区**的日期时间；不要用来表示「某事件发生时刻」 |
| `ZonedDateTime` / `OffsetDateTime` | 需要时区或 offset 的人类时间 |

```java
// ❌ 用 LocalDateTime 存「下单时间」——跨时区/DST 会歧义
private LocalDateTime createdAt = LocalDateTime.now();

// ✅ 事件时刻用 Instant；需要展示再转时区
private Instant createdAt = Instant.now();

// ❌ now() 依赖 JVM 默认时区，CI / 生产 / 开发机不一致
LocalDate.now();

// ✅ 显式 ZoneId，或注入 Clock 便于测试
LocalDate.now(ZoneOffset.UTC);
LocalDate.now(clock);
```

### 格式化陷阱

```java
// ❌ YYYY 是 week-based year，跨年周会错年
DateTimeFormatter.ofPattern("YYYY-MM-dd");

// ✅ 日历年用 yyyy
DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

与旧 API 互转：`date.toInstant()`、`Date.from(instant)`、`LocalDateTime.ofInstant(instant, zone)`。

---

## 并发：线程池与 CompletableFuture

Java 8 **没有虚拟线程**。I/O 密集靠合理的线程池，而不是 `newCachedThreadPool` 或无界队列硬扛。

```java
// ❌ 无界队列 + 默认拒绝策略，过载时延迟爆炸或 OOM
ExecutorService exec = Executors.newFixedThreadPool(8); // 队列无界

// ❌ 不 shutdown，线程泄漏
Executors.newFixedThreadPool(8).submit(task);

// ✅ 有界队列 + 明确拒绝策略 + 生命周期管理
ThreadPoolExecutor exec = new ThreadPoolExecutor(
    8, 16, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(500),
    new ThreadPoolExecutor.CallerRunsPolicy());
// 应用关闭时：shutdown → awaitTermination → shutdownNow
```

### CompletableFuture

```java
// ❌ I/O 任务丢进 commonPool（supplyAsync 无 Executor）
CompletableFuture.supplyAsync(() -> restTemplate.getForObject(url, Dto.class));

// ✅ 显式 I/O 线程池
CompletableFuture.supplyAsync(() -> callRemote(), ioExecutor);

// ❌ thenApply 里再返回 CF → 嵌套 CompletableFuture<CompletableFuture<T>>
.thenApply(id -> findAsync(id));

// ✅ 依赖异步用 thenCompose
.thenCompose(id -> findAsync(id));

// ❌ 在异步回调里 get()/join()，易饿死线程池甚至死锁
.thenApply(x -> other.join());

// ✅ 组合用 allOf / thenCombine；只在边界 join 一次
```

**超时**：Java 8 没有 `orTimeout` / `completeOnTimeout`（Java 9+）。需要超时就用 `get(timeout, unit)`，或自建 scheduler + `applyToEither` 完成异常。

**异常**：链路末端用 `exceptionally` / `handle` / `whenComplete`，避免异常被静默吞掉。

```java
// ❌ 共享可变的 SimpleDateFormat / HashMap 当缓存
private static final SimpleDateFormat SDF = ...;
private final Map<String, String> cache = new HashMap<>(); // 多线程 put

// ✅ ConcurrentHashMap；日期用 java.time
private final ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();
```

---

## Spring Boot 2

包名是 **`javax.*`**，不是 `jakarta.*`。审查时不要要求迁到 Jakarta，除非 PR 目标就是升级 Boot 3。

### 依赖注入

```java
// ❌ 字段 @Autowired：难测、依赖不透明
@Autowired
private UserRepository userRepo;

// ✅ 构造器注入（Boot 2 单构造器可省略 @Autowired）
private final UserRepository userRepo;

public UserService(UserRepository userRepo) {
    this.userRepo = userRepo;
}
```

### 配置

```java
// ❌ 密钥硬编码；@Value 散落各处
@Value("${app.payment.api-key}")
private String apiKey;

// ✅ @ConfigurationProperties（Java 8 用 class，不是 record）
@ConfigurationProperties(prefix = "app.payment")
public class PaymentProperties {
    private String apiKey;
    private int timeoutMs;
    // getters / setters
}
```

记得 `@EnableConfigurationProperties`；Boot **2.2+** 也可用 `@ConfigurationPropertiesScan`（`@SpringBootApplication` 默认会扫启动类所在包）。

### RestTemplate 必须设超时

默认 **无限等待**。下游挂死会占满 Tomcat / 业务线程。

```java
// ❌ 裸 new RestTemplate()，无超时
return new RestTemplate();

// ✅ Boot 2.1+ 用 Duration
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
        .setConnectTimeout(Duration.ofSeconds(2))
        .setReadTimeout(Duration.ofSeconds(5))
        .build();
}
```

若自定义 `ClientHttpRequestFactory`（如 Apache HttpClient），确认 factory 上也设置了 connect / read / connectionRequest 超时，避免 builder 超时被覆盖失效。

### 事务代理陷阱（Boot 2/3 相同）

```java
// ❌ 同类 self-call，@Transactional 不生效（JDK/CGLIB 代理拦不到）
public void create(Order o) {
    save(o); // 内部调用
}
@Transactional
public void save(Order o) { ... }

// ✅ 事务边界放在 public 入口；或拆到另一个 Bean
@Transactional
public void create(Order o) { saveInternal(o); }
```

`@Transactional` 加在 `private` 方法上同样无效。

---

## JPA / Hibernate 5

实体注解来自 `javax.persistence.*`。

### N+1

> 通用原理见 [N+1 查询跨语言指南](cross-cutting/n-plus-one-queries.md)

```java
// ❌ EAGER 或循环触发懒加载
@OneToMany(fetch = FetchType.EAGER)
private List<Order> orders;

for (User u : userRepo.findAll()) {
    u.getOrders().size(); // 懒加载时 N 次查询
}

// ✅ JOIN FETCH / @EntityGraph；默认保持 LAZY
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### 事务与只读

```java
// ❌ Controller 开事务；或 private 上 @Transactional
// ✅ Service 公共方法；读操作加 readOnly
@Transactional(readOnly = true)
public User get(Long id) { ... }
```

### Entity 与 Lombok

```java
// ❌ @Data 生成的 equals/hashCode 易拖入懒加载字段
@Entity
@Data
public class User { ... }

// ✅ @Getter/@Setter；equals/hashCode 基于稳定业务键，或「仅当 id 非 null」的 id 比较
// ⚠️ 不要用含懒加载关联的全部字段；新建未持久化实体 id 均为 null，纯 id equals 会把它们判成不相等（通常可接受）
@Entity
@Getter
@Setter
public class User {
    @Id
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return id != null && id.equals(((User) o).id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

### 时间字段

Hibernate 5 对 `java.time` 支持可用，但 `ZonedDateTime` 映射到无时区 `TIMESTAMP` 时会按 JVM 时区归一化，跨区易出偏差。优先：

- 库内 UTC：`Instant` 或 `OffsetDateTime`
- 配置 `spring.jpa.properties.hibernate.jdbc.time_zone=UTC`（若团队约定）
- 遗留 `java.util.Date` 字段迁移时再谈；新代码不要新增 `Date` / `@Temporal`

---

## 异常处理

Boot 2 没有 Spring 6 的 `ProblemDetail` 一等公民支持（那是 Boot 3 叙事）。用统一 `@ControllerAdvice` + 明确 HTTP 状态即可。

```java
// ❌ 吞异常、printStackTrace、返回 null 掩盖失败
try {
    userService.create(user);
} catch (Exception e) {
    e.printStackTrace();
    return null;
}

// ✅ 业务异常 + 全局处理
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(UserNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ApiError("USER_NOT_FOUND", e.getMessage()));
    }
}
```

资源关闭用 try-with-resources（Java 7+），不要手写 `finally { close() }` 还漏 null 判断。

---

## 测试

```java
// ❌ 凡测必 @SpringBootTest（慢、脆）
@SpringBootTest
public class UserServiceTest { ... }

// ✅ 纯单元：JUnit 4/5 + Mockito
@RunWith(MockitoJUnitRunner.class) // JUnit 4
// 或 @ExtendWith(MockitoExtension.class) // JUnit 5
public class UserServiceTest {
    @Mock private UserRepository repo;
    @InjectMocks private UserService service;

    @Test
    public void shouldCreateUser() { ... }
}
```

时间相关逻辑注入 `Clock`，避免 `Instant.now()` 写死导致无法断言。

遗留栈常见 JUnit 4；若已混用 JUnit 5，同一模块不要两套 Runner 风格混到无法维护。

---

## Review Checklist

### 版本与范围
- [ ] 已确认是 Java 8 / Boot 2 / `javax.*`，未用 Java 17+ API 或 `jakarta.*` 强行要求
- [ ] 未把「改用 Record / 虚拟线程 / 文本块」当作 blocking 意见

### 语言特性
- [ ] Lambda 短小；优先方法引用与标准函数式接口
- [ ] Stream 用于转换/过滤/归约，而非简单副作用循环
- [ ] 收集使用 `Collectors.*`（无 `Stream.toList()`）
- [ ] `toMap` 处理了 null value 与重复 key
- [ ] `parallelStream` 有充分理由，且无共享可变状态
- [ ] Optional 仅作返回值；无 `isPresent`+`get` 滥用；昂贵默认用 `orElseGet`
- [ ] 日期用 `java.time`；无共享 `SimpleDateFormat`；类型选对（`Instant` vs `LocalDateTime`）
- [ ] 格式化用 `yyyy` 而非 week-based `YYYY`（除非真要周历年）

### 并发
- [ ] 线程池有界、可关闭；I/O 不用 `ForkJoinPool.commonPool()`
- [ ] CF 传了显式 Executor；`thenCompose` 用于嵌套异步；有超时与异常处理
- [ ] 共享状态用并发集合；无静态可变 `DateFormat`

### Spring Boot 2 / JPA
- [ ] 构造器注入；`@ConfigurationProperties` 收拢配置
- [ ] `RestTemplate`（及 RequestFactory）设置了 connect/read 超时
- [ ] `@Transactional` 在 public 入口，无 self-invocation / private 失效问题
- [ ] 无 N+1；Entity 不用 `@Data`；时间字段策略明确（UTC）
- [ ] 包名保持 `javax.*` 一致，无 javax/jakarta 混用

### 质量
- [ ] 异常不吞；统一错误响应
- [ ] try-with-resources 管理 I/O 与 DB 资源
- [ ] 核心逻辑有单元测试；时间可注入 `Clock`

---

## References

- [What to Look for in Java 8 Code (JetBrains)](https://blog.jetbrains.com/upsource/2016/08/03/what-to-look-for-in-java-8-code/)
- [JDK-8148463: Collectors.toMap fails on null values](https://bugs.openjdk.org/browse/JDK-8148463)
- [Oracle Tutorial: Parallelism](https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html)
- [Baeldung: Migrating to Java 8 Date/Time API](https://www.baeldung.com/migrating-to-java-8-date-time-api)
- [Baeldung: CompletableFuture and ThreadPool](https://www.baeldung.com/java-completablefuture-threadpool)
- [Spring Boot RestTemplate customization](https://docs.spring.io/spring-boot/docs/2.7.x/reference/html/io.html#io.rest-client.resttemplate)
- [Thorben Janssen: Hibernate/JPA Date and Time](https://thorben-janssen.com/hibernate-jpa-date-and-time/)
