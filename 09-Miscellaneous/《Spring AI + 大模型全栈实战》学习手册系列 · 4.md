# 专题四：《Ollama 模型管理与调优：让 AI 模型在低配服务器上流畅运行》

## 📚 企业级 RAG 智能问答系统全栈实施指南

### 第四部分：大模型本地部署与优化

---

## 第 1 章 Ollama 概述与核心优势

### 1.1 什么是 Ollama？

Ollama 是一个开源的本地大语言模型运行平台，专为简化 LLM 部署而设计。它让开发者能够在本地服务器上轻松运行、管理和优化大语言模型。

```
┌─────────────────────────────────────────────────────────────┐
│                      Ollama 架构                            │
├─────────────────────────────────────────────────────────────┤
│  用户请求 → API 接口 → 模型加载 → 推理引擎 → 流式响应        │
│                      (llama.cpp 后端)                        │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 为什么选择 Ollama？

| 对比项 | Ollama | 原生 llama.cpp | HuggingFace | vLLM |
|--------|--------|---------------|-------------|------|
| 部署难度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| 模型管理 | 一键拉取 | 手动下载 | 手动下载 | 手动配置 |
| 内存优化 | 自动量化 | 需手动配置 | 需手动配置 | 需手动配置 |
| API 接口 | RESTful | 需自行封装 | 需自行封装 | RESTful |
| 多模型支持 | 100+ | 依赖 GGUF | 依赖 PyTorch | 依赖 PyTorch |
| 社区生态 | 活跃 | 活跃 | 最活跃 | 活跃 |

### 1.3 Ollama 核心特性

| 特性 | 说明 | 技术实现 | 性能提升 |
|------|------|----------|----------|
| 🚀 一键部署 | `ollama pull <model>` | 自动下载量化模型 | 部署时间从 30 分钟→30 秒 |
| 📦 模型量化 | Q4/Q5/Q8 多种精度 | GGUF 格式 | 内存占用降低 50-75% |
| 🔌 REST API | 标准 HTTP 接口 | `/api/generate` | 集成效率提升 80% |
| 🔄 流式输出 | SSE 协议 | `stream: true` | 首字延迟<500ms |
| 💾 模型缓存 | 自动管理 | 本地存储 | 二次加载速度提升 10 倍 |
| 🛡 安全隔离 | 容器化运行 | Docker 支持 | 生产环境就绪 |

---

## 第 2 章 Ollama 安装与配置实战

### 2.1 Linux 一键安装脚本

```bash
#!/bin/bash
# install-ollama.sh

echo "🚀 开始安装 Ollama..."

# 1. 下载并执行官方安装脚本
curl -fsSL https://ollama.com/install.sh | sh

# 2. 验证安装
ollama --version

# 3. 创建系统服务
sudo systemctl enable ollama
sudo systemctl start ollama

# 4. 检查服务状态
sudo systemctl status ollama

# 5. 配置开机自启
sudo systemctl enable ollama

echo "✅ Ollama 安装完成！"
echo "   - 服务端口：11434"
echo "   - 模型目录：~/.ollama/models"
echo "   - 日志位置：journalctl -u ollama"
```

### 2.2 Docker 部署方案（推荐生产环境）

```yaml
# docker-compose.yml
services:
  ollama:
    container_name: ollama
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ./ollama-data:/root/.ollama
      - ./models:/models
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_PORT=11434
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_NUM_THREADS=2
      - OLLAMA_MAX_LOADED_MODELS=1
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2.0'
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - rag-network

networks:
  rag-network:
    driver: bridge
```

### 2.3 关键环境变量配置

| 变量名 | 默认值 | 推荐值 (2C4G) | 说明 | 影响 |
|--------|--------|---------------|------|------|
| `OLLAMA_HOST` | 127.0.0.1 | 0.0.0.0 | 监听地址 | 允许外部访问 |
| `OLLAMA_PORT` | 11434 | 11434 | 服务端口 | 防火墙需放行 |
| `OLLAMA_KEEP_ALIVE` | 5m | 5m | 模型保持时间 | 超时自动释放内存 |
| `OLLAMA_NUM_THREADS` | CPU 核心数 | 2 | 推理线程数 | 防止 CPU 满载 |
| `OLLAMA_MAX_LOADED_MODELS` | 1 | 1 | 最大加载模型数 | 防止内存溢出 |
| `OLLAMA_MODELS` | ~/.ollama/models | /models | 模型存储路径 | 便于数据持久化 |
| `OLLAMA_DEBUG` | false | false | 调试模式 | 生产环境关闭 |

### 2.4 30 秒验证安装成功

```bash
# 1. 检查服务状态
curl http://localhost:11434/api/tags

