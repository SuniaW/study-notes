

这份文档不仅包含基础配置，还整合了**Node版本兼容性、Git冲突处理、本地代理避坑**以及**一键自动化脚本**等实战干货。

---

# 🚀 Docker Compose 容器管理与自动化部署进阶指南

## 📌 核心理念
使用 Docker Compose 是生产环境的“黄金标准”。它通过 `docker-compose.yml` 将复杂的 `docker run` 参数持久化，实现**基础设施即代码 (IaC)**。

---

## 🛠️ 第一步：环境基座配置

### 1.1 SSH 密钥与 GitHub 联通
确保服务器有权拉取私有仓库：
1. `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"` (一路回车)。
2. `cat ~/.ssh/id_rsa.pub` 复制公钥至 GitHub Settings -> SSH Keys。
3. **测试连接**：`ssh -T git@github.com`。看到 *Hi SuniaW!* 为成功。

### 1.2 目录规范
```bash
mkdir -p /usr/servise/web-ui
cd /usr/servise/web-ui
git clone git@github.com:SuniaW/web-ui.git .
```

---

## 📄 第二步：核心配置文件优化 (含避坑指南)

### 2.1 Dockerfile (必须使用 Node 20+)
**实战经验**：Vite 7+ 构建时需要 Node 20+ 的加密 API，否则报 `crypto.hash is not a function`。
```dockerfile
# 阶段 1: 构建 (必须用 node:20-alpine)
FROM node:20-alpine AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm config set registry https://registry.npmmirror.com && npm install
COPY . .
RUN npm run build

# 阶段 2: 运行 (轻量化 Nginx)
FROM nginx:stable-alpine AS production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 2.2 nginx.conf (SPA 必备)
```nginx
server {
    listen       80;
    server_name  localhost;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html; # 解决 Vue Router 刷新 404
    }
}
```

### 2.3 docker-compose.yml (标准化管理)
```yaml
# 注意：最新版 Docker 已不再推荐使用 version 字段
services:
  web-ui:
    build: .
    image: web-ui:latest
    container_name: web-ui-app
    restart: always
    ports:
      - "80:80"  # 若 80 端口受限可改为 "8080:80"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 🤖 第三步：【新增】自动化部署脚本 (`deploy.sh`)

为了彻底解决 **Git 冲突**、**旧容器残留** 和 **磁盘爆满** 问题，使用此脚本实现一键更新。

### 3.1 创建脚本
`vi deploy.sh` 并粘贴以下内容：

```bash
#!/bin/bash

# 配置区域
PROJECT_PATH="/usr/servise/web-ui"
BRANCH="main"
CONTAINER_NAME="web-ui-app"

# 颜色定义
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

echo -e "${YELLOW}🚀 开始自动化部署流程...${NC}"

# 1. 代码同步
cd $PROJECT_PATH || exit
echo -e "${YELLOW}📥 强制对齐远程代码，解决潜在冲突...${NC}"
git fetch --all
git reset --hard origin/$BRANCH

# 2. 彻底重启服务
echo -e "${YELLOW}🔄 正在停止旧容器并重新构建...${NC}"
docker compose down
docker compose up -d --build --force-recreate

# 3. 验证状态
sleep 3
STATUS=$(docker inspect -f '{{.State.Running}}' $CONTAINER_NAME 2>/dev/null)
if [ "$STATUS" == "true" ]; then
    echo -e "${GREEN}✅ 部署成功！容器正在运行。${NC}"
else
    echo -e "${RED}❌ 部署失败，请检查 docker compose logs。${NC}"
    exit 1
fi

# 4. 磁盘维护
echo -e "${YELLOW}🧹 清理悬空镜像，释放阿里云磁盘空间...${NC}"
docker image prune -f

echo -e "${GREEN}=========================================${NC}"
echo -e "${GREEN}🎉 站点已上线: http://8.140.221.150${NC}"
echo -e "${GREEN}=========================================${NC}"
```

### 3.2 赋权并执行
```bash
chmod +x deploy.sh
./deploy.sh
```

---

## ⚠️ 第四步：实战排障“金律” (必看)

### 4.1 502 Bad Gateway 的“冒牌货”
*   **现象**：浏览器报错 502，但服务器 `curl 127.0.0.1` 正常。
*   **根源**：本地电脑开启了 **Clash/VPN 代理**。
*   **特征**：浏览器 F12 显示 `Remote Address: 127.0.0.1:17890`。
*   **方案**：关闭本地代理或将服务器 IP 加入白名单。

### 4.2 nginx.conf: not found 陷阱
*   **现象**：`COPY nginx.conf` 报错。
*   **根源**：`.dockerignore` 文件里写了 `*.conf`。
*   **方案**：确保 `.dockerignore` 只包含 `node_modules` 和 `dist`，不要过滤配置文件。

---

## 🔄 第五步：日常更新工作流
以后你只需在本地修改代码并 `git push`，然后在服务器执行：
```bash
./deploy.sh
```

### 💡 为什么这种方案最稳健？
1.  **Git Reset**：无视服务器上的意外改动，确保生产环境与仓库一致。
2.  **Compose Down/Up**：彻底重启，消除端口占用和网络残留。
3.  **Prune**：自动清理旧镜像，防止阿里云 20G/40G 的小磁盘被撑爆。
4.  **Logging**：限制日志大小，防止长期运行产生的日志文件过大。

---

## ✅ 最终确认清单
- [ ] **Node 版本**：Dockerfile 镜像是否已升至 `node:20-alpine`？
- [ ] **安全组**：阿里云 80 端口（或 8080）是否已开放？
- [ ] **本地网络**：访问前是否已关闭本地 Clash/VPN？
- [ ] **自动脚本**：`deploy.sh` 是否已赋予执行权限？