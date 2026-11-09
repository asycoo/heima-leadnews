# 项目部署与持续集成（Jenkins + Git + Docker）

> 讲义：day12《项目部署_持续集成》  
> 本文按 **Windows** 环境写（Docker Desktop、本机装 Jenkins）。

**角色约定：**

| 角色 | 是谁 | 干什么 |
| --- | --- | --- |
| **构建机（100）** | Windows | **Jenkins** + **Docker 私有仓库 Registry（:5000）**；拉代码、打包、打镜像、**push 进私有仓** |
| **运行机（130）** | Windows | **不跑 Jenkins**；只从 100 的私有仓 **pull**，再 `docker run` 全部微服务 |

> **私有库在哪？在 100，不在 130。**  
> 地址示例：`http://192.168.200.100:5000`。130 只是客户端来拉镜像。

IP 可按你局域网改；下文仍用 `100` / `130` 指构建机 / 运行机。

---

## 0. 先建立画面

```text
开发提交 Git
    ↓
【构建机 Windows】Jenkins
    ├─ Git 拉代码
    ├─ Maven 编译 / install / 打 jar
    ├─ dockerfile-maven-plugin 打镜像
    ├─ tag + push → 本机 Registry :5000
    └─ Publish Over SSH → 运行机
    ↓
【运行机 Windows】Docker Desktop
    ├─ docker pull
    ├─ 删旧容器
    └─ docker run（prod）
```

| 东西 | 角色 |
| --- | --- |
| Git | 代码仓库 |
| Jenkins | CI：构建、推镜像、远程启动 |
| JDK 8 / Maven | 编译打包 |
| Docker Desktop | 构建机打镜像；运行机跑容器 |
| Registry | **Docker 私有仓，装在 100 的 :5000**（130 只 pull） |
| Publish Over SSH | Jenkins → 运行机执行 `docker pull/run` |
| Nacos Profile | 容器 `-e PARAMS=--spring.profiles.active=prod` |

名词：`$JOB_NAME` / `%JOB_NAME%` ≈ 镜像名后缀 ≈ 运行机容器名；`insecure-registry` = 允许 HTTP 私有仓。

---

## 1. 构建机：装 Jenkins

1. 先装 **JDK 8**，配置系统环境变量 `JAVA_HOME`（如 `C:\Program Files\Java\jdk1.8.0_xxx`），`PATH` 加 `%JAVA_HOME%\bin`  
2. 官网下 Jenkins Windows 安装包安装；或下载 `jenkins.war` 后：

```bat
java -jar jenkins.war --httpPort=16060
```

3. 讲义习惯端口 **16060**（安装向导/启动参数里改）

访问：`http://localhost:16060/` 或 `http://构建机IP:16060/`  
首次密码在 Jenkins home 下 `secrets\initialAdminPassword`（常见在用户目录 `.jenkins` 或安装目录）。

装推荐插件 → 建管理员。

### 必装插件

| 插件 | 干什么 |
| --- | --- |
| Maven Integration | Maven 构建 |
| Docker plugin | Docker 集成 |
| Git plugin | 拉代码 |
| Publish Over SSH | **远程到运行机执行脚本（关键）** |
| SSH | 远程命令 |

更新中心异常可改：`http://updates.jenkins.io/update-center.json`

### 全局工具配置

系统管理 → 全局工具配置：

| 工具 | Windows 示例路径 |
| --- | --- |
| JDK | `C:\Program Files\Java\jdk1.8.0_xxx` |
| Maven | `C:\apache-maven-3.6.1` |
| Git | `C:\Program Files\Git\bin\git.exe` |
| Docker | Docker Desktop 安装后 `docker` 一般已在 PATH |

Maven 的全局 `settings.xml` 要指到**已上传到构建机的本地仓库**（见下一节）。

---

## 2. 构建机：Git / Maven / Docker Desktop

### Git

