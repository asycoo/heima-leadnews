# Nginx 前端部署与反向代理流程

> 讲义：day01（APP）、day03（自媒体）、day08（管理端）、day13（自媒体 `$args`）  
> 作用：一个 Nginx 同时托管多个前端静态页，并把接口请求反代到对应网关

---

## Nginx 在项目里干什么

```text
浏览器访问 :880x
    ├─ /              → 返回前端静态资源（html/js/css）
    └─ /约定前缀/**   → 剥掉前缀，转发到对应 Gateway
                           ↓
                      Gateway 再路由到各微服务
```

| 端 | 访问端口 | 静态资源目录（示例） | 接口前缀 | 反代到网关 |
| --- | --- | --- | --- | --- |
| APP | 8801 | `app-web/` | `/app/` | `localhost:51601` |
| 自媒体 | 8802 | `wemedia-web/` | `/wemedia/MEDIA/` | `localhost:51602` |
| 管理端 | 8803 | `admin-web/` | `/service_6001/` | `localhost:51603` |

> 需求文档里管理端 upstream 常写 `6001`，代码里 admin-gateway 实际端口是 **51603**，联调以实际端口为准。

---

## 总流程

```text
1. nginx.conf 用 include 加载 leadnews.conf/*.conf（多端各一个 conf）
2. 每个 conf：upstream 指向网关 + server 监听端口
3. location /        → root 指到前端解压目录，当静态服务器
4. location ~ 前缀   → 正则捕获路径，proxy_pass 到网关（剥前缀）
5. 启动/重载 nginx，浏览器打开对应端口联调
```

---

## 1. 怎么挂进 Nginx

`conf/nginx.conf` 的 `http` 块里引入目录，不要把三端全写死在一个文件里：

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    include leadnews.conf/*.conf;   # APP / 自媒体 / 管理端各一份
}
```

目录结构大致：

```text
nginx/conf/
├── nginx.conf
└── leadnews.conf/
    ├── heima-leadnews-app.conf
    ├── heima-leadnews-wemedia.conf
    └── heima-leadnews-admin.conf
```

常用命令：`nginx` 启动，`nginx -s reload` 改配置后重载。

---

## 2. 单端配置怎么读（以 APP 为例）

```nginx
upstream heima-app-gateway {
    server localhost:51601;   # 真正的 app-gateway
}

server {
    listen 8801;              # 浏览器访问这个端口

    # ① 静态页
    location / {
        root D:/workspace/app-web/;
        index index.html;
    }

    # ② 接口反代：/app/xxx → 网关 /xxx
    location ~/app/(.*) {
        proxy_pass http://heima-app-gateway/$1;
        proxy_set_header HOST $host;
        proxy_pass_request_body on;
        proxy_pass_request_headers on;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 请求怎么被改写

```text
浏览器：http://localhost:8801/app/user/api/v1/login/login_auth
                │
                │  location ~/app/(.*)  捕获组 $1 = user/api/v1/login/login_auth
                ▼
proxy_pass → http://localhost:51601/user/api/v1/login/login_auth
                │
                ▼
app-gateway 再按 /user/** 路由到 leadnews-user
```

要点：

- `~/app/(.*)`：正则，`(.*)` 抓前缀后面的路径
- `proxy_pass .../$1`：用捕获内容拼目标 URL，等于**剥掉 `/app`**
- `proxy_set_header`：把 Host、真实 IP 传下去，方便网关/日志识别客户端

---

## 3. 三端配置对照

### APP（day01）

```nginx
listen 8801;
location / { root .../app-web/; }
location ~/app/(.*) {
    proxy_pass http://heima-app-gateway/$1;   # → 51601
}
```

登录联调示例：`http://localhost:8801/app/user/api/v1/login/login_auth`

### 自媒体（day03 / day13）

```nginx
listen 8802;
location / { root .../wemedia-web/; }
location ~/wemedia/MEDIA/(.*) {
    proxy_pass http://heima-wemedia-gateway/$1;   # → 51602
}
```

day13 新前端有 GET 多参数，要带上查询串：

```nginx
proxy_pass http://heima-wemedia-gateway/$1?$args;
```

不加 `?$args` 时，路径能到网关，**URL 上的查询参数可能丢**。

### 管理端（day08）

```nginx
listen 8803;
location / { root .../admin-web/; }
location ~/service_6001/(.*) {
    proxy_pass http://heima-admin-gateway/$1;   # → 51603（不要照抄文档里的 6001）
}
```

示例：`http://localhost:8803/service_6001/login/in` → 网关 `/login/in` → `leadnews-admin`

---

## 4. 一次完整访问串起来

```text
打开 http://localhost:8801
  → location / 返回 app-web/index.html

前端调登录，请求发到 /app/user/api/v1/login/login_auth
  → Nginx 剥掉 /app，转到 51601 网关
  → 网关 /login 放行，进 user 服务登录，返回 token

之后业务请求同样走 /app/...，Header 带 token
  → Nginx 只负责转发，鉴权仍在网关 AuthorizeFilter
```

Nginx **不做登录校验**，只做：静态资源 + 反代到网关。

---

## 5. 部署到远程时改什么（day12）

本地联调：`upstream` 写 `localhost:51601`。  
服务部署到 `192.168.200.100` 后，只改 upstream：

```nginx
upstream heima-app-gateway {
    server 192.168.200.100:51601;
}
```

前端仍访问本机 `8801`，接口被代理到远端网关。

---

## 联调检查

1. `root` 路径是否指向解压后的前端目录（错了会 404 白屏）
2. `listen` 端口是否和浏览器地址一致（8801/8802/8803）
3. `upstream` 端口是否对准真实网关（51601/51602/51603）
4. 接口前缀是否和前端约定一致（`/app`、`/wemedia/MEDIA`、`/service_6001`）
5. 自媒体 GET 带参：`proxy_pass` 是否加了 `?$args`
6. 改完配置执行 `nginx -s reload`
