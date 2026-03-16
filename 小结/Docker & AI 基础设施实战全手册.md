# 🐳 Docker & AI 基础设施实战全手册
**（涵盖 Docker、Docker Compose、Milvus 向量数据库、Java 部署及 Ollama 模型管理）**

---

## 第一部分：核心概念与架构规范

### 1.1 Docker 与 Docker Compose 的关系
| 特性 | Docker (基础/单兵) | Docker Compose (编排/团队) |
| :--- | :--- | :--- |
| **定义** | 用于创建和运行**单个**容器。 | 用于定义和运行**多容器**应用程序。 |
| **比喻** | 像是一块**砖头**（单个容器）。 | 像是**建筑图纸**，规定砖头如何堆叠成房子。 |
| **配置方式** | 命令行参数冗长（`docker run ...`）。 | YAML 配置文件（`docker-compose.yml`）。 |
| **场景** | 测试单个服务、运行单一工具。 | 运行包含 Java + Milvus + MySQL 的完整系统。 |

### 1.2 架构黄金准则：One Process Per Container
**结论：一个容器只部署一个微服务。**
若强行在单一容器内运行多个服务（如 Nginx+MySQL+Java），会面临以下致命问题：
1.  **无法独立扩展**：无法针对单个高负载服务进行动态扩容。
2.  **部署耦合**：修改一处代码需重启整个大容器，导致所有服务停机。
3.  **进程管理困难 (PID 1 问题)**：子服务崩溃可能无法触发 Docker 的自动重启机制。
4.  **日志混乱**：多服务日志交织，排错极其困难。
5.  **违背微服务初衷**：在物理层面强行耦合了逻辑上解耦的服务。

---

## 第二部分：环境搭建与阿里云专属优化

### 2.1 阿里云 Ubuntu 22.04 极速安装
在阿里云国内环境下，请务必使用镜像源加速：

```bash
# 1. 一键安装 Docker 及 Compose 插件
sudo apt-get update
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

# 2. 设置开机自启
sudo systemctl enable docker
sudo systemctl start docker

# 3. 免 sudo 权限优化（执行后建议重新连接 SSH）
sudo usermod -aG docker $USER
newgrp docker 
```

### 2.2 核心避坑：配置镜像加速器 (必做)
国内拉取镜像极易报 `i/o timeout` 或 `TLS handshake timeout`，必须配置。
编辑 `/etc/docker/daemon.json`：
```json
{
  "registry-mirrors": [
    "https://你的阿里云专属加速器地址",
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.nju.edu.cn",
    "https://docker.mirrors.sjtug.sjtu.edu.cn",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```
**生效命令：**
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
*注意：若重启失败，请检查 JSON 格式（如逗号、引号是否匹配）。*

### 2.3 Windows 虚拟化报错排查
*   **报错 `WSL needs updating`**：运行 `wsl --update` 并 `wsl --shutdown`。
*   **报错 `Virtualization support not detected`**：
    1.  BIOS 中开启 `VT-x` 或 `SVM`。
    2.  Windows 功能中勾选 `虚拟机平台` 和 `适用于 Linux 的 Windows 子系统`。
*   **🚨 阿里云 Windows ECS 警告**：云服务器（虚拟机）默认不支持嵌套虚拟化。若在 Windows ECS 上跑 Docker 极度卡顿或报错，**强烈建议重装为 Ubuntu 系统**。

---

## 第三部分：Milvus 向量数据库深度解析

### 3.1 核心架构与特性
Milvus 是一款云原生向量数据库，专为处理高维向量（Embedding）设计。
*   **存算分离**：接入层、协调层、执行层和存储层完全分离，支持极致扩展。
*   **混合搜索**：支持“向量搜索 + 标量过滤”，并于 2.5+ 版本集成 **BM25 全文检索**。
*   **硬件加速**：支持 NVIDIA GPU 加速（CAGRA 算法）。

### 3.2 主流向量数据库对比 (2025)
| 维度 | Milvus | Pinecone | Weaviate | Qdrant |
| :--- | :--- | :--- | :--- | :--- |
| **开源状态** | 完全开源 | 闭源 SaaS | 开源 | 开源 |
| **主要优势** | 万亿级规模处理能力 | 零运维成本 | 集成 GraphQL | Rust 编写、轻量级 |
| **部署环境** | K8s, Docker, Lite | 仅限云端 | Docker, Cloud | Docker, Cloud |

