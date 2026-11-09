# MinIO + Freemarker 工程实现与业务串联

> 讲义：day02（Freemarker、MinIO starter、文章静态化） / day03（素材上传） / day04（审核下载图片）  
> 代码参考：day12 / day13 · `heima-file-starter`、`heima-leadnews-article`

---

## 总流程

```text
【文件能力】
微服务引入 heima-file-starter + minio.*
  → 自动装配 FileStorageService / MinioClient
  → upload / download / delete

【文章详情静态化】
文章内容 JSON
  → Freemarker（article.ftl）渲染成 html 字符串
  → uploadHtmlFile 上传 MinIO
  → 写入 ap_article.static_url
  → APP 详情直接打开该 URL（不再实时查库渲染）
```

MinIO 概念对照：

| MinIO | 类比 |
| --- | --- |
| bucket | 目录（项目固定 `leadnews`，控制台先建好） |
| object / key | 文件路径，如 `2021/04/26/xxx.jpg` |
| endpoint | SDK 连接地址（上传/下载/删除走它） |
| readPath | 拼给前端/库表的访问前缀 |

---

## 1. MinIO starter：`heima-file-starter`

```text
heima-file-starter
├── config
│   ├── MinIOConfigProperties   # 绑定 minio.*
│   └── MinIOConfig             # 创建 MinioClient Bean
├── service
│   ├── FileStorageService
│   └── impl/MinIOFileStorageService
└── META-INF/spring.factories
```

### 1.1 配置项

```yaml
minio:
  accessKey: minio
  secretKey: minio123
  bucket: leadnews
  endpoint: http://192.168.200.130:9000
  readPath: http://192.168.200.130:9000
```

| 字段 | 干什么 | 注意 |
| --- | --- | --- |
| `bucket` | put/get 指定桶 | 控制台桶名必须一致 |
| `endpoint` | `MinioClient` 连接地址；delete/downLoad 用它从全路径 `replace` 前缀 | |
| `readPath` | 上传成功后拼对外 URL | 常与 endpoint 相同；若不一致，delete/downLoad 仍按 endpoint 截，可能解路径失败 |
| `accessKey` / `secretKey` | 鉴权 | 与 docker 启动环境变量一致 |

返回全路径：

```text
{readPath}/{bucket}/{yyyy/MM/dd}/{filename}
例：http://192.168.200.130:9000/leadnews/2021/04/26/xxx.html
```

### 1.2 `@ConditionalOnClass` + 自动装配

```java
@Configuration
@EnableConfigurationProperties({MinIOConfigProperties.class})
@ConditionalOnClass(FileStorageService.class)  // classpath 有该接口才注册 MinioClient
public class MinIOConfig {
    @Bean
    public MinioClient buildMinioClient() {
        return MinioClient.builder()
                .credentials(accessKey, secretKey)
                .endpoint(endpoint)  // 只用 endpoint
                .build();
    }
}
```

`spring.factories` 注册 `MinIOFileStorageService`，其上 `@Import(MinIOConfig.class)`：

```text
spring.factories
  → MinIOFileStorageService
  → @Import MinIOConfig（ConditionalOnClass 通过）
  → 绑定 minio.* → MinioClient
  → 业务 @Autowired FileStorageService
```

### 1.3 接口

| 方法 | 用途 |
| --- | --- |
| `uploadImgFile` | 素材图，`contentType=image/jpg` |
| `uploadHtmlFile` | 文章静态页，`contentType=text/html` |
| `downLoadFile` | 审核时按全路径拉字节 |
| `delete` | 按全路径删 |

对象路径：`[prefix/] + yyyy/MM/dd/ + filename`（业务常传 prefix=`""`）。

### 1.4 微服务怎么用

```text
1. pom 引入 heima-file-starter
2. 配置 minio.*
3. @Autowired FileStorageService
```

| 服务 | 场景 |
| --- | --- |
| wemedia | `uploadImgFile` 素材；`downLoadFile` 审核拉图 |
| article | Freemarker 生成 html 后 `uploadHtmlFile` |

---

## 2. Freemarker：把文章内容渲染成静态 HTML

### 2.1 为什么用它（day02 方案）

| 方案 | 做法 | 问题 |
| --- | --- | --- |
| 方案一 | 详情实时查 `ap_article_content`，服务端/前端再渲染 | 高并发下库压力大 |
| 方案二（采用） | 发布时用模板生成 html → 存 MinIO → 详情只访问 `static_url` | 读多写少更合适 |

Freemarker 角色：**模板 + 数据模型 → 输出 html 文本**（不是给用户当页面引擎直接访问，而是后端生成文件）。

### 2.2 article 服务接入

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
<dependency>
    <groupId>com.heima</groupId>
    <artifactId>heima-file-starter</artifactId>
</dependency>
```

模板位置（Boot 默认）：

```text
heima-leadnews-article/src/main/resources/templates/article.ftl
```

测试工程 `freemarker-demo` 里常见配置（理解用；article 用默认即可）：

```yaml
spring:
  freemarker:
    cache: false
    settings:
      template_update_delay: 0
    suffix: .ftl
    template-loader-path: classpath:/templates
