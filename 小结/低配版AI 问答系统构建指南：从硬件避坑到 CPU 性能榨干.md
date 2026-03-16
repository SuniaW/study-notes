
# 低配版AI 问答系统构建指南：从硬件避坑到 CPU 性能榨干

在资源有限（普通阿里云 ECS、无 GPU）的情况下，构建一个基于 **Spring Boot + Spring AI + Milvus + Ollama** 的问答系统，关键在于“**合理的预期管理**”与“**极限的软件调优**”。

## 一、 核心硬件与业务逻辑
在服务器上运行 Java AI 服务，各硬件分工如下：
*   **CPU**：负责逻辑处理（Spring 业务、PDF 解析）和 AI 推理（Ollama 的数学计算）。核心数决定并发量。
*   **内存 (RAM)**：**系统的命门**。Java (JVM)、Milvus 索引、Ollama 模型都需要驻留内存。内存不足会导致系统直接崩溃 (OOM)。
*   **GPU**：AI 的加速器。没有它，AI 也能跑，但速度会变慢 10 倍以上。
*   **硬盘 (SSD)**：决定了向量数据库检索速度和模型加载速度。

---

## 二、 环境搭建与避坑指南

### 1. Ollama 安装与显卡排查
在 Linux 上一键安装：
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
**常见报错处理：**
*   **`nvidia-smi` 未找到**：若非 GPU 实例，此报错可忽略。
*   **`aplay command not found`**：阿里云精简镜像缺少组件。执行 `apt update && apt install alsa-utils -y` 修复。
*   **远程访问配置**：若需跨机器调用 API，需执行 `sudo systemctl edit ollama.service` 并在 `[Service]` 下添加 `Environment="OLLAMA_HOST=0.0.0.0"`。

### 2. Milvus 向量数据库 (Docker 部署)
创建 `milvus` 文件夹并下载官方 `docker-compose.yml` 启动：
```bash
sudo docker compose up -d
```
*建议：* 使用 **Attu** 作为可视化界面，方便调试政策文档的向量存取。

---

## 三、 个人实验环境的“极限调优”（无 GPU 必看）

如果你使用的是普通的双核或四核 ECS，必须通过以下手段保证 2s 左右的响应：

### 1. 模型降级：以小博大 (最有效)
**严禁在 CPU 环境下强跑 Qwen-7B**。为了流畅度，请使用以下模型：
*   **Qwen2.5-1.5B**：个人实验的“神级”选择，理解力尚可，CPU 运行极快（15+ tokens/s）。
*   **DeepSeek-R1-Distill-Qwen-1.5B**：适合需要逻辑推理的政策问答。
*   **命令**：`ollama run qwen2.5:1.5b`

### 2. 内存救命药：开启 Swap
云服务器内存通常较小，务必手动分配虚拟内存防止服务被系统杀掉：
```bash
sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
```

### 3. Milvus 瘦身
在 `docker-compose.yml` 中，限制 Milvus 容器的内存使用（例如限制在 2G 内存内），为 Spring Boot 和 Ollama 留出空间。

### 4. JVM 参数调优
Java 进程建议使用 G1 垃圾回收器，限制堆内存大小：
```bash
java -Xms1g -Xmx2g -XX:+UseG1GC -jar government-qa-system.jar
```

---

## 四、 RAG 问答系统运行全流程

一个完整的“政务政策问答”请求在服务器上是这样流转的：

1.  **解析阶段**：Spring Boot 通过 `Apache Tika` 解析上传的 PDF/Word 政策文件。
2.  **向量化**：调用 `bge-small-zh` 模型将文字转为向量（CPU 对此微型模型压力不大）。
3.  **检索阶段**：在 **Milvus** 中进行相似度搜索，提取最相关的政策原文。
4.  **Prompt 组装**：将“用户问题”+“政策片段”组装成一段“限定范围”的提示词。
5.  **推理与流式响应**：
    *   Ollama 使用 **1.5B 模型** 在 CPU 上生成答案。
    *   后端通过 **SSE (Server-Sent Events)** 协议将字符流实时推送到 Vue3 前端。
    *   用户在 500ms 内看到首字，2s 内看到完整逻辑。

---

## 五、 实验配置总结表

| 维度 | 推荐配置 (个人实验版) | 预期表现 |
| :--- | :--- | :--- |
| **ECS 规格** | 2 核 4G 或 4 核 8G | 勉强维持多组件运行 |
| **模型选择** | **Qwen2.5-1.5B** | **响应延迟 < 2s** |
| **关键技术** | **开启 8G Swap** | 系统不卡死、不重启 |
| **向量库** | Milvus Standalone | 检索延迟 < 50ms |
| **交互方式** | 流式输出 (SSE) | 用户体感极佳 |

**结语**：
在没有 GPU 的个人实验场景下，不要追求模型的“参数大”，而要追求系统的“链路通”。使用 **Qwen2.5-1.5B** 配合 **Swap 虚拟内存**，你完全可以在一台普通的阿里云服务器上构建出一个极其流畅的政务 AI 助手。