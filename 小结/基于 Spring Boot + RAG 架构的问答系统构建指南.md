# 基于 Spring Boot + RAG 架构的问答系统构建指南

## 一、 系统架构与运行逻辑
本项目采用典型的 **RAG（检索增强生成）** 架构，将政务文档（PDF/Word）转化为可检索的知识库，结合大模型进行精准回答。

1.  **数据层 (ETL)**：Spring Boot 读取文档，通过 `bge-small-zh` 模型将文本向量化，存入 **Milvus** 向量数据库。
2.  **检索层 (Retrieval)**：用户提问时，系统将问题向量化，并在 Milvus 中检索最相关的政策片段。
3.  **生成层 (Generation)**：将检索到的片段作为上下文（Context）交给 **Ollama** 运行的 **Qwen-7B** 模型。
4.  **展示层 (Presentation)**：**Vue3** 前端通过 **SSE (Server-Sent Events)** 实现流式文字输出，并展示引用来源。

---

## 二、 阿里云服务器选型建议
要实现 **2s 以内的响应延迟**，硬件性能是决定性因素。

| 组件 | 推荐配置 | 理由 |
| :--- | :--- | :--- |
| **实例规格** | **阿里云 GPU 实例 (gn7i 或 gn6i)** | 必须配备 NVIDIA GPU，否则 7B 模型推理将极其缓慢（>10s） |
| **GPU** | **NVIDIA A10 (24GB)** 或 **T4 (16GB)** | Qwen-7B 约占 5-8GB 显存，A10 可确保首字极速响应 |
| **CPU/内存** | **8 核 32GB** | Milvus 检索和 Spring Boot 文档解析非常消耗内存 |
| **系统盘** | **100GB ESSD** | 确保模型加载和日志写入的高吞吐 |

> **避坑提醒**：普通 ECS 实例（无 GPU）只能以 CPU 模式运行 Ollama，体验极差，仅适合测试 1.5B 以下的超轻量模型。

---

## 三、 核心环境安装步聚

### 1. NVIDIA 驱动与显卡环境
在云服务器上，首先确认硬件是否存在：
```bash
lspci | grep -i nvidia
```
**修复 `aplay` 报错并安装驱动**：
由于云服务器镜像精简，需先安装基础包：
```bash
apt update && apt install alsa-utils -y
# 安装推荐驱动（以 550 版本为例）
apt install nvidia-driver-550-server nvidia-utils-550-server -y
reboot # 必须重启生效
```
验证驱动：运行 `nvidia-smi`，看到显卡信息表即为成功。

### 2. Ollama 安装与配置
一键安装脚本：
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
**进阶配置：允许 Spring Boot 远程调用**
```bash
sudo systemctl edit ollama.service
# 添加以下内容：
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
# 重启服务
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

### 3. Milvus 向量数据库 (Docker 部署)
创建 `milvus` 目录并下载官方 `docker-compose.yml`：
```bash
mkdir milvus && cd milvus
wget https://github.com/milvus-io/milvus/releases/download/v2.4.0/milvus-standalone-docker-compose.yml -O docker-compose.yml
docker compose up -d
```
*建议同时部署可视化工具 **Attu**，方便查看向量数据。*

---

## 四、 后端实现要点 (Spring AI)

### 1. 模型选型
*   **大模型**：使用 Ollama 运行 `qwen2.5:7b-instruct-q4_K_M`（量化版平衡了速度与精度）。
*   **向量模型**：使用 `bge-small-zh`，其轻量化特征非常适合快速检索。

### 2. 关键代码逻辑
*   **流式响应**：在 Controller 中使用 `Flux<ChatResponse>`。
```java
@GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ChatResponse> chat(@RequestParam String message) {
    return chatClient.prompt().user(message).stream().content();
}
```
*   **RAG 溯源**：从 Milvus 的 `SearchResponse` 中提取 `metadata`（如文件名、页码），并拼接到返回结果中。

---

## 五、 性能优化：如何达到 2s 响应？

1.  **模型量化**：在 Ollama 中务必使用 `q4` 或 `q5` 量化版本，减少显存占用并提升生成速度。
2.  **Milvus 索引优化**：为向量字段创建 `HNSW` 索引，`M` 值设为 16，`efConstruction` 设为 128，可实现毫秒级检索。
3.  **JVM 调优**：针对 Java 服务，设置 `-Xmx16g`（根据内存实际情况），避免在高并发解析 PDF 时触发频繁 GC。
4.  **前端 SSE 渲染**：Vue3 使用 `EventSource` 接收数据，避免等待模型生成完整段落，提升用户“首字”体感速度。

---

## 六、 常见问题排查
*   **Ollama 警告 "CPU-only mode"**：通常是驱动未识别。请检查 `nvidia-smi` 是否正常，并重启 ollama 服务。
*   **Milvus 连接超时**：检查 Docker 容器端口 `19530` 是否被云服务器安全组拦截。
*   **内存不足 (OOM)**：Milvus 的数据是预加载到内存的，如果内存不足 16GB，建议限制 Milvus 的内存配额。

---

通过这套方案，您可以利用阿里云的强大算力，配合 Spring AI 生态，构建出一个响应迅速、回复准确的专业政务政策问答平台。