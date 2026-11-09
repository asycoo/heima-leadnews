# Feign 远程调用工程实现

> 讲义：day04（审核调 article + 降级）、day05（schedule）、day10（article 调 wemedia）  
> 代码参考：day12 / day13 · `heima-leadnews-feign-api`

---

## 总流程

```text
① feign-api：定义 @FeignClient 接口（路径、参数与目标 Controller 一致）
② 被调方微服务：写 @RestController 实现该接口（真正业务）
③ 调用方微服务：pom 依赖 feign-api + 启动类 @EnableFeignClients
④ 业务里 @Autowired IXxxClient，像本地方法一样调用
⑤ Feign 按服务名从 Nacos 找实例，发 HTTP 请求
```

典型例子：审核通过后 wemedia 调 article 保存 APP 文章。

```text
WmNewsAutoScanServiceImpl
  → articleClient.saveArticle(dto)
  → Feign → Nacos 解析 leadnews-article
  → ArticleClient#saveArticle → ApArticleService
```

---

## 1. 工程结构：独立模块 `heima-leadnews-feign-api`

接口集中放一个模块，调用方 / 被调方都依赖它，避免互相依赖对方整个 service。

```text
heima-leadnews-feign-api
├── pom.xml          # openfeign + model + common
└── com.heima.apis
    ├── article/IArticleClient (+ fallback/)
    ├── user/IUserClient
    ├── wemedia/IWemediaClient
    └── schedule/IScheduleClient
```

`pom` 核心依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

`heima-leadnews-service` 父 POM 已统一引入 `heima-leadnews-feign-api`，下面各微服务一般不用再单独写一遍。

---

## 2. 定义远程接口（api 侧）

```java
@FeignClient(value = "leadnews-article", fallback = IArticleClientFallback.class)
public interface IArticleClient {

    @PostMapping("/api/v1/article/save")
    ResponseResult saveArticle(@RequestBody ArticleDto dto);
    // ...
}
```

要点：

| 项 | 说明 |
| --- | --- |
| `value` / `name` | **Nacos 里的服务名**，不是 URL、不是包名 |
| 方法上的 `@PostMapping` 等 | 必须和被调方 Controller 路径一致 |
| `@RequestBody` / `@PathVariable` / `@RequestParam` | 与服务端注解对齐，否则参数绑不上 |
| `fallback` | 可选；失败/超时走降级类 |

项目里已有客户端：

| 接口 | 服务名 | 主要用途 |
| --- | --- | --- |
| `IArticleClient` | `leadnews-article` | 保存文章、统计、评论配置等 |
| `IUserClient` | `leadnews-user` | 按 id 查 APP 用户 |
| `IWemediaClient` | `leadnews-wemedia` | 自媒体用户、频道 |
| `IScheduleClient` | `leadnews-schedule` | 延迟任务增删拉 |

---

## 3. 被调方：实现接口 = 对外 HTTP API

在**提供服务的微服务**里建 `feign` 包，`@RestController` + `implements IXxxClient`：

```java
// heima-leadnews-article
@RestController
public class ArticleClient implements IArticleClient {

    @Autowired
    private ApArticleService apArticleService;

    @Override
    @PostMapping("/api/v1/article/save")
    public ResponseResult saveArticle(@RequestBody ArticleDto dto) {
        return apArticleService.saveArticle(dto);
    }
}
```

同样模式：`UserClient`、`WemediaClient` 等。

注意：实现类上的 Mapping **再写一遍**，和接口声明保持一致；Feign 调的是这个 HTTP 接口，不是直接调 Java 实现类。

---

## 4. 调用方：怎么「注入进来」

不是手工 `new`，也不是 XML。分三步：

### 4.1 依赖 api 模块

service 父工程已依赖 `heima-leadnews-feign-api` → 类路径上有 `IArticleClient` 等接口 + openfeign。