```

注入的是 Freemarker 的 `Configuration`（Boot 自动配置），用来 `getTemplate("article.ftl")`。

### 2.3 模板怎么吃数据：`article.ftl`

文章内容在库里是 JSON 数组，大致：

```json
[
  {"type":"text","value":"一段文字"},
  {"type":"image","value":"http://.../xxx.jpg"}
]
```

Java 侧放进数据模型：

```java
Map<String,Object> contentDataModel = new HashMap<>();
contentDataModel.put("content", JSONArray.parseArray(content));
template.process(contentDataModel, out);
```

模板里用 FTL 遍历（静态部分用 Freemarker 写死；点赞/评论等仍是页面里的 Vue）：

```ftl
<#if content??>
  <#list content as item>
    <#if item.type='text'>
      <van-col ...>${item.value}</van-col>
    <#else>
      <van-image src="${item.value}"></van-image>
    </#if>
  </#list>
</#if>
```

模板里还会引用 css/js（如 `../../../plugins/css/index.css`），讲义要求把 `index.css` / `index.js` **手动上传到同一 MinIO bucket**，否则静态页样式/脚本 404。

### 2.4 核心代码：`ArticleFreemarkerServiceImpl`

触发点：`ApArticleServiceImpl` 保存/更新文章后：

```java
// 异步生成静态文件并上传 MinIO
articleFreemarkerService.buildArticleToMinIO(apArticle, dto.getContent());
```

实现步骤：

```text
1. content 非空
2. configuration.getTemplate("article.ftl")
3. 数据模型 put("content", JSONArray.parseArray(content))
4. template.process → StringWriter 得到 html 字符串
5. 转 InputStream，uploadHtmlFile("", articleId + ".html", in)
6. 更新 ap_article.static_url = 返回的 MinIO 全路径
7. （后续）发 Kafka 同步 ES，带上 staticUrl
```

```java
@Async
public void buildArticleToMinIO(ApArticle apArticle, String content) {
    Template template = configuration.getTemplate("article.ftl");
    Map<String,Object> contentDataModel = new HashMap<>();
    contentDataModel.put("content", JSONArray.parseArray(content));
    StringWriter out = new StringWriter();
    template.process(contentDataModel, out);

    InputStream in = new ByteArrayInputStream(out.toString().getBytes());
    String path = fileStorageService.uploadHtmlFile("", apArticle.getId() + ".html", in);

    apArticleService.update(Wrappers.<ApArticle>lambdaUpdate()
            .eq(ApArticle::getId, apArticle.getId())
            .set(ApArticle::getStaticUrl, path));
}
```

细节：

- `@Async`：不堵保存主流程；article 启动类要有 `@EnableAsync`
- 文件名用 `文章id.html`，便于对应
- day02 早期用测试类 `ArticleFreemarkerTest` 手动跑；后来接到保存文章链路自动生成

### 2.5 Freemarker ↔ MinIO 分工

| 组件 | 负责 |
| --- | --- |
| Freemarker | 模板 + 内容 JSON → **内存中的 html** |
| MinIO | 把 html **持久化**成可访问文件，返回 URL |
| `static_url` | APP 详情入口；列表/搜索也会带上 |

没有 Freemarker：只有原始 JSON，不能直接当详情页。  
没有 MinIO：html 只能落本地盘，不好给 APP/CDN 访问。

---

## 3. 需求串联

### 3.1 素材上传（wemedia）

```text
MultipartFile → uploadImgFile → 存 wm_material.url（MinIO 全路径）
```

### 3.2 审核拉图（wemedia）

```text
文章里的图片 url → downLoadFile → byte[] → OCR / 图片审核
```

`downLoadFile` 用 `endpoint` 剥前缀再拆 `bucket` + `object`，故库里 url 前缀要和 endpoint 对齐。

### 3.3 发布后静态化（article = Freemarker + MinIO）

```text
审核通过 → Feign 保存 APP 文章（内容进 ap_article_content）
  → buildArticleToMinIO
  → ftl 渲染 → uploadHtmlFile → static_url
  → APP 打开详情：请求 static_url（MinIO 上的 html）
```

### 3.4 整条链

```text
上传素材 ──uploadImg──► MinIO 图片
        │
        ▼ 写文章引用 url
审核 ──downLoadFile──► 取图审核
        │
        ▼ 通过 → 保存文章
Freemarker(article.ftl) ──生成 html──► uploadHtml ──► MinIO
        │
        ▼ static_url
APP 详情直出静态页（css/js 也在 MinIO）
```

---

## 4. 细节速查

**MinIO**

1. 桶 `leadnews` 先建；`bucket` 配置一致  
2. endpoint 连服务 / 解路径；readPath 拼对外 URL  
3. `@ConditionalOnClass(FileStorageService)` + `spring.factories` 控制装配  
4. 库表存全 URL，不是相对 object 名  

**Freemarker**

5. 模板在 `classpath:/templates/article.ftl`  
6. 数据模型 key 为 `content`，值是内容 JSON 数组  
7. `template.process` 产出字符串，再交给 `uploadHtmlFile`  
8. `@Async` + `@EnableAsync`；`static_url` 指向 MinIO html  
9. `index.css` / `index.js` 需预先上传到 MinIO，与模板引用路径匹配  

---

## 一句话

**Freemarker 负责「内容 → html」；MinIO starter 负责「html/图片存取与 URL」；文章详情用 `static_url` 直出 MinIO，素材与审核则走 upload/download。**
