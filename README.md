# 黑马头条 · 精简学习包

个人学习用精简副本，结构如下：

```text
notes/          自己整理的流程笔记（登录鉴权、Nginx、MinIO、Feign、审核、Kafka、ES、热点、Jenkins…）
courses/        按天：讲义 + sql/配置类资料（不含 PPT、不含每日完整代码）
  day13-.../
    code/       仅 day13 的 heima-leadnews 工程源码
    01-说明 / 02-需求 / 03-资料(nginx、mongo sql)
resources/
  sql/          各库脚本汇总
  nginx/        nginx 配置汇总
  xxl-job/      tables_xxl_job.sql
```

说明：

- 代码只保留 **day13** 工程；`model`/`common` 等已包含在该工程内。
- 其它天只保留 **讲义** 与 **sql/配置**，去掉 PPT、完整中间代码、maven 仓库 zip、xxl-job 整包源码等大文件。
