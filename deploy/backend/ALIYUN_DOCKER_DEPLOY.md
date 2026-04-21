# 阿里云 Docker Compose 生产部署指南

本文档说明如何将当前后端项目部署到阿里云 ECS。推荐第一版使用 ECS + Docker Compose + 阿里云容器镜像服务 ACR，后续有弹性伸缩或多节点需求时再迁移到 ACK/Kubernetes。

## 1. 部署架构

```text
用户 -> 域名 -> ECS 安全组 80/443 -> fba_nginx -> fba_server
                                           |-> fba_postgres
                                           |-> fba_redis
                                           |-> fba_rabbitmq
                                           |-> fba_celery_worker / beat / flower
```

生产模板文件：

- `deploy/backend/docker-compose/docker-compose.prod.yml`
- `deploy/backend/docker-compose/.env.docker.prod.example`
- `deploy/backend/docker-compose/.env.server.prod.example`
- `deploy/backend/nginx.prod.conf`

## 2. 阿里云准备

### 2.1 ECS

建议配置：

- 测试环境：2C4G
- 生产基础版：4C8G 起步
- 系统：Alibaba Cloud Linux 3/4 或 Ubuntu 22.04
- 磁盘：建议单独挂载数据盘，至少保留 40G+

### 2.2 安全组

公网只放行：

```text
22/tcp   SSH，建议只允许你的固定公网 IP
80/tcp   HTTP
443/tcp  HTTPS
```

不要对公网开放 PostgreSQL、Redis、RabbitMQ、Flower、Grafana、Prometheus 等内部端口。

### 2.3 ACR

在阿里云容器镜像服务 ACR 创建命名空间和镜像仓库，例如：

```text
地域：cn-shanghai
命名空间：stone_spaces
仓库：origin-test-backend
```

镜像地址示例：

```text
crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:20260421-server
```

当前部署使用 ACR 公网地址：

```text
crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com
```

如果 ECS 能稳定访问 ACR 专有网络地址，也可以改用 `-vpc` 地址：

```text
crpi-24v7nvpntv8628sf-vpc.cn-shanghai.personal.cr.aliyuncs.com
```

如果访问该地址出现 `context deadline exceeded`，说明当前 ECS 到 ACR VPC Endpoint 不通，先使用公网地址部署。

## 3. 构建并推送镜像

在本地或 CI 机器执行。根据当前 Dockerfile，需要分别构建 server、worker、beat、flower 四个镜像。

```bash
export REGISTRY=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com
export NAMESPACE=stone_spaces
export REPO=origin-test-backend
export TAG=20260421
```

登录 ACR：

```bash
docker login ${REGISTRY}
```

构建并推送：

```bash
docker build --build-arg SERVER_TYPE=fba_server \
  -t ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-server .
docker push ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-server

docker build --build-arg SERVER_TYPE=fba_celery_worker \
  -t ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-worker .
docker push ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-worker

docker build --build-arg SERVER_TYPE=fba_celery_beat \
  -t ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-beat .
docker push ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-beat

docker build --build-arg SERVER_TYPE=fba_celery_flower \
  -t ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-flower .
docker push ${REGISTRY}/${NAMESPACE}/${REPO}:${TAG}-flower
```

## 4. 准备服务器目录

登录 ECS：

```bash
ssh root@<ecs-public-ip>
```

安装 Docker 和 Docker Compose 后检查：

```bash
docker version
docker compose version
```

创建部署目录：

```bash
mkdir -p /opt/origin-test/{logs/fba,ssl,backup}
cd /opt/origin-test
```

上传或复制以下文件到 `/opt/origin-test`：

```text
docker-compose.prod.yml
.env.docker.prod
.env.server.prod
nginx.prod.conf
ssl/
```

可以从项目中复制：

```bash
cp deploy/backend/docker-compose/docker-compose.prod.yml /opt/origin-test/
cp deploy/backend/docker-compose/.env.docker.prod /opt/origin-test/.env.docker.prod
cp deploy/backend/docker-compose/.env.server.prod /opt/origin-test/.env.server.prod
cp deploy/backend/nginx.prod.conf /opt/origin-test/nginx.prod.conf
```

## 5. 修改生产配置

### 5.1 `.env.docker.prod`

必须替换镜像地址和密码：

```text
FBA_SERVER_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:20260421-server
FBA_CELERY_WORKER_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:20260421-worker
FBA_CELERY_BEAT_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:20260421-beat
FBA_CELERY_FLOWER_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:20260421-flower

POSTGRES_PASSWORD=<strong-postgres-password>
REDIS_PASSWORD=<strong-redis-password>
RABBITMQ_DEFAULT_PASS=<strong-rabbitmq-password>
```

