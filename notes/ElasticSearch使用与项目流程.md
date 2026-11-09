# 搜索模块：ElasticSearch + MongoDB

> 讲义：day07《app端文章搜索》  
> 代码：`heima-leadnews-search`、`es-init`；审核通过后 Freemarker → Kafka → ES

---

## 0. 先建立画面：ES 和 Mongo 各干什么？

APP 搜索相关其实有两块需求：

1. **搜文章**：关键词、高亮、按时间往下翻 → 用 **ElasticSearch**  
2. **搜的周边**：每人一份搜索历史、输入时联想词 → 用 **MongoDB**

为什么文章不直接 MySQL `LIKE`？数据一大就慢，中文分词/高亮也不擅长。  
为什么历史不放 MySQL？每人一份、量大、读写简单，文档型 Mongo 更合适。

```text
MySQL（article）  = 文章业务真相
ElasticSearch     = 文章搜索专用副本
MongoDB           = 搜索历史 + 联想词（按用户/词条存文档）
```

流程上记住：

1. **搜文章**：`leadnews-search` 查 ES 索引 `app_info_article`  
2. **同步文章进 ES**：审核通过并静态化后，Kafka 通知 search 写入  
3. **搜的时候顺带**：登录用户异步把关键词写入 Mongo 历史；输入框联想词也查 Mongo  

ES 名词小抄：

| 词 | 人话 |
| --- | --- |
| 索引 index | 「搜索专用表」，本项目 `app_info_article` |
| 文档 document | 一篇文章一条，id=雪花文章 id |
| 映射 mapping | 字段类型；text 要分词，keyword 可只存不搜 |
| IK / ik_smart | 中文分词器 |
| RestHighLevelClient | Java 调 ES（7.4） |

Mongo 名词小抄：

| 词 | 人话 |
| --- | --- |
| 集合 collection | 类似表；`ap_user_search`、`ap_associate_words` |
| 文档 | 一条记录；用 `@Document("集合名")` 映射 |
| MongoTemplate | Spring 操作 Mongo 的模板类 |

---

## 1. 环境搭建（使用前）

### 1.1 Docker 起 ES 7.4.0

```shell
docker pull elasticsearch:7.4.0

docker run -id --name elasticsearch -d --restart=always \
  -p 9200:9200 -p 9300:9300 \
  -v /usr/share/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
  -e "discovery.type=single-node" \
  elasticsearch:7.4.0
```

- `9200`：HTTP API（Postman / 客户端都走这）  
- `single-node`：单机开发，不组集群  

### 1.2 装 IK 中文分词（版本要对上 7.4.0）

把 `elasticsearch-analysis-ik-7.4.0.zip` 解压到容器映射的 plugins 目录，例如：

```shell
cd /usr/share/elasticsearch/plugins
mkdir analysis-ik && cd analysis-ik
# 放入 zip 后
unzip elasticsearch-analysis-ik-7.4.0.zip
```

重启 ES 后，mapping 里才能写 `"analyzer":"ik_smart"`。

### 1.3 创建索引 + mapping

Postman：`PUT http://192.168.200.130:9200/app_info_article`

```json
{
  "mappings": {
    "properties": {
      "id": { "type": "long" },
      "publishTime": { "type": "date" },
      "layout": { "type": "integer" },
      "images": { "type": "keyword", "index": false },
      "staticUrl": { "type": "keyword", "index": false },
      "authorId": { "type": "long" },
      "authorName": { "type": "text" },
      "title": { "type": "text", "analyzer": "ik_smart" },
      "content": { "type": "text", "analyzer": "ik_smart" }
    }
  }
}
```

要点：

- `title` / `content`：`text` + IK → 可按关键词搜  
- `images` / `staticUrl`：`keyword` 且 `index:false` → **只存给前端用，不参与检索**  
- 点进详情仍走 `staticUrl`（MinIO 静态页），和搜索职责分开  

常用检查：

```text
GET  /app_info_article              # 看 mapping
GET  /app_info_article/_search      # 查文档
DELETE /app_info_article            # 删索引（慎用）
```

---

## 2. 项目里怎么配置 ES

### 2.1 依赖（search 服务）

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.4.0</version>
</dependency>
<!-- 以及 elasticsearch / elasticsearch-rest-client，版本对齐 7.4.0 -->
```

### 2.2 Nacos / yml 配置

```yaml
spring:
  autoconfigure:
    exclude: org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
    # search 主存 ES/Mongo，可排除 JDBC
elasticsearch:
  host: 192.168.200.130
  port: 9200
```

search 的 `bootstrap.yml` 里还有 Nacos、端口 `51804`、服务名 `leadnews-search`。

### 2.3 客户端 Bean

`ElasticSearchConfig`：

```java
@Configuration
@ConfigurationProperties(prefix = "elasticsearch")
public class ElasticSearchConfig {
    private String host;
    private int port;

