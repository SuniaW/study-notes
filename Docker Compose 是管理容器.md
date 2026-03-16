使用 **Docker Compose** 是管理单容器或多容器应用的“黄金标准”。它能将你的运行参数（端口、镜像名、挂载卷等）持久化到配置文件中，避免每次手动敲一长串 `docker run` 命令。

针对你的项目地址 `git@github.com:SuniaW/web-ui.git`，以下是详细的专业化部署指南：

---

### 第一步：准备工作 (SSH Key)

因为你使用的是 Git SSH 地址 (`git@github.com...`)，你的阿里云服务器必须配置 SSH Key 并添加到 GitHub，否则无法拉取代码。

1.  **检查并生成密钥**（如果已有请跳过）：
    ```bash
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    # 一路回车即可
    ```
2.  **查看公钥**：
    ```bash
    cat ~/.ssh/id_rsa.pub
    ```
3.  **添加到 GitHub**：
    复制输出的内容，打开 GitHub -> Settings -> SSH and GPG keys -> New SSH Key，粘贴进去。

---

### 第二步：克隆项目并进入目录

```bash
cd /usr/servise  # 进入你存放项目的父目录
git clone git@github.com:SuniaW/web-ui.git
cd web-ui
```

---

### 第三步：创建三个核心配置文件

在 `web-ui` 根目录下，确保存在以下三个文件：

#### 1. `Dockerfile` (多阶段构建)
```dockerfile
# 阶段 1: 构建
FROM node:20-alpine AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm config set registry https://registry.npmmirror.com && npm install
COPY . .
RUN npm run build

# 阶段 2: 运行
FROM nginx:stable-alpine AS production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 2. `nginx.conf` (Nginx 转发)
```nginx
server {
    listen       80;
    server_name  localhost;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```

#### 3. `docker-compose.yml` (新增：Compose 配置)
这个文件定义了如何启动你的服务。
```yaml
version: '3.8'
services:
  web-ui:
    build: 
      context: .
      dockerfile: Dockerfile
    image: web-ui:latest      # 生成的镜像名称
    container_name: web-ui-app # 容器名称
    restart: always           # 宿主机重启后自动启动容器
    ports:
      - "80:80"               # 宿主机端口:容器端口
    logging:                  # 限制日志大小，防止撑爆磁盘
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

### 第四步：启动与部署

在项目根目录下，只需要执行这一条命令：

```bash
# -d 表示后台运行，--build 表示强制重新构建镜像
docker-compose up -d --build
```

**常用管理命令：**
*   **查看状态**：`docker-compose ps`
*   **查看日志**：`docker-compose logs -f web-ui`
*   **停止服务**：`docker-compose stop`
*   **停止并删除容器/网络**：`docker-compose down`

---

### 第五步：以后代码更新了怎么办？ (专业流程)

当你本地修改了代码并 `git push` 后，服务器上的更新流程如下：

1.  **拉取代码并自动重建**：
    ```bash
    # 一行命令搞定所有：拉取 -> 重新构建镜像 -> 重启受影响的容器
    git pull && docker-compose up -d --build
    ```

2.  **清理垃圾 (可选但推荐)**：
    Docker 每次构建新镜像后，旧镜像会变成 `<none>` (悬空镜像)。为了节省阿里云空间，可以定期清理：
    ```bash
    docker image prune -f
    ```

---

### 💡 为什么这是“专业做法”？

1.  **基础设施即代码**：你的端口映射、重启策略全部记录在 `docker-compose.yml` 中，换一台服务器也能秒级部署。
2.  **零停机时间更新**：Docker Compose 在启动新容器成功后才会停止旧容器，最大限度减少服务中断。
3.  **隔离性**：Compose 会为你的项目创建一个独立的虚拟网络，安全且整洁。
4.  **易于扩展**：如果未来你需要增加后端（如 Python/Node API）或数据库（Redis/MySQL），只需在同一个 `docker-compose.yml` 里增加几行配置即可实现全栈联调。

**确认部署结果：**
执行完 `docker-compose up -d --build` 后，请再次使用手机 5G 或关闭本地代理后访问 `http://8.140.221.150`。