安装 [Git for Windows](https://git-scm.com/)，安装后：

```bat
git --version
```

### Maven（含：把开发机依赖仓库拷到构建机）

讲义做法：**不是让 100 慢慢从公网下依赖**，而是把你开发机上已经下好的本地仓库打个包，拷到构建机（100），再让 Maven 用这份仓库。

```text
开发机（你日常写代码的 Windows）
  └─ 本地仓库，一般在 C:\Users\你\.m2\repository
        ↓ 打包（zip）上传 / 拷贝
构建机 100
  └─ 解压到例如 D:\maven-repo\repository
        ↓
  settings.xml 的 <localRepository> 指到这个目录
```

步骤：

1. 构建机解压 Maven，如 `C:\apache-maven-3.6.1`  
2. 环境变量：

| 变量 | 值 |
| --- | --- |
| `MAVEN_HOME` | `C:\apache-maven-3.6.1` |
| `PATH` | 追加 `%MAVEN_HOME%\bin` |

3. `mvn -v` 验证  
4. **把开发机 `.m2\repository` 整份拷到 100**（讲义：不上传会自动下载，又慢又容易失败）  
5. 编辑构建机 `%MAVEN_HOME%\conf\settings.xml`，指定这份仓库，例如：

```xml
<localRepository>D:/maven-repo/repository</localRepository>
```

6. Jenkins「全局工具配置」里的 Maven / 全局 settings，也指向这份 `settings.xml`

之后 Jenkins 在 100 上 `mvn install` 时，优先用这份已拷过去的依赖；父工程任务仍会把黑马头条自己的模块 install 进同一仓库。

### Docker Desktop

构建机、运行机都装 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)（按向导开 WSL2）。

构建机用途：**build 镜像 + push Registry**，不长期跑业务容器。

---

## 3. Docker 私有仓库：就在 100

### 3.1 为什么要私有仓？装在哪？

```text
100 打出来的镜像  ──push──►  100:5000 私有仓（Registry）
                                    │
130 业务机  ◄──pull───────────────┘
```

- **私有库 = Docker Registry，部署在 100（和 Jenkins 同一台）**  
- 端口 **5000**；访问目录：`http://192.168.200.100:5000/v2/_catalog`  
- 130 **不装** Registry，只当 Docker 客户端来 pull  
- 不用私有仓也行（save/load + scp），但多服务时又烦又不稳，讲义用私有仓统一管理（类似「镜像版的 Git」）

### 3.2 在 100 上启动 Registry

```bat
docker pull registry:latest
docker run -d -p 5000:5000 --name registry -v registry-data:/var/lib/registry registry:latest
```

| 参数 | 含义 |
| --- | --- |
| `-p 5000:5000` | 宿主机 5000 → 仓服务 |
| `-v registry-data:...` | 镜像数据持久化（卷名可改成盘符路径） |
| `--name registry` | 容器名 |

浏览器：`http://192.168.200.100:5000/v2/_catalog`  
空仓：`{"repositories":[]}`；push 成功后这里能列出仓库名。

### 3.3 100 和 130 都要配 insecure-registries

内网 Registry 一般是 **HTTP**。Docker 默认只信 HTTPS，所以两边 Docker Desktop 都要放行：

Settings → Docker Engine：

```json
{
  "insecure-registries": ["192.168.200.100:5000"]
}
```

Apply & Restart。

| 机器 | 为啥配 |
| --- | --- |
| **100** | 本地 `docker push 192.168.200.100:5000/...` |
| **130** | `docker pull 192.168.200.100:5000/...` |

### 3.4 运行机 130

- 装 Docker Desktop  
- 能 ping / 访问 `100:5000`  
- 已配 `insecure-registries`  
- **不要**在 130 再起一个 Registry（除非你故意做高可用，课程不需要）

---

## 4. 多环境（代码 + Nacos）

开发 / 测试 / 生产连的库不同 → Spring Profile。

`bootstrap.yml`：

```yaml
server:
  port: 51801
spring:
  application:
    name: leadnews-user
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.200.130:8848
      config:
        server-addr: 192.168.200.130:8848
        file-extension: yml
  profiles:
    active: dev
```

Nacos DataId 命名：

```text
${spring.application.name}-${profile}.${file-extension}
例：leadnews-user-dev.yml
    leadnews-user-prod.yml
```

| 注意 | 说明 |
| --- | --- |
| prefix | 默认=`spring.application.name` |
| profile 为空 | DataId 变成 `服务名.yml`（中间没有 `-`） |
| file-extension | `yml` / `properties` |

运行机容器必须带：

```text
-e PARAMS="--spring.profiles.active=prod"
```

才会去拉 `*-prod.yml`。每个要部署的微服务在 Nacos 里都要有对应 prod 配置。

---

## 5. 微服务如何打成 Docker 镜像（代码侧，写细）

目标：每个微服务先变成镜像，再在 **130** 起容器。讲义推荐用 **dockerfile-maven-plugin**（比手动传 jar + 手写 docker build 省事）。