# 2. 拉取测试模型
ollama pull qwen2.5:0.5b

# 3. 测试推理
ollama run qwen2.5:0.5b "你好，请自我介绍"

# 4. 测试 API 接口
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:0.5b",
  "prompt": "什么是 RAG 技术？",
  "stream": false
}' | jq '.response'

# 5. 查看已安装模型
ollama list

# 6. 查看模型详细信息
ollama show qwen2.5:0.5b
```

---

## 第 3 章 模型量化与选型指南

### 3.1 量化技术详解

量化是将模型权重从高精度（FP32）转换为低精度（INT4/INT8）的技术，可大幅降低内存占用。

#### 量化等级对比

| 量化类型 | 精度 | 内存压缩比 | 精度损失 | 适用场景 | 文件大小示例 (7B 模型) |
|----------|------|------------|----------|----------|------------------------|
| **FP32** | 32 位浮点 | 1x | 0% | 训练/研究 | 28GB |
| **FP16** | 16 位浮点 | 2x | <1% | 高精度推理 | 14GB |
| **Q8_0** | 8 位整数 | 4x | 1-2% | 生产环境推荐 | 7GB |
| **Q5_K_M** | 5 位混合 | 5.5x | 2-3% | 平衡方案 | 5GB |
| **Q4_K_M** | 4 位混合 | 7x | 3-5% | 低配服务器推荐 | 4GB |
| **Q3_K_S** | 3 位混合 | 9x | 5-8% | 极端低配 | 3GB |
| **Q2_K** | 2 位混合 | 12x | 8-12% | 不推荐 | 2.5GB |

### 3.2 Ollama 模型命名规则

```
<模型名>:<参数量><量化版本>

示例：
- qwen2.5:0.5b        → Qwen2.5 0.5B 参数，默认量化
- qwen2.5:1.5b-q4_K_M → Qwen2.5 1.5B 参数，Q4 量化
- qwen2.5:7b-q8_0     → Qwen2.5 7B 参数，Q8 量化
- bge-m3:latest       → BGE-M3 Embedding 模型，最新版
```

### 3.3 2025 年推荐模型清单

| 场景 | 模型 | 参数量 | 量化版本 | 内存占用 | 推理速度 | 推荐指数 |
|------|------|--------|----------|----------|----------|----------|
| **2C4G 服务器** | qwen2.5:0.5b | 0.5B | Q4_K_M | 0.8GB | 50 token/s | ⭐⭐⭐⭐⭐ |
| **4C8G 服务器** | qwen2.5:1.5b | 1.5B | Q4_K_M | 1.6GB | 35 token/s | ⭐⭐⭐⭐⭐ |
| **8C16G 服务器** | qwen2.5:7b | 7B | Q4_K_M | 4.5GB | 20 token/s | ⭐⭐⭐⭐ |
| **GPU 环境** | qwen2.5:7b | 7B | FP16 | 14GB | 80 token/s | ⭐⭐⭐⭐⭐ |
| **Embedding** | bge-m3 | 567M | Q4_K_M | 1.2GB | 100 句/s | ⭐⭐⭐⭐⭐ |
| **代码生成** | codellama:7b | 7B | Q4_K_M | 4.5GB | 18 token/s | ⭐⭐⭐⭐ |
| **多语言** | llama3.1:8b | 8B | Q4_K_M | 5.0GB | 18 token/s | ⭐⭐⭐⭐ |

### 3.4 模型拉取与管理命令

```bash
# 1. 拉取模型
ollama pull qwen2.5:1.5b

# 2. 拉取特定量化版本
ollama pull qwen2.5:1.5b-q4_K_M

# 3. 查看已安装模型
ollama list

# 4. 查看模型详细信息
ollama show qwen2.5:1.5b

# 5. 查看模型文件位置
ls -lh ~/.ollama/models

# 6. 删除模型
ollama rm qwen2.5:1.5b

# 7. 复制模型（创建自定义版本）
ollama cp qwen2.5:1.5b my-qwen

# 8. 导出模型
ollama cp qwen2.5:1.5b /backup/my-model

