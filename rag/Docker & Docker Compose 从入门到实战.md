

# 🐳 Docker & Docker Compose 从入门到实战
**—— 附阿里云 ECS (Ubuntu 22.04) 最佳部署与避坑指南**

## 目录
1. [核心概念：Docker 与 Docker Compose](#1-核心概念docker-与-docker-compose)
2. [架构规范：一个容器部署多少个微服务？](#2-架构规范一个容器部署多少个微服务)
3. [环境避坑：Windows 虚拟化与阿里云嵌套问题](#3-环境避坑windows-虚拟化与阿里云嵌套问题)
4. [最佳实践：在阿里云 Ubuntu 22.04 极速安装](#4-最佳实践在阿里云-ubuntu-2204-极速安装)
   5.[实战手册：Docker 常用命令速查 (Cheat Sheet)](#5-实战手册docker-常用命令速查)
6. [进阶使用：Docker Compose 模板化部署](#6-进阶使用docker-compose-模板化部署)
7. [阿里云专属避坑指南](#7-阿里云专属避坑指南)

---

## 1. 核心概念：Docker 与 Docker Compose

**简单来说：Docker 是基础，Docker Compose 是基于 Docker 的上层管理工具。**

如果用**建筑**来做比喻：
*   **Docker** 就像是**砖块**。你关注的是如何造好这一块砖（单个容器）。
*   **Docker Compose** 就像是**建筑图纸**。它规定了需要多少砖块、它们之间如何连接，最后通过一张图纸把整个房子（应用系统）建起来。

| 特性 | Docker | Docker Compose |
| :--- | :--- | :--- |
| **对象** | 单个容器 (Single Container) | 多容器应用栈 (Multi-container Stack) |
| **配置方式** | 命令行参数 (`docker run -p 80:80 nginx`) | YAML 配置文件 (`docker-compose.yml`) |
| **适用场景** | 测试单个服务、运行单一工具 | 运行包含前端、后端、数据库的完整应用 |

---

## 2. 架构规范：一个容器部署多少个微服务？

**结论：理论上可以装无数个，但最佳实践是“1 个容器只部署 1 个微服务”。**

业界黄金法则：**One Process Per Container**（一个容器一个主进程）。如果把 Nginx、MySQL、Java 服务强行塞进同一个容器（即“富容器”模式），会面临以下 5 个致命问题：

1.  **无法独立扩展：** 订单服务高并发需要扩容，却被迫连带扩容了不需要并发的用户服务，浪费资源。
2.  **构建与部署耦合：** 改了一行代码，必须重启包含所有服务的整个大容器，导致其他服务被迫停机。
3.  **进程管理困难 (PID 1 问题)：** Docker 依赖主进程判断容器死活。多服务下，子服务崩溃可能无法触发自动重启。
4.  **日志监控混乱：** 多个服务的日志交织在同一个标准输出中，排错如同噩梦。
5.  **违背微服务初衷：** 微服务旨在解耦，放进同个容器是物理层面的强行耦合。

> **正确做法：** 使用 Docker Compose 或 Kubernetes Pod，将相关联的服务分别打包成独立镜像，在同一网络下协同工作。

---

## 3. 环境避坑：Windows 虚拟化与阿里云嵌套问题

在 Windows 系统上安装 Docker Desktop 时，常遇到以下报错及解决思路：

### 报错 1：`WSL needs updating`
*   **原因：** Windows Linux 子系统内核过旧。
*   **解决：** 以管理员身份打开 PowerShell，运行 `wsl --update`，完成后运行 `wsl --shutdown` 重启服务。

### 报错 2：`Virtualization support not detected`
*   **原因：** CPU 虚拟化未开启，或 Windows 相关功能未激活。
*   **解决：**
    1. 检查任务管理器 -> 性能 -> CPU -> “虚拟化”是否为“已启用”。
    2. 若未启用，需重启进入主板 BIOS 开启 `VT-x` (Intel) 或 `SVM Mode` (AMD)。
    3. 若已启用仍报错：搜索“启用或关闭 Windows 功能”，勾选 `虚拟机平台` 和 `适用于 Linux 的 Windows 子系统`，并重启电脑。

### 🚨 阿里云 Windows ECS 专属大坑：嵌套虚拟化
如果你的 Windows 是**阿里云 ECS 云服务器**，默认是不支持运行 Docker Desktop 的！
*   **原因：** 云服务器本身已是虚拟机，默认不支持在虚拟机里再跑虚拟机（即不支持嵌套虚拟化/Hyper-V/WSL2）。
*   **最终解法：** 放弃在低配 Windows ECS 上死磕 Docker，**强烈建议重装系统为 Linux (Ubuntu)**，Docker 在 Linux 上是原生运行，无任何虚拟化损耗！

---

## 4. 最佳实践：在阿里云 Ubuntu 22.04 极速安装

重装为 Ubuntu 22.04 后，请执行以下标准流程（含国内源加速），只需 3 步：

**Step 1: 一键安装 Docker 及 Compose 插件**
```bash
sudo apt-get update
# 使用阿里云源飞速下载安装
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
# 设置开机自启
sudo systemctl enable docker
sudo systemctl start docker
```

**Step 2: 配置镜像加速器（极其重要，否则国内无法拉取镜像）**
```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors":[
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.nju.edu.cn",
    "https://docker.mirrors.sjtug.sjtu.edu.cn"
  ]
}
EOF
```

**Step 3: 重载配置并验证**
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证安装结果（现代版本 Docker 已内置 compose 命令）
docker --version
docker compose version
```

---

## 5. 实战手册：Docker 常用命令速查

| 场景 | 命令 | 说明 |
| :--- | :--- | :--- |
| **测试拉取** | `docker run hello-world` | 验证 Docker 是否正常工作 |
| **运行容器** | `docker run -d -p 80:80 --name my-web nginx` | `-d`后台运行，`-p`端口映射 |
| **查看运行中** | `docker ps` | 列出正在跑的容器 |
| **查看所有** | `docker ps -a` | 列出所有（含已停止的）容器 |
| **排错看日志** | `docker logs -f <容器名/ID>` | 排错神器，`Ctrl+C` 退出 |
| **进入容器** | `docker exec -it <容器名> bash` | 像 SSH 一样进入容器内部 |
| **启停控制** | `docker start / stop / restart <容器名>` | 启动/停止/重启 服务 |
| **删除容器** | `docker rm -f <容器名>` | `-f` 代表强制停止并删除 |
| **清理空间** | `docker system prune` | **慎用**，一键清理所有未使用的镜像/容器 |

---

## 6. 进阶使用：Docker Compose 模板化部署

当需要部署复杂应用时，摒弃冗长的 `docker run`，改写 `docker-compose.yml`。

*注：现代版本中，命令已从 `docker-compose` (V1) 升级为 `docker compose` (V2，去掉了横杠)。*

**1. 创建配置文件示例 (`docker-compose.yml`):**
```yaml
version: '3'
services:
  web:
    image: nginx:latest
    container_name: my-nginx
    restart: always       # 崩溃或开机自动重启
    ports:
      - "80:80"           # 主机端口:容器端口
    volumes:
      - ./html:/usr/share/nginx/html # 挂载本地目录到容器
```

**2. 管理命令:**
*   **一键启动后台运行:** `docker compose up -d`
*   **查看所有服务日志:** `docker compose logs -f`
*   **停止并销毁服务:** `docker compose down`

---

## 7. 阿里云专属避坑指南

在阿里云 ECS 环境下，除了 Docker 自身的机制，你必须注意云厂商的网络限制：

1.  **安全组未放行 (最常见的新手坑):**
    *   现象：容器启动成功（`docker ps` 显示 Up），但浏览器访问超时。
    *   解决：登录阿里云 ECS 控制台 -> 安全组 -> 入方向规则 -> **放行对应的 TCP 端口（如 80, 3306, 8080）**。
2.  **权限报错 (Permission Denied):**
    *   现象：普通用户执行 docker 命令报错。
    *   解决：每次加 `sudo` 执行，或者将用户加入 docker 组：`sudo usermod -aG docker $USER`（执行后需重新登录 SSH 终端生效）。
3.  **磁盘爆满:**
    *   现象：服务器无法写入文件，系统卡顿。
    *   解决：定期使用 `docker system df` 监控空间，使用 `docker image prune` 清理无用镜像。