### 5.1 总流程（和 Jenkins 的对应关系）

```text
① 每个微服务 pom 加插件 + 根目录放 Dockerfile     ← 改代码，提交 Git
② Jenkins 父工程任务：install 公共模块进 100 的 Maven 仓
③ Jenkins 微服务任务：
     clean install + dockerfile:build   → 100 本地出现镜像 docker_storage/服务名
     tag + push                         → 推进 100:5000 私有仓
④ SSH 到 130：pull → 删旧容器 → run（prod）
```

### 5.2 每个微服务 pom 要配什么（以 user 为例）

在 `heima-leadnews-user/pom.xml`（其它服务同样改一份）：

```xml
<properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <!-- 镜像名前缀；最终镜像 = docker_storage/heima-leadnews-user -->
    <docker.image>docker_storage</docker.image>
</properties>

<build>
    <!-- 打出来的 jar 名，要和 Dockerfile / JOB_NAME 对得上 -->
    <finalName>heima-leadnews-user</finalName>
    <plugins>
        <!-- 打可执行 fat jar -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals><goal>repackage</goal></goals>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.7.0</version>
            <configuration>
                <source>${java.version}</source>
                <target>${java.version}</target>
            </configuration>
        </plugin>
        <!-- 关键：根据同目录 Dockerfile 构建镜像 -->
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>dockerfile-maven-plugin</artifactId>
            <version>1.3.6</version>
            <configuration>
                <repository>${docker.image}/${project.artifactId}</repository>
                <buildArgs>
                    <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                </buildArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

| 配置项 | 含义 |
| --- | --- |
| `docker.image=docker_storage` | 镜像仓库命名空间前缀 |
| `repository=.../${project.artifactId}` | 完整镜像名，如 `docker_storage/heima-leadnews-user` |
| `finalName` | `target/heima-leadnews-user.jar` |
| `JAR_FILE` buildArg | 传给 Dockerfile 的 `ARG JAR_FILE` |
| `dockerfile:build` | Jenkins Maven 步骤里要写上这个 goal |

**Jenkins 任务名**建议就叫 `heima-leadnews-user`，因为后面脚本用 `%JOB_NAME%` / `$JOB_NAME` 当镜像名后缀和容器名。

### 5.3 微服务根目录 Dockerfile（与 pom 同级）

```dockerfile
# 基础 JDK 镜像
FROM java:8
# 容器内 /tmp 用卷，不写进镜像层
VOLUME /tmp
# 接收插件传入的 jar 路径
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
# 限制内存，少占运行机资源
ENV JAVA_OPTS="\
-server \
-Xms256m \
-Xmx512m \
-XX:MetaspaceSize=256m \
-XX:MaxMetaspaceSize=512m"
# 占位：docker run -e PARAMS=... 传入
ENV PARAMS=""
ENTRYPOINT ["sh","-c","java -jar $JAVA_OPTS /app.jar $PARAMS"]
```

| 指令 | 干什么 |
| --- | --- |
| `FROM java:8` | 运行环境；拉不到可换 OpenJDK 8 镜像 |
| `VOLUME /tmp` | 临时文件 |
| `ARG` + `COPY` | 把 fat jar 拷成 `/app.jar` |
| `JAVA_OPTS` | JVM 参数 |
| `PARAMS` | 空默认；运行时塞 `--spring.profiles.active=prod` |
| `ENTRYPOINT` | 真正启动命令 |

本地可先试（在有 Docker 的机器）：

```bat
mvn clean install -Dmaven.test.skip=true dockerfile:build -f 路径\heima-leadnews-user\pom.xml
docker images
```

应能看到 `docker_storage/heima-leadnews-user`。

---

## 6. Jenkins 任务怎么配（写细：100 构建 → 私有仓 → 130 运行）

前置：100 上 Git/Maven/Docker/Registry 已好；开发机 `.m2\repository` 已拷到 100；微服务 pom + Dockerfile 已进 Git。

### 6.1 任务一：父工程 / 基础依赖 install（必须先跑）

微服务编译依赖 `heima-leadnews-model`、`common` 等，要先装进 **100 的 Maven 本地仓**。

新建任务（Freestyle 或 Maven 项目均可），名称随意，如 `heima-leadnews-base`。

| 配置项 | 怎么填 |
| --- | --- |
| 源码管理 → Git | 仓库 URL |
| Credentials | Git 用户名密码（Add → Username with password） |
| 分支 | 如 `*/master` 或你的分支 |
| Build | Invoke top-level Maven targets |
| Maven Version | 选全局工具里配好的 Maven |
| Goals | 对父工程 pom：`clean install -Dmaven.test.skip=true`（路径用 Root POM / `-f` 按你仓库结构） |

构建日志应看到：拉代码 → 编译 → BUILD SUCCESS。  
失败常见原因：100 上 `localRepository` 没指到拷过去的依赖、或父 pom 路径不对。

> 以后公共模块有改动，先重新跑这个任务，再跑各微服务。

### 6.2 配好 100 → 130 的 SSH（所有远程部署任务共用）

1. **130**：Windows「可选功能」安装 **OpenSSH 服务器**，服务设为自动启动；防火墙放行 22  
2. Jenkins 插件：已装 **Publish Over SSH**  
3. **Manage Credentials**：添加 130 登录账号（用户名/密码或密钥）  
4. **Manage Jenkins → Configure System → Publish over SSH → SSH Servers**：  
   - Name：如 `leadnews-130`  
   - Hostname：`192.168.200.130`  
   - Username：与凭证一致  
   - Remote Directory：可填 `/` 或某工作目录  
   - 勾选用凭证 / 填密码，**Test Configuration** 要成功  

### 6.3 任务二：单个微服务（以 user 为例；article / 网关等同理）

#### （1）新建任务

- 类型：与讲义类似的 Maven/自由风格任务  
- **任务名务必**与镜像 artifact 一致：`heima-leadnews-user`（后面全靠 `%JOB_NAME%`）

#### （2）参数化构建

勾选「This project is parameterized」→ 增加 String Parameter：

| 参数名 | 默认值 | 含义 |
| --- | --- | --- |
| `docker_registry` | `192.168.200.100:5000` | **就是 100 上的私有仓地址** |

#### （3）源码管理

同任务一：同一 Git 仓库 + 凭证 + 分支。

#### （4）Build：Maven（在 100 执行）

Goals（`-f` 路径按你 Git 检出后的真实相对路径改）：

```text
clean install -Dmaven.test.skip=true dockerfile:build -f heima-leadnews/heima-leadnews-service/heima-leadnews-user/pom.xml
```

| 片段 | 含义 |
| --- | --- |
| `clean install` | 清理并打包、装到 100 本地 Maven 仓 |
| `-Dmaven.test.skip=true` | 跳过测试 |
| `dockerfile:build` | 调用插件，在 **100 Docker** 打出镜像 `docker_storage/heima-leadnews-user` |
| `-f .../pom.xml` | 必须指向该微服务的 pom |

构建成功后，在 **100** 上 `docker images` 应能看到 `docker_storage/heima-leadnews-user`（此时还只在 100 本地，130 还没有）。

#### （5）Build：Execute Windows batch command（仍在 100：打标签并推入私有仓）

```bat
set image_tag=%docker_registry%/docker_storage/%JOB_NAME%
echo ===== 清理悬空镜像 =====
docker image prune -f

