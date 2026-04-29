# Debian 服务器部署说明

这份文档是给 `rwsmd` 以后重复部署用的，重点面向 Debian 服务器，尤其是 1 核 1G 这类小机器。

## 先看结论

推荐方案分两类：

| 服务器规格 | 推荐方案 | 说明 |
|---|---|---|
| `1核1G`、`1核2G` | **方案 A：别在服务器源码构建，直接运行已构建镜像** | 最稳，最省内存 |
| `2核4G` 及以上 | **方案 B：服务器直接源码构建** | 可以用 `docker-compose.dev.yml` |

这次踩坑的核心原因有两个：

1. 旧 Docker 环境下，`buildx` 和 daemon API 版本不匹配。
2. `1核1G` 机器在前端 `vite build` 阶段很容易 OOM。

所以以后默认按 **方案 A** 走。

---

## 目录说明

和本说明配套的文件：

- `deploy/docker-compose.server.yml`
- `deploy/docker-compose.dev.yml`
- `deploy/.env.example`

说明：

- `docker-compose.server.yml`
  只运行镜像，不在服务器构建源码。
- `docker-compose.dev.yml`
  从源码构建镜像，适合内存更大的机器。

---

## 方案 A：小内存 Debian 服务器推荐方案

### 适用场景

- Debian 11 / 12
- 服务器只有 `1核1G` 或 `1核2G`
- 服务器只负责运行服务，不负责源码编译

### 步骤 1：在别的机器上构建镜像

建议在你本地电脑，或者一台内存更大的 Linux / Windows Docker 主机上执行。

```bash
git clone https://github.com/zrwsmd/rw-sub-proxy.git
cd rw-sub-proxy

docker build --platform linux/amd64 -t rw-sub-proxy:latest .
docker save rw-sub-proxy:latest | gzip > rw-sub-proxy_latest_linux_amd64.tar.gz
```

如果你的服务器不是 `amd64`，把 `linux/amd64` 改成对应平台。

### 步骤 2：把镜像和部署文件传到服务器

```bash
scp rw-sub-proxy_latest_linux_amd64.tar.gz root@your-server:/opt/
```

服务器上再拉代码仓库，只拿部署文件也行：

```bash
cd /opt
git clone https://github.com/zrwsmd/rw-sub-proxy.git
```

### 步骤 3：在服务器加载镜像

```bash
cd /opt
gunzip -c rw-sub-proxy_latest_linux_amd64.tar.gz | docker load
docker image ls | grep rw-sub-proxy
```

### 步骤 4：准备环境文件

```bash
cd /opt/rw-sub-proxy/deploy
cp .env.example .env
mkdir -p data postgres_data redis_data
```

编辑 `.env`，至少改这些：

```env
APP_IMAGE=rw-sub-proxy:latest
BIND_HOST=127.0.0.1
SERVER_PORT=9091

POSTGRES_PASSWORD=改成强密码
POSTGRES_USER=sub2api
POSTGRES_DB=sub2api

ADMIN_EMAIL=admin@你的域名
ADMIN_PASSWORD=改成强密码

JWT_SECRET=自己生成的32字节hex
TOTP_ENCRYPTION_KEY=自己生成的32字节hex

TZ=Asia/Shanghai

POSTGRES_MAX_CONNECTIONS=200
POSTGRES_SHARED_BUFFERS=128MB
DATABASE_MAX_OPEN_CONNS=20
DATABASE_MAX_IDLE_CONNS=5
REDIS_POOL_SIZE=128
REDIS_MIN_IDLE_CONNS=2
```

生成密钥示例：

```bash
openssl rand -hex 32
openssl rand -hex 32
```

### 步骤 5：启动服务

```bash
cd /opt/rw-sub-proxy/deploy
docker compose -f docker-compose.server.yml up -d
docker compose -f docker-compose.server.yml ps
docker compose -f docker-compose.server.yml logs -f sub2api
```

如果你前面设置的是：

- `BIND_HOST=127.0.0.1`
- `SERVER_PORT=9091`

那就用 Nginx / Caddy 反代到：

```text
http://127.0.0.1:9091
```

如果你直接想公网访问，就把 `BIND_HOST` 改成：

```env
BIND_HOST=0.0.0.0
```

