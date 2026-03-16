
# 🐳 Docker 详解与实战部署手册

## 一、 Docker 核心概念简述
Docker 是一种容器化技术，通过将应用及其依赖打包在“容器”中，实现“一次构建，到处运行”。
*   **镜像 (Image)**：只读模板（如 Ubuntu + Java 环境）。
*   **容器 (Container)**：镜像运行时的实例（类似进程）。
*   **仓库 (Registry)**：存放镜像的地方（如 Docker Hub）。
*   **优势**：极其轻量、启动秒级、环境高度一致。

---

## 二、 基础环境配置与加速（必做）
在境内服务器访问 Docker Hub 经常超时，必须配置镜像加速器。

### 1. 配置镜像加速器
编辑或创建配置文件：
```bash
sudo vim /etc/docker/daemon.json
```
写入以下内容（确保 JSON 格式正确，不要有中文标点）：
```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```
### 2. 重启 Docker 服务
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
*注：如果重启失败，请检查文件是否有拼写错误。可用 `dockerd --debug` 排查。*

---

## 三、 实战：部署 Milvus 与 Java 服务
推荐使用 **Docker Compose** 统一管理多个容器。

### 1. 准备 Dockerfile (Java 服务)
在 `java-app` 目录下创建 `Dockerfile`：
```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY app.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2. 编写 docker-compose.yml
在根目录创建该文件，一键编排所有服务：
```yaml
version: '3.5'
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.5
    container_name: milvus-etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls=http://0.0.0.0:2379 --data-dir=/etcd

  minio:
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    container_name: milvus-minio
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    command: minio server /export --console-address ":9001"

  milvus:
    image: milvusdb/milvus:v2.3.15
    container_name: milvus-standalone
    ports:
      - "19530:19530"
    depends_on: ["etcd", "minio"]

  java-service:
    build: ./java-app
    container_name: my-java-app
    ports:
      - "8080:8080"
    environment:
      - MILVUS_HOST=milvus-standalone # 关键：通过容器名通信
    depends_on: ["milvus"]
```

### 3. 启动命令
```bash
docker compose up -d
```

---

## 四、 可视化管理：部署 Portainer
Portainer 提供网页界面来管理所有容器。

### 1. 运行 Portainer 容器
```bash
docker run -d -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce
```

### 2. 解决 "Timed out" 安全报错
如果你打开页面看到“Your Portainer instance timed out...”，说明你超过 5 分钟未设置密码。
*   **解决方法**：执行 `docker restart portainer` 重置计时器，然后立即访问页面设置管理员密码。

---

## 五、 阿里云服务器访问配置
在阿里云环境部署后，外部无法直接访问 9000 或 8080 端口，需完成以下两步：

### 1. 获取公网 IP
在服务器内部执行：
```bash
curl ifconfig.me
```

### 2. 放行安全组规则（防火墙）
1.  登录 **阿里云 ECS 控制台**。
2.  进入 **安全组** -> **配置规则**。
3.  **入方向** 添加手动规则：
    *   **协议**：TCP
    *   **端口范围**：`9000` (Portainer)、`8080` (Java)
    *   **源地址**：`0.0.0.0/0` (或指定你的本地 IP)

---

## 六、 常用维护指令速查
*   **查看运行中的容器**：`docker ps`
*   **查看容器日志**：`docker logs -f [容器名]`
*   **进入容器内部**：`docker exec -it [容器名] /bin/bash`
*   **强制删除所有容器**：`docker rm -f $(docker ps -aq)`
*   **清理无用镜像**：`docker image prune`

---
**提示**：在多容器环境中，容器间通信请务必使用 **服务名**（如 `milvus-standalone`），而不是 `localhost` 或 `127.0.0.1`。