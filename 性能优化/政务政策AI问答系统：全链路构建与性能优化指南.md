
# 🚀 政务政策AI问答系统：全链路构建与性能优化指南
**技术栈：Spring Boot + Spring AI + Milvus + Ollama (Qwen) + Vue3**

---

## 一、 系统架构：RAG（检索增强生成）逻辑
该系统将政务文档转化为大模型的“外部知识库”，解决大模型“一本正经胡说八道”的问题。
1.  **数据清洗**：Spring Boot 解析 PDF/Word 文档。
2.  **向量化 (Embedding)**：利用 `bge-small-zh` 模型将文本转为向量坐标。
3.  **向量检索 (Milvus)**：在海量政策中毫秒级匹配最相关的片段。
4.  **推理生成 (Ollama)**：将“问题+片段”交给 **Qwen (通义千问)** 生成准确回答。
5.  **前端展示 (Vue3)**：通过 **SSE (流式输出)** 实现逐字跳动的回复效果。

---

## 二、 阿里云服务器选型建议

| 场景 | 推荐规格 | 预期表现 |
| :--- | :--- | :--- |
| **正式投产/演示** | **GPU实例 (gn7i)**<br>NVIDIA A10 (24G显存) | **秒开**。7B模型首字延迟 < 500ms，生成速度极快。 |
| **中端应用** | **GPU实例 (gn6i)**<br>NVIDIA T4 (16G显存) | **流畅**。7B模型响应约 1-2s，性价比最高。 |
| **个人实验 (低成本)** | **普通ECS (g7/c7)**<br>4核8G 或 2核4G | **勉强**。需换成 **1.5B小模型**，首字延迟 2-3s。 |

---

## 三、 核心环境搭建 (Linux)

### 1. NVIDIA 驱动与显卡修复 (针对 GPU 实例)
云服务器常缺少基础音频/视频组件导致驱动工具报错：
```bash
# 修复 aplay command not found 报错
apt update && apt install alsa-utils -y

# 确认显卡硬件存在
lspci | grep -i nvidia

# 安装推荐驱动（以 550 服务器版为例）
apt install nvidia-driver-550-server nvidia-utils-550-server -y
reboot  # 安装后必须重启
```
*验证：重启后执行 `nvidia-smi` 看到显卡参数表即成功。*

### 2. Ollama 安装与远程配置
一键安装：
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
**开启远程访问（允许 Spring Boot 调用）：**
执行 `sudo systemctl edit ollama.service`，在打开的编辑器中（若是 **Nano** 则按 `Ctrl+O` 保存 `Ctrl+X` 退出；若是 **Vim** 则输入 `:wq` 回车）：
```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```
**生效配置：**
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 3. Milvus 向量库部署 (Docker)
```bash
mkdir milvus && cd milvus
wget https://github.com/milvus-io/milvus/releases/download/v2.4.0/milvus-standalone-docker-compose.yml -O docker-compose.yml
sudo docker compose up -d
```

---

## 四、 个人实验环境优化 (无 GPU / 纯 CPU 场景)
在普通 CPU 服务器上，必须进行“极限调优”才能避免系统卡死：

### 1. 模型降级：以小博大 (最核心)
**严禁跑 7B 模型！** 改用 CPU 运行的“神级”小模型：
*   **推荐模型**：`qwen2.5:1.5b`（理解力极佳，CPU 速度飞快）。
*   **推理强化版**：`deepseek-r1:1.5b`（适合复杂逻辑分析）。
*   **下载命令**：`ollama run qwen2.5:1.5b`

### 2. 开启 Swap 虚拟内存 (防止崩溃)
防止 Milvus 和 Java 服务抢占内存导致系统杀掉进程：
```bash
sudo fallocate -l 8G /swapfile  # 创建 8G 虚拟内存
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 3. Milvus 内存瘦身
在 `docker-compose.yml` 的 `standalone` 服务下添加内存限制：
```yaml
deploy:
  resources:
    limits:
      memory: 2048M  # 限制 Milvus 占用 2G 内存
```

---

## 五、 后端 Spring AI 关键优化点

1.  **首字延迟优化 (SSE)**：
    必须使用流式输出（`stream().content()`），让用户在 500ms 内看到响应，提升体感速度。
2.  **异步解析文档**：
    政务 PDF 文档通常很大，使用 `@Async` 异步解析并存入向量库，避免阻塞 Web 线程导致前端超时。
3.  **JVM 参数建议**：
    由于 RAG 涉及大量字符串处理和向量运算，建议使用 G1 垃圾回收器：
    ```bash
    java -Xms2g -Xmx4g -XX:+UseG1GC -jar app.jar
    ```

---

## 六、 实验配置总结表

| 维度 | GPU 生产方案 | CPU 实验方案 |
| :--- | :--- | :--- |
| **Ollama 模型** | **Qwen2.5-7B** | **Qwen2.5-1.5B** |
| **响应速度** | < 1s (极速) | 1.5s - 3s (流畅) |
| **内存管理** | 物理内存充足 | **必须开启 Swap 虚拟内存** |
| **向量检索** | Milvus HNSW 索引 | Milvus 默认索引 + 内存限制 |
| **核心挑战** | 驱动安装与显卡直通 | CPU 负载与 OOM 崩溃预防 |

---

**结语**：
构建政务 AI 问答系统的核心在于**硬件环境的适配**。如果是演示和正式使用，请务必选用 **GPU 实例**；如果是个人学习，请坚持使用 **1.5B 级别小模型** 并配置好 **Swap 虚拟内存**，依然可以获得极佳的实验体验。