---

## 方案 B：服务器直接源码构建

### 适用场景

- 至少 `2核4G`
- 或者你明确知道这台机器能跑完 Docker 构建

### 步骤

```bash
cd /opt
git clone https://github.com/zrwsmd/rw-sub-proxy.git
cd /opt/rw-sub-proxy/deploy
cp .env.example .env
mkdir -p data postgres_data redis_data
```

然后按需要改 `.env`。

如果你的 Docker 比较老，先加兼容环境变量：

```bash
export DOCKER_API_VERSION=1.41
export DOCKER_BUILDKIT=0
```

再构建启动：

```bash
docker compose -f docker-compose.dev.yml build
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml ps
docker compose -f docker-compose.dev.yml logs -f sub2api
```

### 不推荐在 1核1G 上这样做

原因：

- 前端 `vite build` 很吃内存
- Go 后端嵌入前端资源时也会继续消耗内存
- 构建缓存很吃磁盘

---

## 常见问题

### 1. `client version 1.52 is too new`

这是 Docker 客户端 / buildx 太新，但 daemon 太旧。

临时兼容方式：

```bash
export DOCKER_API_VERSION=1.41
export DOCKER_BUILDKIT=0
```

如果以后长期用源码构建，建议统一升级 Docker Engine 和 Compose / buildx。

### 2. `JavaScript heap out of memory`

这说明是前端构建内存不够。

处理优先级：

1. 不要在 1G 小机器源码构建
2. 改用 **方案 A**
3. 实在要在服务器构建，再考虑临时加 swap

### 3. `git pull` 提示本地文件会被覆盖

比如：

```text
Your local changes to the following files would be overwritten by merge:
deploy/docker-compose.dev.yml
```

先恢复这个文件，再拉代码：

```bash
git checkout -- deploy/docker-compose.dev.yml
git pull
```

### 4. `docker compose logs -f sub2api` 没输出

通常有两种情况：

1. 容器还没真正启动成功
2. 你卡在镜像构建阶段，还没进入运行阶段

先看状态：

```bash
docker compose -f docker-compose.server.yml ps -a
```

或：

```bash
docker compose -f docker-compose.dev.yml ps -a
```

---

## 更新方式

### 方案 A 更新

1. 在别的机器重新构建镜像
2. 导出并传到服务器
3. 服务器加载新镜像
4. 重启服务

命令示例：

```bash
docker build --platform linux/amd64 -t rw-sub-proxy:latest .
docker save rw-sub-proxy:latest | gzip > rw-sub-proxy_latest_linux_amd64.tar.gz
scp rw-sub-proxy_latest_linux_amd64.tar.gz root@your-server:/opt/
```

服务器执行：

```bash
cd /opt
gunzip -c rw-sub-proxy_latest_linux_amd64.tar.gz | docker load
cd /opt/rw-sub-proxy/deploy
docker compose -f docker-compose.server.yml up -d
```

### 方案 B 更新

```bash
cd /opt/rw-sub-proxy
git pull
cd deploy
export DOCKER_API_VERSION=1.41
export DOCKER_BUILDKIT=0
docker compose -f docker-compose.dev.yml build
docker compose -f docker-compose.dev.yml up -d
```

---

## 清理和回退

### 停止当前部署

方案 A：

```bash
cd /opt/rw-sub-proxy/deploy
docker compose -f docker-compose.server.yml down --remove-orphans
```

方案 B：

```bash
cd /opt/rw-sub-proxy/deploy
docker compose -f docker-compose.dev.yml down --remove-orphans
```

### 清理失败构建缓存

```bash
docker builder prune -af
docker image prune -af
```

### 删除整个部署目录

```bash
rm -rf /opt/rw-sub-proxy
```

### 如果加过 swap，回退

```bash
swapoff /swapfile 2>/dev/null || true
rm -f /swapfile
sed -i '\|/swapfile none swap sw 0 0|d' /etc/fstab
```

---

## 最终建议

以后默认记住这一条：

**1核1G 服务器不要做源码构建，直接运行外部构建好的镜像。**

如果只是想稳定上线服务，优先用：

- `docker-compose.server.yml`

如果只是想开发调试源码，再用：

- `docker-compose.dev.yml`