    @Bean
    public RestHighLevelClient client() {
        return new RestHighLevelClient(RestClient.builder(
                new HttpHost(host, port, "http")));
    }
}
```

业务里 `@Autowired RestHighLevelClient` 即可。

---

## 3. ES 在项目流程中怎么用

整条链路：

```text
① 一次性：es-init 把历史文章 bulk 进 ES
② 日常写入：审核通过 → 静态化 → Kafka → search 写入/更新文档
③ 日常查询：APP 搜关键词 → search 服务查 ES → 列表带高亮和 staticUrl
```

### 3.1 历史数据初始化（es-init）

模块：`heima-leadnews-test/es-init`  

思路：查库拼 `SearchArticleVo` 列表 → `BulkRequest` 批量 `IndexRequest` 写入 `app_info_article`。

```text
for 每篇文章:
  IndexRequest.id(文章id).source(json)
bulk 提交
```

只在环境刚搭好、索引空的时候跑；之后靠 Kafka 增量同步。

### 3.2 增量同步（审核通过之后）

文章保存并 Freemarker 上传 MinIO 后：

```text
ArticleFreemarkerServiceImpl
  → kafkaTemplate.send(ARTICLE_ES_SYNC_TOPIC, SearchArticleVo的json)
```

search 侧监听：

```java
@KafkaListener(topics = ArticleConstants.ARTICLE_ES_SYNC_TOPIC)
public void onMessage(String message) {
    SearchArticleVo vo = JSON.parseObject(message, SearchArticleVo.class);
    IndexRequest indexRequest = new IndexRequest("app_info_article");
    indexRequest.id(vo.getId().toString());
    indexRequest.source(message, XContentType.JSON);
    restHighLevelClient.index(indexRequest, RequestOptions.DEFAULT);
}
```

含义：以文章 id 为文档 id，**有则覆盖、无则新增**，搜索库跟上发布。

和审核总流程的位置：

```text
审核通过 → Feign saveArticle → 雪花 id
         → 异步静态化 MinIO
         → Kafka 同步 ES   ← 这里
APP 搜索 → 命中文档 → 点详情打开 staticUrl
```

### 3.3 搜索查询（核心用法）

接口：`POST /api/v1/article/search/search`  
实现：`ArticleSearchServiceImpl#search`

入参大致：`searchWords`、`pageSize`、`minBehotTime`（往下翻用时间游标）。

查询拼装（人话 + 代码对应）：

```text
索引：app_info_article

must：关键字在 title、content 上 query_string（OR，会走 IK 分词）
filter：publishTime < minBehotTime（只取更早的，做「加载更多」）
分页：from=0，size=pageSize（配合时间条件，不是传统页码深翻）
排序：publishTime 倒序
高亮：title，前后包上红色 font 标签
```

关键代码骨架：

```java
SearchRequest searchRequest = new SearchRequest("app_info_article");
BoolQueryBuilder bool = QueryBuilders.boolQuery();
bool.must(QueryBuilders.queryStringQuery(dto.getSearchWords())
        .field("title").field("content").defaultOperator(Operator.OR));
bool.filter(QueryBuilders.rangeQuery("publishTime").lt(dto.getMinBehotTime().getTime()));

SearchSourceBuilder builder = new SearchSourceBuilder();
builder.query(bool);
builder.size(dto.getPageSize());
builder.sort("publishTime", SortOrder.DESC);
// HighlightBuilder 配 title ...
restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
```

结果处理：把 hit 的 `_source` 转 Map；有高亮用高亮片段做 `h_title`，否则用原标题。  
列表字段里带有 `staticUrl`，点进去仍是 MinIO 静态详情。

搜文章的同时：若已登录且是第一页（`fromIndex==0`），会 **`@Async` 调 `ApUserSearchService.insert`** 把关键词写入 Mongo 历史（见下一节）。

---

## 4. MongoDB：搜索历史 + 联想词

### 4.1 为什么历史用 Mongo？

- 每个用户都有一份搜索记录，量大  
- 结构简单（用户 id + 关键词 + 时间），加载要快  
- 不适合硬塞进关系库做复杂关联  

讲义要求：展示最近 **10 条**、按时间倒序；可删；超过 10 条顶掉最久的。

### 4.2 环境与配置

安装（Docker）：

```shell
docker pull mongo
docker run -di --name mongo-service --restart=always \
  -p 27017:27017 \
  -v ~/data/mongodata:/data \
  mongo
```

search 服务依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

Nacos / yml（讲义库名 `leadnews-history`）：

```yaml
spring:
  data:
    mongodb:
      host: 192.168.200.130
      port: 27017
      database: leadnews-history
```

