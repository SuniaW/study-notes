

# Docker 容器化技术实战手册

## 一、 核心概念：Docker 与 Docker Compose

| 特性 | Docker (基础) | Docker Compose (编排) |
| :--- | :--- | :--- |
| **定义** | 用于创建和运行**单个**容器。 | 用于定义和运行**多容器**应用程序。 |
| **比喻** | 像是一块砖头。 | 像是建筑图纸，规定了砖头如何堆叠。 |
| **场景** | 运行一个独立的 Redis 或 Nginx。 | 运行 Java + Milvus + MySQL 组成的完整系统。 |
| **操作** | 命令行参数冗长，难以记录。 | 通过一个 `docker-compose.yml` 文件一键启动。 |

---

## 二、 环境搭建（针对阿里云 ECS / Ubuntu 22.04）

### 1. 快速安装 Docker & Compose
在阿里云 Ubuntu 服务器上，推荐使用官方脚本配合阿里云镜像源：
```bash
# 一键安装脚本
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

# 启动并设置开机自启
sudo systemctl start docker
sudo systemctl enable docker
```

### 2. 核心配置：解决“镜像拉取超时” (必做)
由于网络环境原因，下载镜像常出现 `i/o timeout`。必须配置镜像加速器。
编辑文件 `/etc/docker/daemon.json`：
```json
{
  "registry-mirrors": [
    "https://你的阿里云专属加速器地址",
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```
**生效配置：**
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 三、 实战：部署 Milvus 与 Java 服务

推荐使用 **Docker Compose** 进行统一编排。Milvus 需要 Etcd（元数据）和 MinIO（存储）支撑。

### 1. 目录结构
```text
milvus-java-app/
├── docker-compose.yml    # 定义所有容器
└── java-app/
    ├── Dockerfile        # Java 镜像构建规则
    └── app.jar           # 你的 Java 程序包
```

### 2. Java 服务配置关键点
*   **连接地址：** 在 Docker 网络中，Java 连接 Milvus 的 Host 必须写服务名 `milvus-standalone`，不能写 `localhost`。
*   **代码示例：**
    ```java
    ConnectParam.newBuilder().withHost("milvus-standalone").withPort(19530).build();
    ```

### 3. 一键部署命令
```bash
# 在根目录下执行
docker compose up -d
```

---

## 四、 常见错误排查 (FAQ)

### 1. 镜像拉取失败 (`TLS handshake timeout`)
*   **原因：** 镜像源负载过高或被封锁。
*   **解决：**
    *   检查 `daemon.json` 格式（逗号、双引号）。
    *   使用阿里云控制台分配的**专属加速地址**。
    *   尝试手动分步拉取：`docker pull milvusdb/milvus:v2.4.0`。

### 2. Docker 启动失败 (`Job for docker.service failed`)
*   **原因：** 通常是因为 `daemon.json` 里的 JSON 格式错误。
*   **解决：** 检查是否有非法字符、少写逗号或括号不匹配。

### 3. Portainer 页面无法打开
*   **原因：** 阿里云安全组未放行 9000 端口。
*   **解决：** 在阿里云控制台的“安全组规则”中，添加入方向规则，协议 TCP，端口 9000，授权对象 `0.0.0.0/0`。

### 4. `the attribute version is obsolete` 警告
*   **原因：** 新版 Docker Compose 不再强制要求文件开头写 `version: '3.x'`。
*   **解决：** 直接删除 `.yml` 文件第一行的 `version` 定义即可。

---

## 五、 Docker 常用管理命令“小抄”

| 功能 | 命令 |
| :--- | :--- |
| **启动所有服务** | `docker compose up -d` |
| **停止并销毁容器** | `docker compose down` |
| **查看运行中的容器** | `docker ps` |
| **查看容器日志** | `docker logs -f <容器名或ID>` |
| **进入容器内部终端** | `docker exec -it <容器名> bash` |
| **删除镜像** | `docker rmi <镜像ID>` |
| **查看本地镜像** | `docker images` |
| **重启某个服务** | `docker restart <服务名>` |

---

## 六、 进阶建议
1.  **数据持久化：** 在 `docker-compose.yml` 中配置 `volumes`，确保容器重启后 Milvus 里的向量数据不丢失。
2.  **可视化管理：** 部署 **Portainer** 容器，通过 Web 界面点击鼠标即可管理所有镜像和容器，非常直观。
3.  **内网通信：** 容器间相互访问时，直接通过服务名（如 `db`, `milvus`）访问，效率最高且安全。