# 9. 清理未使用模型
ollama prune
```

---

## 第 4 章 2G 内存极限优化实战

### 4.1 内存分配策略（2C4G 服务器）

> ⚠️ **警告**：2G 内存是 RAG 系统的"生存线"，必须执行精细化资源管理

#### 内存资产负债表

| 组件 | 物理内存 | 优化后 | 优化手段 | 风险等级 |
|------|----------|--------|----------|----------|
| **Ollama (LLM)** | 2.5GB | 1.0GB | Q4 量化 + 线程限制 | 🔴 高 |
| **Ollama (Embed)** | 1.2GB | 0.4GB | 换用 bge-small-zh | 🔴 高 |
| **Milvus** | 2.8GB | 1.0GB | 内存限制配置 | 🔴 高 |
| **Spring Boot** | 0.8GB | 0.4GB | JVM 参数限制 | 🟡 中 |
| **系统预留** | 0.7GB | 0.2GB | 关闭冗余服务 | 🟡 中 |
| **总计** | **8.0GB** | **3.0GB** | **需 Swap 补充** | - |

### 4.2 Swap 虚拟内存配置（必做）

```bash
#!/bin/bash
# setup-swap.sh

echo "🚀 配置 Swap 虚拟内存..."

# 1. 检查现有 Swap
free -h

# 2. 创建 4G Swap 文件（2G 内存服务器建议 4-8G Swap）
sudo fallocate -l 4G /swapfile

# 3. 设置权限（安全要求）
sudo chmod 600 /swapfile

# 4. 格式化为 Swap
sudo mkswap /swapfile

# 5. 启用 Swap
sudo swapon /swapfile

# 6. 验证 Swap
free -h

# 7. 永久生效（写入 fstab）
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 8. 调整 Swap 使用倾向（默认 60，改为 30 减少 Swap 使用）
sudo sysctl vm.swappiness=30
echo 'vm.swappiness=30' | sudo tee -a /etc/sysctl.conf

# 9. 调整脏页回写策略
sudo sysctl vm.dirty_ratio=10
sudo sysctl vm.dirty_background_ratio=5

echo "✅ Swap 配置完成！"
echo "   - Swap 大小：4GB"
echo "   - Swappiness: 30（减少 Swap 使用频率）"
```

### 4.3 Ollama 启动参数优化

```bash
# 2C4G 服务器推荐启动命令
ollama serve \
  --host 0.0.0.0 \
  --port 11434 \
  --num-gpu 0 \
  --num-threads 2 \
  --max-loaded-models 1

# 环境变量方式（推荐 Docker 使用）
export OLLAMA_HOST=0.0.0.0
export OLLAMA_PORT=11434
export OLLAMA_NUM_THREADS=2
export OLLAMA_MAX_LOADED_MODELS=1
export OLLAMA_KEEP_ALIVE=5m
export OLLAMA_DEBUG=false

ollama serve
```

### 4.4 模型加载优化技巧

```bash
# 1. 预加载模型到内存（减少首次请求延迟）
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:0.5b",
  "prompt": "warmup",
  "stream": false
}'

# 2. 设置模型保持时间（5 分钟后自动释放）
export OLLAMA_KEEP_ALIVE=5m

# 3. 限制并发请求数（防止内存溢出）
# 在 Modelfile 中设置
PARAMETER num_thread 2
PARAMETER max_queue 1

# 4. 监控模型内存占用
watch -n 1 'ollama ps'

# 5. 手动卸载模型
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:0.5b",
  "prompt": "",
  "keep_alive": 0
}'
```

### 4.5 系统级优化配置

```bash
#!/bin/bash
# system-optimize.sh

echo "🚀 系统级优化配置..."

# 1. 关闭不必要的系统服务
sudo systemctl stop bluetooth
sudo systemctl disable bluetooth
sudo systemctl stop cups
sudo systemctl disable cups

# 2. 清理系统缓存
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches

# 3. 限制 Docker 内存使用
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "max-concurrent-downloads": 3,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

# 4. 重启 Docker 服务
sudo systemctl restart docker

# 5. 清理 Docker 冗余数据
docker system prune -af

# 6. 安装监控工具
sudo apt update
sudo apt install -y htop iotop

# 7. 设置文件描述符限制
echo '* soft nofile 65536' | sudo tee -a /etc/security/limits.conf
echo '* hard nofile 65536' | sudo tee -a /etc/security/limits.conf

echo "✅ 系统优化完成！"
```

---

## 第 5 章 安全组与网络配置

### 5.1 云服务器安全组配置（阿里云/腾讯云）

| 端口 | 协议 | 方向 | 授权对象 | 说明 | 优先级 |
|------|------|------|----------|------|--------|
| **11434** | TCP | 入站 | 应用服务器 IP | Ollama API 端口 | 🔴 高 |
| **19530** | TCP | 入站 | 应用服务器 IP | Milvus 服务端口 | 🔴 高 |
| **9091** | TCP | 入站 | 应用服务器 IP | Milvus 管理端口 | 🟡 中 |
| **8080** | TCP | 入站 | 0.0.0.0/0 | Spring Boot 应用 | 🔴 高 |
| **22** | TCP | 入站 | 管理 IP | SSH 远程登录 | 🔴 高 |
| **3000** | TCP | 入站 | 管理 IP | Attu Web 管理 | 🟢 低 |

### 5.2 Linux 防火墙配置（UFW）

```bash
#!/bin/bash
# firewall-setup.sh

