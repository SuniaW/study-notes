# 🚀 阿里云 Docker + Milvus + Java 部署全攻略

## 一、 环境安装：阿里云 ECS (Ubuntu 22.04) 最佳实践

### 1. 极速安装 Docker & Docker Compose V2
使用阿里云镜像源，规避官方源下载缓慢的问题。
```bash
# 1. 更新系统索引并运行阿里云加速安装脚本
sudo apt-get update
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

# 2. 启动服务并设置开机自启
sudo systemctl start docker
sudo systemctl enable docker

# 3. 免 sudo 优化（执行后需重新连接终端）
sudo usermod -aG docker $USER
newgrp docker 
```

### 2. 配置镜像加速器（解决镜像下载超时）
由于网络限制，**必须**配置国内镜像源。建议前往阿里云控制台获取“专属加速地址”。
```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://你的阿里云专属加速地址", 
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.nju.edu.cn"
  ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 二、 核心概念：单兵 vs 团队
| 维度 | Docker (单兵) | Docker Compose (团队) |
| :--- | :--- | :--- |
| **关注点** | 单个容器的生命周期 | 多容器协同工作（Java + Milvus + DB） |
| **命令** | `docker run ...` (参数多，难记录) | `docker compose up -d` (配置可复用) |
| **场景** | 跑一个简单的 Redis | 部署复杂的微服务架构 |

---

## 三、 实战部署：Milvus + Java 服务

### 1. 项目结构推荐
```text
my-project/
├── docker-compose.yml     # 核心编排文件（包含 Milvus、Etcd、MinIO、JavaApp）
└── java-app/
    ├── Dockerfile         # Java 镜像定义
    └── app.jar            # 编译好的 Java 程序
```

### 2. 关键连接规则 (Networking)
*   **服务发现**：在 `docker-compose.yml` 中，Java 连接 Milvus 的 Host 应填服务名 `milvus-standalone`，端口 `19530`，**严禁**使用 `localhost` 或 `127.0.0.1`。
*   **安全组配置**：若需从本地电脑连接 ECS 上的服务，必须在阿里云控制台开放以下端口：
    *   `19530` (Milvus) / `9000` (MinIO/Portainer) / `8080` (Java API)。

---

## 四、 可视化管理：Portainer

Portainer 提供网页界面管理容器。**注意：其有 5-10 分钟的安全保护机制。**

### 1. 安装命令
```bash
docker run -d -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce
```

### 2. 无法访问/超时排查
*   **现象**：提示安全超时，无法设置密码。
*   **对策**：重启容器以重置计时器：`docker restart portainer`。
*   **后续**：立即访问 `http://服务器IP:9000` 设置 12 位以上密码。

---

## 五、 故障排查与常用命令汇总

### 1. 常见错误对策
*   **i/o timeout**：镜像源失效。更换 `daemon.json` 里的地址或使用阿里云个人镜像。
*   **attribute version is obsolete**：YAML 警告。删除 `docker-compose.yml` 第一行的 `version: '3.x'` 即可。
*   **Job for docker.service failed**：检查 `daemon.json` 格式（逗号、引号、大括号是否配对）。
*   **WSL/BIOS 报错**：Windows 环境需执行 `wsl --update` 或在 BIOS 开启虚拟化技术。

### 2. 高频命令小抄 (V2 规范)
| 功能 | 命令 |
| :--- | :--- |
| **一键启动整套服务** | `docker compose up -d` |
| **停止并删除所有服务** | `docker compose down` |
| **查看运行中的容器** | `docker ps` |
| **查看实时日志** | `docker logs -f <容器名>` |
| **进入容器内部终端** | `docker exec -it <容器名> bash` |
| **强制删除指定容器** | `docker rm -f <容器名>` |

---

## 六、 进阶建议
1.  **数据持久化**：务必在 Compose 文件中配置 `volumes`，否则 Milvus 里的向量数据和数据库内容会在容器删除后丢失。
2.  **镜像源维护**：国内镜像源波动频繁，建议定期检查 `daemon.json`。
3.  **安全性**：Portainer 设置完密码后，建议在阿里云安全组限制访问 9000 端口的来源 IP。

---
**提示：** 现在的重点是维护好 `daemon.json` 中的镜像源，这是在国内顺畅使用 Docker 的生命线。