### 4.2 启动类扫描 Feign 接口

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages = "com.heima.apis")  // 必须扫到 api 包
public class WemediaApplication { ... }
```

`basePackages` 指向 `com.heima.apis`（或其子包）。  
Spring 会为每个 `@FeignClient` 生成代理 Bean，才能 `@Autowired`。

day13 里 wemedia 写成：

```java
@EnableFeignClients(basePackages = {"com.heima.apis", "com.heima.apis.user"})
```

扫 `com.heima.apis` 本已含子包；多写一个不影响。

### 4.3 业务注入接口

```java
@Autowired
private IArticleClient articleClient;

ResponseResult responseResult = articleClient.saveArticle(dto);
```

注入的是 **Feign 代理**，不是 article 服务里的 `ArticleClient` 实现类。  
代理内部：拼 HTTP → Ribbon/LoadBalancer 选实例 → 发请求 → 反序列化结果。

---

## 5. 配置

### 5.1 服务发现（前提）

调用方、被调方都要注册 Nacos，`@EnableDiscoveryClient`。  
Feign 的 `value = "leadnews-article"` 靠注册中心解析成具体 IP:端口。

### 5.2 Feign / 超时 / 降级（调用方，如 wemedia）

`bootstrap.yml` 或 Nacos 配置：

```yaml
feign:
  hystrix:
    enabled: true          # 开启后 fallback 才会生效（课程用的 Hystrix）
  client:
    config:
      default:
        connectTimeout: 2000
        readTimeout: 2000  # 超时易触发降级
```

### 5.3 降级类如何进容器

fallback 写在 **feign-api** 里，且加了 `@Component`：

```java
@Component
public class IArticleClientFallback implements IArticleClient {
    @Override
    public ResponseResult saveArticle(ArticleDto dto) {
        return ResponseResult.errorResult(AppHttpCodeEnum.SERVER_ERROR, "获取数据失败");
    }
}
```

调用方默认不一定扫到 `com.heima.apis.article.fallback`，所以 wemedia 额外：

```java
@Configuration
@ComponentScan("com.heima.apis.article.fallback")
public class InitConfig {
}
```

再在接口上声明：

```java
@FeignClient(value = "leadnews-article", fallback = IArticleClientFallback.class)
```

未开 `feign.hystrix.enabled=true` 时，fallback 通常不生效。

---

## 6. 角色对照（一次调用里谁干什么）

以「审核通过保存 APP 文章」为例：

| 角色 | 位置 | 做什么 |
| --- | --- | --- |
| 接口定义 | feign-api `IArticleClient` | 约定服务名 + 路径 |
| 调用方 | wemedia `WmNewsAutoScanServiceImpl` | `@Autowired` 后调用 |
| 代理 | Spring Cloud OpenFeign 生成 | 发 HTTP |
| 被调方 | article `ArticleClient` | 真正 `saveArticle` |
| 降级 | `IArticleClientFallback` | article 挂了/超时返回错误结果 |

```text
wemedia ──Feign──► Nacos(leadnews-article) ──HTTP──► article/ArticleClient
   │                                                      │
   └── 失败且开启 hystrix ──► IArticleClientFallback ◄────┘
```

---

## 7. 接入检查清单

调用方：

1. 依赖 `heima-leadnews-feign-api`  
2. `@EnableFeignClients(basePackages = "com.heima.apis")`  
3. 注册 Nacos  
4. 需要降级：`feign.hystrix.enabled=true` + `@ComponentScan` fallback 包  
5. `@Autowired IXxxClient`

被调方：

1. 实现 `IXxxClient` 的 `@RestController`  
2. Mapping / 参数注解与 api 接口一致  
3. 注册到 Nacos，服务名 = `@FeignClient` 的 value  

---

## 一句话

**接口放 feign-api；被调方实现成 Controller；调用方 EnableFeignClients 后注入接口；Feign 按服务名走 Nacos 发 HTTP；降级靠 hystrix + fallback 组件扫描。**