echo ===== 打上私有仓标签（指向 100:5000）=====
docker tag docker_storage/%JOB_NAME% %image_tag%

echo ===== 推送到 100 私有仓 =====
docker push %image_tag%

echo ===== 删除带仓库前缀的本地 tag（省空间）=====
docker rmi %image_tag%
```

| 步骤 | 含义 |
| --- | --- |
| `image_tag` | 例：`192.168.200.100:5000/docker_storage/heima-leadnews-user` |
| `docker tag` | 给本地镜像多一个「带私有仓地址」的名字 |
| `docker push` | **推到 100:5000**；浏览器 `_catalog` 应能看见 |
| `docker rmi` 掉 tag | 只删带前缀的标签名，不删私有仓里的镜像 |

推送失败优先查：100 的 `insecure-registries`、Registry 容器是否在跑、参数 `docker_registry` 是否写成了 130。

#### （6）构建后步骤：Send files or execute commands over SSH（到 130）

选刚才配的 SSH Server（130）：

**Exec command：**

```bash
echo '===== 从 100 私有仓拉镜像 ====='
docker pull $docker_registry/docker_storage/$JOB_NAME

echo '===== 删旧容器 ====='
if [ -n "$(docker ps -a -f name=$JOB_NAME --format '{{.ID}}')" ]; then
  docker rm -f $(docker ps -a -f name=$JOB_NAME --format '{{.ID}}')
