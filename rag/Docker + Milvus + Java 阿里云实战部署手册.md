

# 🐳 Docker + Milvus + Java 阿里云实战部署手册

## 第一部分：环境安装与网络优化 (Ubuntu 22.04)

在阿里云国内环境，安装和镜像拉取速度是第一痛点。

### 1.1 一键极速安装 Docker
使用阿里云镜像源脚本，自动处理依赖并安装最新版 Docker 及 Compose V2。
```bash
# 更新索引并安装
sudo apt-get update
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

# 启动并设置开机自启
sudo systemctl start docker
sudo systemctl enable docker

# 免 sudo 权限优化（执行后建议重新连接 SSH）
sudo usermod -aG docker $USER
newgrp docker 
```

### 1.2 核心配置：镜像加速器
**必须配置**，否则 `docker pull` 极易失败。建议在阿里云控制台获取“专属加速地址”。
- **文件路径**：`/etc/docker/daemon.json`
```json
{
  "registry-mirrors": [
    "https://你的阿里云专属加速地址", 
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.nju.edu.cn"
  ]
}
```
**生效命令**：
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 第二部分：架构思维——从单兵到团队

| 维度 | Docker (单兵) | Docker Compose (团队) |
| :--- | :--- | :--- |
| **定义** | 关注单个容器的生命周期 | 关注多容器协同（编排） |
| **适用场景** | 运行一个独立的 Redis 或 Nginx | 运行 Java + Milvus + MySQL 全家桶 |
| **优势** | 简单直接 | **一键启动**、配置可复用、网络自动隔离 |
| **典型命令** | `docker run ...` (参数极长) | `docker compose up -d` (读取 .yml 文件) |

---

## 第三部分：实战部署 Milvus + Java 服务

### 3.1 推荐项目结构
建议将所有编排文件放在统一目录，便于管理和持久化。
```text
/opt/my-app/
├── docker-compose.yml     # 定义 Milvus, Etcd, Minio, JavaApp
└── java-service/
    ├── Dockerfile         # Java 镜像构建文件
    └── target/app.jar     # 你的 Java 程序
```

### 3.2 部署核心要点
1.  **服务发现 (Networking)**：
    *   在 Compose 网络中，Java 连接 Milvus 的地址应设为服务名：`milvus-standalone`，而非 `localhost`。
    *   默认端口：`19530`。
2.  **数据持久化**：
    *   在 `.yml` 中务必配置 `volumes`，将 Milvus 数据挂载到宿主机，防止容器删除后向量数据丢失。
3.  **安全组开通 (阿里云后台)**：
    *   **19530**：Milvus 远程连接。
    *   **9000**：MinIO 或 Portainer 界面。
    *   **8080**：Java 服务对外接口。

---

## 第四部分：可视化管理 (Portainer)

### 4.1 安装与“超时保护”机制
Portainer 启动后 5-10 分钟内若未设置密码，会自动关闭初始化页面以保安全。

*   **安装命令**：
    ```bash
    docker run -d -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce
    ```
*   **无法设置密码怎么办？**
    只需重启容器即可重置 5 分钟计时器：
    ```bash
    docker restart portainer
    ```
*   **后续操作**：立即访问 `http://服务器IP:9000` 并设置 12 位以上的强密码。

---

## 第五部分：故障排查与常用命令小抄

### 5.1 常见报错处理
*   **`i/o timeout`**：镜像站失效。对策：检查 `daemon.json` 是否有误，或尝试更换最新的镜像源。
*   **`the attribute version is obsolete`**：Docker Compose 升级后的警告。对策：直接删除 `.yml` 文件顶部的 `version: '3.x'` 即可。
*   **`Job for docker.service failed`**：通常是 `daemon.json` 里的 JSON 格式写错（如少了个逗号）。
*   **Windows 报错**：执行 `wsl --update` 并确保 BIOS 开启了虚拟化。

### 5.2 高频命令手册
| 功能 | 命令 (Docker Compose V2) |
| :--- | :--- |
| **启动所有服务** | `docker compose up -d` |
| **停止并删除容器** | `docker compose down` |
| **查看服务实时日志** | `docker logs -f <容器名>` |
| **进入容器环境** | `docker exec -it <容器名> bash` |
| **查看容器资源占用** | `docker stats` |
| **清理无用的镜像/缓存** | `docker system prune -a` |

---

## 六、 结语与进阶建议
1.  **定期备份**：对挂载的 `volumes` 目录进行定期备份，这是你向量数据库的命脉。
2.  **网络隔离**：在生产环境，尽量不要将数据库端口（如 Etcd, MinIO）暴露给公网，仅通过 Java 服务内部访问。
3.  **保持更新**：国内镜像站波动频繁，建议收藏 1-2 个稳定的备用镜像源地址。

---
**祝你的 Docker 实战之路顺畅！**