操作入口：注入 `MongoTemplate`。

### 4.3 集合与实体

| 集合 | 实体 | 用途 |
| --- | --- | --- |
| `ap_user_search` | `ApUserSearch` | 用户搜索历史 |
| `ap_associate_words` | `ApAssociateWords` | 联想词词库 |

```java
@Document("ap_user_search")
public class ApUserSearch {
    private String id;
    private Integer userId;
    private String keyword;
    private Date createdTime;
}

@Document("ap_associate_words")
public class ApAssociateWords {
    private String id;
    private String associateWords;
    private Date createdTime;
}
```

### 4.4 搜索历史怎么写 / 查 / 删

触发写：`ArticleSearchServiceImpl#search` 里：

```java
if (user != null && dto.getFromIndex() == 0) {
    apUserSearchService.insert(dto.getSearchWords(), user.getId());
}
```

`insert`（`@Async`，不堵搜索主流程）：

```text
1. 同用户 + 同 keyword 已存在 → 只更新 createdTime（提到最新）
2. 不存在：
     该用户历史 < 10 条 → 直接 save
     已满 10 条 → findAndReplace 最久那条（按 createdTime 倒序列表的最后一条）
```

查列表：`POST /api/v1/history/load`  
→ 按当前登录 `userId` 查，`createdTime` 倒序返回。

删一条：`POST /api/v1/history/del`  
→ 校验登录后 `remove(userId + id)`。

用户 id 来自网关透传的 `userId` → `AppTokenInterceptor` → `AppThreadLocalUtil`（和 APP 鉴权同一套路）。

### 4.5 联想词

输入框边输边提示：`POST /api/v1/associate/search`

```text
入参 searchWords
Mongo：associateWords 正则模糊匹配
limit pageSize（最大 20）
返回匹配到的联想词列表
```

```java
Query query = Query.query(
    Criteria.where("associateWords").regex(".*?\\" + dto.getSearchWords() + ".*"));
query.limit(dto.getPageSize());
mongoTemplate.find(query, ApAssociateWords.class);
```

词库数据一般预置在 Mongo（资料导入）；不是 ES 搜文章的结果。

### 4.6 和 ES 在同一次「搜索」里怎么配合

```text
用户输入关键词点搜索
  ├─ ES：查 app_info_article → 文章列表（高亮、staticUrl）
  └─ Mongo：异步记一条 ap_user_search（登录且首页）

用户看历史 / 删历史 → 只打 Mongo
用户输入联想 → 只打 Mongo（ap_associate_words）
```

---

## 5. 配置与使用对照表

| 项 | 值 / 位置 |
| --- | --- |
| ES 版本 | 7.4.0 |
| ES 地址 | `elasticsearch.host/port` → 一般 `192.168.200.130:9200` |
| ES 客户端 | `RestHighLevelClient`（`ElasticSearchConfig`） |
| 索引名 | `app_info_article` |
| 分词 | `ik_smart`（title、content） |
| 批量初始化 | `es-init` BulkRequest |
| 增量同步 Topic | `article.es.sync.topic` |
| 同步消费者 | `SyncArticleListener` |
| Mongo | `spring.data.mongodb.*`，库 `leadnews-history` |
| Mongo 客户端 | `MongoTemplate` |
| 历史集合 | `ap_user_search` |
| 联想词集合 | `ap_associate_words` |
| 搜索服务 | `leadnews-search` :51804 |

---

## 6. 和别的组件怎么分工（别混）

| 组件 | 管什么 |
| --- | --- |
| MySQL article | 文章业务数据 |
| MinIO + staticUrl | 详情页静态 HTML |
| ES | 文章关键词检索、高亮、按时间筛 |
| Kafka | 发布后把文章「抄」进 ES |
| Mongo | 搜索历史、联想词 |

---

## 7. 快速自问

1. 为什么不直接查 MySQL 搜文章？→ 性能与中文分词/高亮。  
2. mapping 里 images 为何 `index:false`？→ 只需返回展示，不参与搜。  
3. 新文章何时进 ES？→ 静态化后 Kafka；老数据靠 es-init。  
4. 为什么历史用 Mongo？→ 按用户存、量大、结构简单。  
5. 历史为何最多 10 条、还要 `@Async`？→ 产品限制条数；异步避免拖慢搜文章。  
6. 联想词和 ES 搜索是一回事吗？→ 不是；联想查 Mongo 词库，文章列表查 ES。

---

## 一句话

**ES 管「搜出哪些文章」；Mongo 管「搜过什么 / 输入联想」；search 服务两者都配：RestHighLevelClient 查 `app_info_article`，MongoTemplate 维护 `ap_user_search` 与 `ap_associate_words`；文章增量靠审核后 Kafka 同步进 ES。**
