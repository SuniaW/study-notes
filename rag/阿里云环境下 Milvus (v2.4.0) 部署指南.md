
# 阿里云环境下 Milvus (v2.4.0) 部署指南

## 1. 问题背景
在执行 `docker compose up -d` 时，由于网络环境限制，拉取 Docker Hub 官方镜像（如 `milvusdb/milvus` 或 `minio/minio`）经常会出现以下报错：
*   `dial tcp 52.175.9.80:443: i/o timeout`
*   `net/http: TLS handshake timeout`
*   `failed to resolve reference`

---

## 2. 解决方案步骤

### 第一步：配置阿里云专属镜像加速器
这是解决下载失败**最有效**的方法。阿里云为每个账号提供免费的私有加速地址。

1.  登录 [阿里云容器镜像服务控制台](https://cr.console.aliyun.com/)。
2.  在左侧菜单点击 **镜像工具** -> **镜像加速器**。
3.  复制你的**加速器地址**（例如：`https://xxxxxx.mirror.aliyuncs.com`）。
4.  在服务器上执行以下命令（请替换下方代码中的地址）：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "你的阿里云加速器地址",
    "https://atomhub.openatom.cn",
    "https://docker.m.daocloud.io",
    "https://mirror.baidubce.com"
  ]
}
EOF
# 重启 Docker 服务
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

### 第二步：准备 Milvus 配置文件
重新下载官方配置文件，并解决 `version` 属性过时的警告。

```bash
# 创建并进入目录
mkdir -p /usr/milvus && cd /usr/milvus

# 下载官方 Docker Compose 配置文件
wget https://github.com/milvus-io/milvus/releases/download/v2.4.0/milvus-standalone-docker-compose.yml -O docker-compose.yml

# 移除过时的 version 标签（消除 WARN 警告）
sed -i '/version:/d' docker-compose.yml
```

---

### 第三步：手动分步拉取镜像（推荐）
为了避免同时拉取多个大镜像导致带宽占满引起超时，建议逐一手动拉取：

```bash
# 1. 拉取 etcd (通常较快)
docker pull quay.io/coreos/etcd:v3.5.5

# 2. 拉取 MinIO (约 100MB+)
docker pull minio/minio:RELEASE.2023-03-20T20-16-18Z

# 3. 拉取 Milvus 主程序 (约 400MB+，最容易超时)
docker pull milvusdb/milvus:v2.4.0
```

**提示：** 如果此时 `docker pull` 依然非常慢，说明加速器未生效或当前节点拥堵。可以尝试直接在 `docker-compose.yml` 中将镜像地址修改为国内源（如 `atomhub.openatom.cn/milvusdb/milvus:v2.4.0`）。

---

### 第四步：启动 Milvus 服务
镜像全部下载完成后，一键启动：

```bash
docker compose up -d
```

---

## 3. 部署后检查

### 1. 检查容器状态
确认所有容器的状态为 `Up` (Running)：
```bash
docker compose ps
```
预期输出应包含：`milvus-standalone`, `milvus-minio`, `milvus-etcd`。

### 2. 检查网络访问
Milvus 默认使用 **19530** 端口。请确保：
*   **阿里云安全组：** 已在阿里云控制台的安全组规则中开放 `19530` 端口。
*   **服务器防火墙：** 如果开启了 `ufw` 或 `firewalld`，需允许该端口。

### 3. 日志排查（如启动失败）
如果某个容器没启动成功，查看日志：
```bash
docker compose logs milvus-standalone
```

---

## 4. 维护建议
*   **停止服务：** `docker compose down`
*   **查看资源占用：** `docker stats`
*   **数据持久化：** 默认数据存放在当前目录下的 `volumes/` 文件夹中，请勿随便删除。