fi
docker image prune -f

echo '===== 在 130 启动 ====='
docker run -d --net=host \
  -e PARAMS="--spring.profiles.active=prod" \
  --name $JOB_NAME \
  $docker_registry/docker_storage/$JOB_NAME
```

| 点 | 含义 |
| --- | --- |
| `docker pull ...100:5000/...` | 镜像来自 **100 私有仓**，不是 130 本地 build |
| 先 `rm` 同名容器 | 避免名字冲突，实现覆盖发布 |
| `--net=host` | 讲义写法；Docker Desktop 若端口异常，改成 `-p 51801:51801`（按服务端口） |
| `PARAMS=prod` | 激活 Nacos prod 配置 |
| `--name $JOB_NAME` | 容器名与任务名一致，方便下次删 |

> Publish Over SSH 传参：确保远程环境能读到 `docker_registry`、`JOB_NAME`（有的环境要在 SSH 脚本里写死 `192.168.200.100:5000` 和镜像名，或勾选传递构建变量，按你插件版本试一下）。不生效就直接把脚本里的变量换成字面量。

#### （7）构建完成后在 130 检查

```bat
docker images
docker ps
```

应看到镜像 `192.168.200.100:5000/docker_storage/heima-leadnews-user` 和同名运行中容器。  
**100 上不应作为业务服务长期 run**（最多留构建缓存镜像）。

### 6.4 其它微服务 / 网关

每个服务复制「任务二」：改任务名、改 `-f` 的 pom 路径即可。  
联调时把 **app 网关** 也按同样方式部署到 130。

### 6.5 Nginx 联调

本机/前端 Nginx 的 upstream 指 **130 网关**，不要指 100：

```nginx
upstream heima-app-gateway {
    server 192.168.200.130:51601;
}
```

---

## 7. 触发器

| 方式 | 怎么配 | 说明 |
| --- | --- | --- |
| URL 远程触发 | 勾选触发远程构建，设 token；`http://100:16060/job/任务名/build?token=xxx` | 给钩子/脚本 |
| 其他工程构建后 | 选上游 Job（如父工程） | 适合 base → 各微服务 |
| 定时 Build periodically | Cron：分 时 日 月 周 | 例 `H 12 * * *` 每天中午 |
| 轮询 SCM | 定时扫仓库 | 讲义：**开销大，不建议** |

```text
H/30 * * * *      约每 30 分钟
0 8,12,22 * * *   每天 8/12/22 点
H 12 * * *        每天中午
```

---

## 8. 配置对照总表

| 位置 | 配置 |
| --- | --- |
| **100** | Jenkins + **Docker 私有仓 Registry:5000** + 构建用 Docker |
| **130** | 只跑业务容器；**无私有仓**；从 100:5000 pull |
| Maven 依赖仓 | 开发机 `.m2\repository` **拷到 100**，`settings.xml`→`localRepository` |
| 微服务代码 | pom 加 `dockerfile-maven-plugin`；根目录 Dockerfile |
| 任务名 | = artifact = 容器名，如 `heima-leadnews-user` |
| 参数 `docker_registry` | `192.168.200.100:5000`（私有仓在 100） |
| insecure-registries | **100 和 130** 都写 `192.168.200.100:5000` |
| 100 bat | `tag` + `push` 进私有仓 |
| 130 SSH | `pull` + `rm` + `run`（prod） |
| Nginx | → **130** 网关端口 |

---

## 9. 快速自问

1. 私有库在哪？→ **在 100 的 `:5000`**，不在 130。  
2. 两台怎么分？→ 100=Jenkins+私有仓+构建；130=只跑容器。  
3. Maven 本地仓库拷到哪？→ 拷到 **100**，给构建用；和 Docker 私有仓是两回事。  
4. `docker_storage/xxx` 和 `100:5000/docker_storage/xxx`？→ 前者是 100 本地构建名；tag/push 后才进私有仓。  
5. push/pull 失败？→ Registry 是否在 100 运行；两边 insecure 是否指向 **100:5000**。  
6. Nginx 指谁？→ **130**。  

---

## 一句话

**私有镜像库装在 100（:5000）和 Jenkins 一起；开发机 Maven 依赖仓也拷到 100；Jenkins 在 100 打镜像并 push 进私有仓，再 SSH 让 130 pull 下来以 prod 启动全部服务。**