### 3.3 快速开始 (Milvus Lite)
通过 Python 即可快速测试：
```python
from pymilvus import MilvusClient
client = MilvusClient("demo_milvus.db")
client.create_collection(collection_name="my_rag", dimension=768)
# 插入与搜索代码略...
```

---

## 第四部分：Milvus + Java 服务实战部署

### 4.1 项目结构规范
```text
/opt/my-app/
├── docker-compose.yml     # 定义 Milvus, Etcd, Minio, JavaApp
└── java-service/
    ├── Dockerfile         # Java 镜像构建文件
    └── target/app.jar     # 你的 Java 程序
```

### 4.2 Java 服务 Dockerfile 示例
```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4.3 Docker Compose 编排模板
```yaml
# 现代版本可省略 version 字段
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
    volumes:
      - ./volumes/milvus:/var/lib/milvus # 必须配置持久化

  java-app:
    build: ./java-service
    container_name: my-java-app
    ports:
      - "8080:8080"
    environment:
      - MILVUS_HOST=milvus-standalone # 关键：使用服务名通信
    depends_on: ["milvus"]
```

### 4.4 部署关键要点
1.  **服务发现**：在 Docker 网络中，Java 连接 Milvus 的 Host 必须写服务名 `milvus-standalone`，**严禁写 `localhost`**。
2.  **安全组开放 (阿里云后台)**：
    *   **19530**：Milvus gRPC 接口。
    *   **9000/9001**：Portainer 或 MinIO 控制台。
    *   **8080**：Java API 接口。

---

## 第五部分：可视化管理 (Portainer)

### 5.1 安装命令
```bash
docker run -d -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce
```

### 5.2 安全避坑：Timed Out 报错
Portainer 启动后 5 分钟内若未设置密码，会出于安全原因自动关闭初始化页面。
*   **解决**：执行 `docker restart portainer` 重置计时器，然后立即访问 `http://服务器IP:9000` 设置强密码。

---

## 第六部分：Ollama 模型管理指南

### 6.1 常用管理命令
| 功能 | 命令 |
| :--- | :--- |
| **列出模型** | `ollama list` |
| **查看运行中** | `ollama ps` |
| **删除模型** | `ollama rm <模型名>` |
| **拉取模型** | `ollama pull <模型名>` |

### 6.2 BGE 向量模型避坑
执行 `ollama pull bge-small-zh` 若报 `file does not exist`，是因为官方库名称不匹配。
*   **推荐替代方案**：使用 **BGE-M3**（支持 80+ 语言，含中文最佳）。
    ```bash
    ollama pull bge-m3
    ```
*   **手动导入特定模型**：
    1. 下载 `.gguf` 文件。
    2. 创建 `Modelfile` 写入 `FROM ./xxx.gguf`。
    3. 执行 `ollama create <自定义名> -f Modelfile`。

---

## 第七部分：故障排查与维护 (Troubleshooting)

### 7.1 Milvus "Connection Refused" (19530 端口)
**报错**：`Connection refused: getsockopt: /8.140.xxx:19530`
1.  **验证网络**：本地执行 `telnet 8.140.xxx 19530`。若 Timeout 是安全组没开；若 Refused 是服务没起。
2.  **检查依赖**：查看日志 `docker logs milvus-standalone`，重点看 etcd 和 MinIO 是否连通。
3.  **重启大法**：执行 `docker compose down` 彻底清理后，再 `docker compose up -d`。

### 7.2 综合常见错误
*   **`the attribute version is obsolete`**：新版 Compose 不再需要文件开头的 `version: '3'`，直接删除即可。
*   **`Job for docker.service failed`**：通常是 `daemon.json` 格式错误（如少了个逗号）。
*   **磁盘爆满**：定期清理。执行 `docker system df` 检查，`docker system prune` 一键清理。

### 7.3 Docker 常用命令速查 (Cheat Sheet)
| 场景 | 命令 |
| :--- | :--- |
| **启动服务** | `docker compose up -d` |
| **停止并销毁** | `docker compose down` |
| **实时日志** | `docker logs -f <容器名>` |
| **进入容器** | `docker exec -it <容器名> bash` |
| **清理空间** | `docker system prune -a` |
| **查看资源占用** | `docker stats` |

---
**提示**：生产环境务必定期备份 `volumes` 挂载目录，那是你向量数据库的命脉。祝您的 Docker 实战之路顺畅！