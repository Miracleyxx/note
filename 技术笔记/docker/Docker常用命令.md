
---

# 🐳 Docker常用命令

> 💡 **核心用途**：容器生命周期管理、镜像构建优化、网络与存储编排。适用于开发调试、DevOps 流水线配置及生产环境故障排查。

## ⚡ 极速实战 (Quick Start)

### 1. 启动服务的标准姿势

后台运行、端口映射、自动重启、具名容器。

``` shell
docker run -d --name my-web-app -p 8080:80 --restart always  -v ./data:/data  nginx:latest
```

### 2. 紧急进入容器 (Debug)

用于进入容器内部查看文件或调试进程。

```shell
# -it 开启交互式终端，/bin/bash 或 /bin/sh 取决于镜像
docker exec -it <container_id_or_name> /bin/bash
```

### 3. "核弹级" 清理 (释放磁盘空间)

⚠️ **慎用**：删除所有停止的容器、未被使用的网络、悬空的镜像和构建缓存。

```shell
docker system prune -a --volumes -f
```

---

## 📖 核心命令字典 (Command Dictionary)

### 1. 容器生命周期 (Container Ops)

|**命令 (Command)**|**关键参数**|**说明 (Description)**|
|---|---|---|
|**`docker run`**|`-d` (后台)<br><br>  <br><br>`--rm` (退出即删)<br><br>  <br><br>`-e` (环境变量)|**核心命令**。从镜像启动新容器。`--rm` 推荐用于临时任务。|
|`docker ps`|`-a` (显示所有)<br><br>  <br><br>`-q` (仅ID)<br><br>  <br><br>`--format`|查看容器列表。配合 `table {{.ID}}\t{{.Names}}` 可自定义输出。|
|`docker stop/start`|`-t 10` (超时秒数)|优雅停止 (SIGTERM) 或启动。若超时则强制杀掉 (SIGKILL)。|
|`docker rm`|`-f` (强制)|删除容器。若容器正在运行需加 `-f`。|
|`docker stats`|`--no-stream`|**监控**。实时查看容器的 CPU、内存、网络 I/O 占用。|

### 2. 镜像管理 (Image Ops)

|**命令 (Command)**|**关键参数**|**说明 (Description)**|
|---|---|---|
|**`docker build`**|`-t` (打标签)<br><br>  <br><br>`-f` (指定文件)<br><br>  <br><br>`--target`|从 Dockerfile 构建镜像。`--target` 用于多阶段构建调试。|
|`docker pull`|`--platform`|拉取镜像。如需拉取 ARM 架构镜像需指定平台。|
|`docker push`|N/A|将本地镜像推送到仓库 (需先 `docker login`)。|
|`docker tag`|N/A|**改名/归档**。`docker tag source:tag target:tag`。|
|`docker rmi`|`-f` (强制)|删除镜像。若有容器依赖该镜像，需先删容器或用 `-f`。|
|`docker image prune`|`-a` (所有未使用)|清理悬空镜像 (Dangling images, 标签为 `<none>` 的)。|

### 3. 调试与交互 (Debugging)

|**命令 (Command)**|**常用参数**|**场景与用途**|
|---|---|---|
|**`docker logs`**|`-f` (实时)<br><br>  <br><br>`--tail 100`|**查错神器**。查看容器标准输出(stdout)和错误(stderr)。|
|`docker inspect`|`-f` (Go模板)|**X光透视**。查看容器/镜像的元数据（IP、挂载点、Cmd）。|
|`docker cp`|N/A|**文件传输**。`docker cp local_file container:/path` (双向可用)。|
|`docker diff`|N/A|**文件变更审计**。查看容器相对于镜像改动了哪些文件 (A/D/C)。|

---

## 💾 存储与数据持久化 (Storage & Volumes)

> ℹ️ **架构师视角**：生产环境**必须**使用 Volume 或 Bind Mount，否则容器重启数据即丢。

### 1. 挂载方式对比

|**类型**|**语法示例 (-v)**|**特点**|**适用场景**|
|---|---|---|---|
|**Bind Mount**|`-v /host/path:/container/path`|宿主机路径映射。性能最好，依赖宿主机结构。|**开发环境** (代码热更新)、挂载配置文件。|
|**Volume**|`-v vol_name:/container/path`|Docker 托管 (`/var/lib/docker/volumes`)。跨平台，易备份。|**生产环境** (数据库、文件存储)。|
|**Tmpfs**|`--tmpfs /path`|仅存储在内存中，不落盘。|存储密钥、高速缓存、临时文件。|

### 2. Volume 管理命令

```shell
# 创建一个专用卷
docker volume create my-db-data

# 查看卷的物理位置 (Mountpoint)
docker volume inspect my-db-data

# 清理所有未被挂载的卷 (释放磁盘)
docker volume prune
```

---

## 🌐 网络架构 (Networking)

> ℹ️ **架构师视角**：不要使用默认的 `bridge` 网络，因为它无法使用 DNS 服务名通信。**总是创建自定义网络**。

### 1. 网络模式图解