echo "🚀 配置防火墙..."

# 1. 安装 UFW（如果未安装）
sudo apt update
sudo apt install -y ufw

# 2. 重置防火墙规则
sudo ufw reset

# 3. 设置默认策略
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 4. 允许 SSH（防止被锁在外面）
sudo ufw allow 22/tcp

# 5. 允许 Ollama 端口（仅内网）
sudo ufw allow from 192.168.1.0/24 to any port 11434 proto tcp

# 6. 允许 Milvus 端口（仅内网）
sudo ufw allow from 192.168.1.0/24 to any port 19530 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 9091 proto tcp

# 7. 允许应用端口（公网）
sudo ufw allow 8080/tcp

# 8. 启用防火墙
sudo ufw enable

# 9. 查看状态
sudo ufw status verbose

# 10. 查看日志
sudo ufw logging on
sudo tail -f /var/log/ufw.log

echo "✅ 防火墙配置完成！"
```

### 5.3 Nginx 反向代理配置（生产环境推荐）

```nginx
# /etc/nginx/sites-available/ollama
server {
    listen 80;
    server_name api.yourdomain.com;

    # 限制请求频率（防止 DDoS）
    limit_req_zone $binary_remote_addr zone=ollama:10m rate=10r/s;

    location / {
        # 应用限流
        limit_req zone=ollama burst=20 nodelay;

        # 反向代理到 Ollama
        proxy_pass http://127.0.0.1:11434;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # SSE 流式响应支持
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }

    # 健康检查接口
    location /health {
        access_log off;
        return 200 "OK";
        add_header Content-Type text/plain;
    }
}
```

### 5.4 JWT 认证中间件（Spring Security）

```java
// OllamaAuthFilter.java
@Component
public class OllamaAuthFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) {
        
        // 1. 获取 Token
        String token = extractToken(request);
        
        // 2. 验证 Token
        if (token != null && jwtTokenProvider.validateToken(token)) {
            // 3. 设置用户上下文
            UsernamePasswordAuthenticationToken auth = 
                jwtTokenProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
            
            // 4. 继续过滤链
            try {
                filterChain.doFilter(request, response);
            } catch (Exception e) {
                log.error("过滤链执行失败", e);
            }
        } else {
            // 5. 返回 401
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        }
    }
    
    private String extractToken(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");
        if (bearer != null && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }
        return null;
    }
}
```

---

## 第 6 章 常见问题排查清单

### 6.1 启动类问题

| 问题 | 症状 | 原因 | 解决方案 | 优先级 |
|------|------|------|----------|--------|
| 服务无法启动 | 端口占用 | 11434 已被占用 | `lsof -i:11434` 查找并 kill | 🔴 高 |
| 模型加载失败 | OOM Killed | 内存不足 | 增加 Swap 或换小模型 | 🔴 高 |
| 连接被拒绝 | Connection refused | 防火墙阻挡 | 检查安全组/UFW 配置 | 🔴 高 |
| 权限错误 | Permission denied | 文件权限问题 | `chown -R $USER ~/.ollama` | 🟡 中 |
| Docker 启动失败 | 容器反复重启 | 资源限制过低 | 调整 docker-compose 内存限制 | 🔴 高 |

### 6.2 性能类问题

| 问题 | 症状 | 原因 | 解决方案 | 优先级 |
|------|------|------|----------|--------|
| 首字延迟>5s | 响应慢 | 模型未预加载 | 启动后先 warmup | 🟡 中 |
| 推理速度<10 token/s | 生成慢 | CPU 线程过多 | 限制 OLLAMA_NUM_THREADS=2 | 🟡 中 |
| 内存持续增长 | 内存泄漏 | 模型未释放 | 设置 OLLAMA_KEEP_ALIVE=5m | 🔴 高 |
| CPU 持续 100% | 系统卡顿 | 并发请求过多 | 限制 max_queue=1 | 🔴 高 |
| 磁盘 IO 高 | 系统卡顿 | Swap 频繁使用 | 增加物理内存或减少模型 | 🟡 中 |

### 6.3 排查命令速查

```bash
# 1. 检查 Ollama 服务状态
systemctl status ollama