### 5.2 `.env.server.prod`

这里的数据库、Redis、RabbitMQ 密码必须和 `.env.docker.prod` 保持一致：

```text
DATABASE_PASSWORD='<strong-postgres-password>'
REDIS_PASSWORD='<strong-redis-password>'
CELERY_RABBITMQ_PASSWORD='<strong-rabbitmq-password>'
```

生成新的 `TOKEN_SECRET_KEY`：

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

然后替换：

```text
TOKEN_SECRET_KEY='<generated-secret>'
```

### 5.3 `nginx.prod.conf`

先用 HTTP 可以不改 `server_name`，如果绑定域名，建议改为你的域名：

```nginx
server_name api.example.com;
```

开启 HTTPS 时：

1. 将证书上传到 `/opt/origin-test/ssl`
2. 取消 `nginx.prod.conf` 里的 443 server block 注释
3. 替换 `example.com.pem` 和 `example.com.key`
4. 将 80 server block 改成跳转 HTTPS

## 6. 启动服务

登录 ACR：

```bash
docker login crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com
```

拉取镜像：

```bash
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod pull
```

启动：

```bash
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod up -d
```

查看状态：

```bash
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod ps
docker logs -f fba_server
docker logs -f fba_nginx
```

验证接口：

```bash
curl -I http://127.0.0.1
curl -I http://<ecs-public-ip>
```

如果 ECS 拉取 `nginx/postgres/redis/rabbitmq` 时出现 Docker Hub 超时：

```text
Get "https://registry-1.docker.io/v2/": context deadline exceeded
```

说明 ECS 访问 Docker Hub 不稳定。生产 Compose 已支持将这些基础镜像也改为从 ACR 拉取：

```text
NGINX_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:base-nginx-1.27
POSTGRES_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:base-postgres-16
REDIS_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:base-redis-7
RABBITMQ_IMAGE=crpi-24v7nvpntv8628sf.cn-shanghai.personal.cr.aliyuncs.com/stone_spaces/origin-test-backend:base-rabbitmq-3.13.7-management
```

GitHub Actions 会自动把这些 Docker Hub 公共镜像同步到你的 ACR，ECS 部署时只从 ACR 拉取。

## 7. 常用运维命令

查看服务：

```bash
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod ps
```

重启：

```bash
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod restart
```

查看日志：

```bash
docker logs -f fba_server
docker logs -f fba_celery_worker
docker logs -f fba_nginx
```

停止：

```bash
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod down
```

不要随意执行 `docker compose down -v`，它会删除数据库、Redis、RabbitMQ 的命名卷数据。

## 8. 升级发布

1. 构建并推送新 tag 镜像
2. 修改 `/opt/origin-test/.env.docker.prod` 中的镜像 tag
3. 拉取并重启

```bash
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod pull
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod down --remove-orphans
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod up -d
```

如果发布失败，改回上一个 tag 后重新执行上面三条命令。

## 9. GitHub Actions 自动化部署

仓库已提供自动化部署 workflow：

```text
.github/workflows/deploy-aliyun.yml
```

触发方式：

- 推送到 `master` 分支自动部署
- 在 GitHub Actions 页面手动执行 `Deploy to Alibaba Cloud ECS`

流水线做四件事：

1. 从 GitHub 拉取代码
2. 将 `nginx`、`postgres`、`redis`、`rabbitmq` 公共镜像同步到阿里云 ACR
3. 构建 `server`、`worker`、`beat`、`flower` 四个镜像
4. 推送镜像到阿里云 ACR
5. 上传最新 `docker-compose.prod.yml` 和 `nginx.prod.conf` 到 ECS
6. SSH 登录 ECS，更新 `.env.docker.prod` 的镜像 tag，并执行 `docker-compose pull && docker-compose down --remove-orphans && docker-compose up -d`

### 9.1 GitHub Secrets

在 GitHub 仓库中进入：

```text
Settings -> Secrets and variables -> Actions -> New repository secret
```

添加以下 Secrets：

```text
ACR_USERNAME=stone_study
ACR_PASSWORD=<阿里云 ACR 登录密码>

ECS_USER=root
ECS_SSH_PRIVATE_KEY=<部署用 SSH 私钥内容>
ECS_DEPLOY_DIR=/opt/origin-test
```