|**驱动 (Driver)**|**命令行参数**|**拓扑说明**|**场景**|
|---|---|---|---|
|**Bridge**|`--net bridge`|**默认/自定义网桥**。容器有独立 IP，通过 NAT 访问外网。|单机多容器通信 (推荐)。|
|**Host**|`--net host`|**去隔离**。容器共用宿主机 IP 和端口。性能最高。|高吞吐场景、端口占用极多时。|
|**None**|`--net none`|**无网络**。只有 loopback 接口。|极高安全要求的离线任务。|
|**Macvlan**|`--net macvlan`|**物理直通**。容器拥有物理局域网的独立 MAC 地址。|传统应用迁移、需独立 IP。|

### 2. 网络实战命令

```shell
# 1. 创建自定义网桥 (支持容器名 DNS 解析)
docker network create my-app-net

# 2. 启动容器并加入网络
docker run -d --name redis --network my-app-net redis
docker run -d --name web --network my-app-net -p 80:80 my-web-app

# 3. 验证通信 (在 web 容器内 ping redis)
# 注意：直接 ping 容器名 "redis"，而不是 IP
docker exec -it web ping redis
```

---

## 🐙 Docker Compose 编排 (Orchestration)

> 💡 **核心用途**：单机多容器编排。将繁琐的 `docker run` 命令转化为可版本控制的 YAML 文件。

### 1. 常用命令速查

|**命令**|**说明**|**备注**|
|---|---|---|
|**`docker compose up -d`**|启动并后台运行|生产环境最常用。会自动构建镜像、创建网络。|
|`docker compose down`|停止并**移除**容器/网络|加上 `-v` 可同时删除挂载的 Volume (慎用)。|
|`docker compose logs -f`|查看所有服务日志|可指定服务名，如 `logs -f nginx`。|
|`docker compose restart`|重启服务|仅重启，不重新加载配置变更。|
|`docker compose ps`|查看服务状态|类似于 `docker ps`，但仅限当前项目。|

### 2. 典型 `compose.yaml` 模板


```yaml
version: '3.8'

services:
  # 1. 前端服务
  web:
    build: .
    ports:
      - "8080:80"
    networks:
      - app-net
    depends_on:
      - db
    environment:
      - DB_HOST=db

  # 2. 数据库服务
  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data # 使用命名卷
    networks:
      - app-net
    environment:
      POSTGRES_PASSWORD: secret

# 3. 定义网络
networks:
  app-net:
    driver: bridge

# 4. 定义存储卷
volumes:
  db-data:
```

---

## 🧬 底层原理与 Dockerfile 技巧 (Mechanisms)

### 1. Dockerfile 指令避坑指南

|**指令**|**最佳实践 (Best Practice)**|**原因 (Why?)**|
|---|---|---|
|**`COPY` vs `ADD`**|**优先用 `COPY`**。仅在需要解压 tar 包时用 `ADD`。|`ADD` 包含下载 URL 等隐晦功能，`COPY` 语义更明确。|
|**`CMD` vs `ENTRYPOINT`**|用 `ENTRYPOINT` 定义可执行文件，用 `CMD` 定义默认参数。|让容器像二进制程序一样可接收参数。|
|**`RUN`**|合并多条 `RUN` 指令，并清理缓存 (`rm -rf /var/lib/apt/lists/*`)。|减少镜像层数，减小体积。|
|**`WORKDIR`**|始终使用绝对路径。避免使用 `RUN cd ...`。|保证路径上下文清晰，防止 "找不到文件" 错误。|

### 2. 镜像分层 (Layer Caching) 原理

Docker 构建时会逐层缓存。

- **优化策略**：将 **变化最少** 的指令（如安装依赖）放在前面，**变化最频繁** 的指令（如 COPY 源代码）放在后面。
    
- **示例**：
    
    ```Dockerfile
    # ✅ 正确做法：先复制依赖描述文件，安装依赖，再复制源代码
    COPY package.json .
    RUN npm install      # 这一层会被缓存，除非 package.json 变了
    COPY . .             # 源代码变了只会重跑这一层及之后
    CMD ["npm", "start"]
    ```
    

---

## 🔍 高级排查技巧 (Expert Troubleshooting)

当 `docker logs` 看不出问题时，请使用以下“手术刀”：

1. **检查容器为何退出 (Exit Code)**:
    
    ```shell
    docker inspect <container> --format='{{.State.ExitCode}}'
    ```
    
    - `137`: OOM (内存溢出，被系统杀掉)。
        
    - `1`: 应用错误 (检查配置或代码)。
        
2. **检查挂载是否生效**:
    
    ```shell
    docker inspect <container> --format='{{json .Mounts}}'
    ```
    
3. 抓包 (网络调试):
    
    容器也是进程，其网卡在宿主机可见 (需查找对应的 veth)。
    
    或者使用共享网络模式调试：
    
    ```shell
    # 启动一个带网络工具的容器，依附到故障容器的网络命名空间
    docker run -it --rm --net container:<故障容器名> nicolaka/netshoot
    ```