# 2. 查看 Ollama 日志
journalctl -u ollama -f

# 3. 检查端口占用
lsof -i:11434
netstat -tlnp | grep 11434

# 4. 查看模型列表
ollama list

# 5. 查看运行中的模型
ollama ps

# 6. 监控系统资源
htop
free -h
df -h

# 7. 查看 Docker 容器状态
docker ps
docker stats ollama

# 8. 测试 API 连接
curl http://localhost:11434/api/tags

# 9. 查看模型详细信息
ollama show <model-name>

# 10. 清理未使用模型
ollama prune
```

---

## 第 7 章 生产环境部署最佳实践

### 7.1 Docker Compose 生产模板

```yaml
# docker-compose.prod.yml
services:
  ollama:
    container_name: ollama
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ./ollama-data:/root/.ollama
      - ./models:/models
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_NUM_THREADS=2
      - OLLAMA_MAX_LOADED_MODELS=1
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2.0'
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - rag-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    container_name: nginx-proxy
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - ollama
    restart: unless-stopped
    networks:
      - rag-network

networks:
  rag-network:
    driver: bridge
```

### 7.2 监控告警配置（Prometheus）

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'ollama'
    static_configs:
      - targets: ['ollama:11434']
    metrics_path: '/api/metrics'
    scrape_interval: 15s
    
  - job_name: 'milvus'
    static_configs:
      - targets: ['milvus:9091']
    scrape_interval: 15s
```

### 7.3 备份与恢复策略

```bash
#!/bin/bash
# backup-ollama.sh

echo "🚀 开始备份 Ollama 数据..."

# 1. 创建备份目录
BACKUP_DIR="/backup/ollama/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# 2. 备份模型数据
cp -r ~/.ollama/models $BACKUP_DIR/models

# 3. 备份配置文件
cp ~/.ollama/Modelfile* $BACKUP_DIR/ 2>/dev/null || true

# 4. 压缩备份
tar -czf $BACKUP_DIR.tar.gz -C $BACKUP_DIR .

# 5. 清理临时目录
rm -rf $BACKUP_DIR

# 6. 清理旧备份（保留 7 天）
find /backup/ollama -name "*.tar.gz" -mtime +7 -delete

echo "✅ 备份完成：$BACKUP_DIR.tar.gz"

# 恢复命令
# tar -xzf backup.tar.gz -C ~/.ollama/
```

### 7.4 性能基准测试

```python
# benchmark_ollama.py
import requests
import time
import statistics

def benchmark_model(model_name: str, num_requests: int = 10):
    """性能基准测试"""
    
    latencies = []
    throughputs = []
    
    for i in range(num_requests):
        start = time.time()
        
        response = requests.post(
            "http://localhost:11434/api/generate",
            json={
                "model": model_name,
                "prompt": "请简要介绍 RAG 技术，约 100 字",
                "stream": False
            }
        )
        
        end = time.time()
        latency = end - start
        
        # 计算吞吐量
        token_count = response.json().get("eval_count", 0)
        throughput = token_count / latency
        
        latencies.append(latency)
        throughputs.append(throughput)
        
        print(f"请求 {i+1}: 延迟={latency:.2f}s, 吞吐量={throughput:.1f} token/s")
    
    # 统计结果
    print("\n" + "="*50)
    print(f"模型：{model_name}")
    print(f"平均延迟：{statistics.mean(latencies):.2f}s")
    print(f"P95 延迟：{sorted(latencies)[int(num_requests*0.95)]:.2f}s")
    print(f"平均吞吐量：{statistics.mean(throughputs):.1f} token/s")
    print(f"最小吞吐量：{min(throughputs):.1f} token/s")
    print("="*50)

# 运行测试
benchmark_model("qwen2.5:0.5b", num_requests=10)
```

---

**本专题完**

> 📌 **核心要点总结**：
> 1. Ollama 是本地部署 LLM 的最佳选择，一键拉取模型
> 2. Q4 量化可在精度损失<5% 的情况下降低 70% 内存占用
> 3. 2C4G 服务器必须配置 4G+ Swap 虚拟内存
> 4. OLLAMA_KEEP_ALIVE=5m 防止内存泄漏
> 5. 云服务器安全组仅开放必要端口，内网通信
> 6. Nginx 反向代理 + JWT 认证是生产环境标配
> 7. 定期备份模型数据，防止意外丢失

> 📖 **下专题预告**：《Spring AI 后端开发实战：30 行代码实现 RAG 核心逻辑》—— 详解 Spring AI 集成、向量存储服务、SSE 流式响应、依赖冲突解决