ECS 公网 IP 已在 workflow 中配置为：

```text
39.102.84.196
```

`ECS_DEPLOY_DIR` 可不填，默认是 `/opt/origin-test`。

### 9.2 ECS SSH 密钥

建议创建一对专门用于 GitHub Actions 部署的 SSH key，不要复用个人登录密钥。

在本地生成：

```bash
ssh-keygen -t ed25519 -C "github-actions-origin-test" -f ./github-actions-origin-test
```

把公钥追加到 ECS：

```bash
ssh-copy-id -i ./github-actions-origin-test.pub root@39.102.84.196
```

把私钥内容填入 GitHub Secret：

```bash
cat ./github-actions-origin-test
```

Secret `ECS_SSH_PRIVATE_KEY` 必须填写完整私钥内容，包含首尾行，例如：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

不要填写 `.pub` 公钥内容，也不要填写 PuTTY `.ppk` 格式。如果是在 Windows 上复制，确保 GitHub Secret 中保留多行换行。

如果不用 `root` 用户，确保部署用户有权限执行 Docker。通常需要加入 `docker` 用户组，或在 workflow 里改成使用 `sudo docker`。

### 9.3 ECS 首次部署要求

自动化部署前，ECS 上必须已经存在这些文件：

```text
/opt/origin-test/docker-compose.prod.yml
/opt/origin-test/.env.docker.prod
/opt/origin-test/.env.server.prod
/opt/origin-test/nginx.prod.conf
/opt/origin-test/ssl/
```

首次部署时手动执行一次：

```bash
cd /opt/origin-test
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod config
```

确保配置能解析后，再触发 GitHub Actions。

### 9.4 自动部署后的验证

在 GitHub Actions 日志里确认最后几步成功：

```text
docker compose pull
docker compose down --remove-orphans
docker compose up -d
docker compose ps
```

当前 ECS 使用 `docker-compose 1.29.2` 时，重建已有容器可能出现：

```text
KeyError: 'ContainerConfig'
```

部署脚本会在 `up -d` 前执行 `down --remove-orphans` 删除旧容器后再新建。该命令不会删除命名卷数据；不要添加 `-v`。

在 ECS 上可以进一步检查：

```bash
cd /opt/origin-test
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod ps
docker logs --tail=100 fba_server
curl -I http://127.0.0.1
```

当前 ECS 使用旧版命令：

```bash
docker-compose version
```

所以 GitHub Actions 部署脚本固定使用 `docker-compose`。

如果日志出现：

```text
unknown shorthand flag: 'f' in -f
```

说明执行到了错误的 Docker CLI 命令，或服务器上的 Compose 命令不可用。登录 ECS 检查：

```bash
docker-compose version
```

该命令需要可用。

如果 Nginx 启动时报类似错误：

```text
error mounting "/var/lib/docker/volumes/fba_static_upload/_data" to ... "/www/fba_server/backend/static/upload": read-only file system
```

说明 ECS 上还在使用旧的 `docker-compose.prod.yml`。workflow 会在部署前自动上传最新 Compose 和 Nginx 配置；如果手动部署，需要重新复制这两个文件到 `/opt/origin-test`。

### 9.5 自动部署回滚

每次部署前，workflow 会在 ECS 上备份 `.env.docker.prod`：

```text
.env.docker.prod.bak.YYYYMMDDHHMMSS
```

如果新版本失败，可以在 ECS 上回滚：

```bash
cd /opt/origin-test
cp .env.docker.prod.bak.<timestamp> .env.docker.prod
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod pull
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod down --remove-orphans
docker compose -f docker-compose.prod.yml --env-file .env.docker.prod up -d
```

## 10. 数据备份

PostgreSQL 备份：

```bash
mkdir -p /opt/origin-test/backup
docker exec fba_postgres pg_dump -U postgres -d fba > /opt/origin-test/backup/fba_$(date +%F_%H%M%S).sql
```

Redis 使用 AOF 持久化，数据在 Docker volume `fba_redis` 中。重要生产环境建议增加定期快照和异地备份。

## 11. 生产建议

- 生产环境优先使用阿里云 RDS PostgreSQL 和云数据库 Redis，减少自维护数据库风险。
- 内部服务不要映射公网端口。
- `.env.server.prod` 和 `.env.docker.prod` 不要提交到公开仓库。
- 定期备份 PostgreSQL。
- ECS 安全组的 SSH 只允许固定 IP。
- 对 `/flower/` 这类管理入口增加访问限制，或只通过内网/SSH 隧道访问。
