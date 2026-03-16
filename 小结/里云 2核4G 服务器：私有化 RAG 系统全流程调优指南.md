
# 🚀 阿里云 2核4G 服务器：私有化 RAG 系统全流程调优指南

**背景：** 2G 内存是 RAG 系统的“生命禁区”，升级至 4G 是从“试验”走向“实战”的明智之举。通过合理的资源切分与限制，我们将系统从“卡死动弹不得”转化为“丝滑稳定响应”。

---

## 一、 深度复盘：为什么 2GB 内存必崩无疑？

在 2GB 内存下，系统处于 **“小马拉大车”** 的超载状态。

### 1. 核心痛点分析
**内存赤字账本：**
- **BGE-M3 (Embed):** ~1.2GB
- **Qwen-1.5B (LLM):** ~1.0GB
- **Milvus (DB):** ~1.0GB
- **Spring Boot:** ~0.5GB
- **总计需求：** 约 **3.7GB**（远超物理内存）

**连锁反应：** 物理内存溢出后，系统疯狂调用 **Swap (虚拟内存)**。由于硬盘读写速度比内存慢数百倍，导致 CPU 长期处于 `iowait` 状态，程序表现为连接超时 `[request_sent]` 或直接卡死。

### 2. 生存级调优方案（若被迫在 2GB 运行）
若必须在极低配置下生存，需采用“断臂求生”策略：

| 组件 | 2G 环境下的表现 | 2G 生存级对策 |
| :--- | :--- | :--- |
| **Ollama (LLM)** | 频繁触发 Swap，响应需数分钟 | 降级至 `qwen2.5:0.5b` (约 397MB) |
| **Ollama (Embed)** | 直接导致 OOM (内存溢出) | 换用 `bge-small-zh` 或 `all-minilm` |
| **Milvus** | 容器因内存不足频繁重启 | 严苛限制 Docker 内存配额 |
| **Spring Boot** | 启动即卡死，无法分配内存 | 极度限制堆内存 `-Xmx256m` |

---

## 二、 实战级方案：4GB 内存的黄金配置

升级到 4G 后，你已跨过生存线。建议采用 **BGE-M3 (高精检索) + Qwen-1.5B (智能对话)** 的黄金组合。

### 1. 内存资产负债表 (合理分配 4GB)
为防止某个进程“暴饮暴食”，必须划定红线：
- **Ollama (模型权重):** 约 **2.2 GB** (加载 BGE-M3 + Qwen-1.5B)
- **Milvus (向量数据库):** 约 **0.8 GB** (通过 Docker 限制)
- **Spring Boot (JVM):** 约 **0.6 GB** (通过启动参数限制)
- **OS 预留 (系统底层):** 约 **0.4 GB**
- **[防线] Swap:** 保持开启 **8GB Swap**，作为应对突发峰值的缓冲。

### 2. 核心组件精细化设置

#### A. Ollama 服务管理 (防内存常驻)
编辑 Ollama 配置，确保模型在不使用时及时释放内存：
```bash
sudo SYSTEMD_EDITOR=vim systemctl edit ollama.service
```
在 `[Service]` 块中添加：
```ini
[Service]
Environment="OLLAMA_KEEP_ALIVE=5m"   # 5分钟不使用自动释放内存
Environment="OLLAMA_NUM_PARALLEL=1"   # 串行处理，防止并发撑爆内存
```
**重启服务：** `sudo systemctl daemon-reload && sudo systemctl restart ollama`

---

## 三、 Milvus 容器限额：资源配额化 (Docker Compose)

针对 2核4G 环境，在 `docker-compose.yml` 中做三件事：**添加内存硬限制、优化启动顺序、针对小内存环境调优。**

```yaml
version: '3.5'

services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/etcd:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 20s
      retries: 3
    # --- 内存限制 ---
    deploy:
      resources:
        limits:
          memory: 256M  # etcd 占用较小，限制在 256M 足够

  minio:
    container_name: milvus-minio
    image: docker.m.daocloud.io/minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    ports:
      - "9001:9001"
      - "9000:9000"
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data
    command: minio server /minio_data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
    # --- 内存限制 ---
    deploy:
      resources:
        limits:
          memory: 512M  # Minio 主要是存储，512M 足够日常使用

  standalone:
    container_name: milvus-standalone
    image: docker.m.daocloud.io/milvusdb/milvus:v2.4.0
    command: ["milvus", "run", "standalone"]
    security_opt:
      - seccomp:unconfined
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
      # --- 针对 4G 内存的性能调优参数 ---
      # 限制 Milvus 内部索引占用，强制让它在达到限制时进行内存回收
      CACHE_SIZE: 512M 
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
      interval: 30s
      start_period: 90s
      timeout: 20s
      retries: 3
    ports:
      - "19530:19530"
      - "9091:9091"
    # --- 强制依赖健康状态 ---
    depends_on:
      etcd:
        condition: service_healthy
      minio:
        condition: service_healthy
    # --- 核心内存限制 ---
    deploy:
      resources:
        limits:
          memory: 1024M  # 严控 Milvus 主程序占用在 1GB 以内

networks:
  default:
    name: milvus
```

---

## 四、 深度拆解：Milvus 优化的六大原理

这份配置的核心逻辑是 **“资源配额化”**，即通过硬性手段防止组件挤占 Ollama 和 Spring Boot 的生存空间。

### 1. 核心组件：资源边界限制 (`deploy.resources.limits`)

这是本次优化最关键的部分。在 Docker 中，如果不加限制，容器会尽可能占用宿主机的全部内存。

* **Etcd (256MB):**
  * **角色：** 存储 Milvus 的元数据（类似目录索引）。
  * **解释：** 元数据通常很小。给它 256MB 是非常宽裕的，主要为了防止极端情况下（如频繁写入）它过度消耗内存。
* **Minio (512MB):**
  * **角色：** 存储实际的向量数据文件和日志。
  * **解释：** 在小规模 RAG 项目中，Minio 的压力并不大。512MB 足以支撑数万条文档的存储与检索。
* **Milvus Standalone (1024MB):**
  * **角色：** 向量数据库的核心引擎，负责向量计算和索引。
  * **解释：** 这是“吃内存”的大户。在 4G 服务器上，我们必须给它画出 **1GB 的“红线”**。一旦超过这个值，Docker 会限制它，从而保护你的 Ollama（大模型）不被系统因内存不足而强制杀死。

---

###  2.依赖逻辑启动 (`depends_on` + `condition`)

原版配置只管“启动顺序”，不管“启动结果”。优化版使用了更严谨的**健康检查依赖**。

* **旧版逻辑：** 启动 Etcd -> 启动 Minio -> 启动 Milvus。
  * *风险：* Etcd 可能还没初始化完，Milvus 就尝试连接，导致 Milvus 启动失败并反复重启，瞬间推高 CPU 占用。
* **优化版逻辑：**
  * Milvus 会等待 Etcd 变成 **`healthy`（健康）** 状态。
  * Milvus 会等待 Minio 变成 **`healthy`（健康）** 状态。
  * **好处：** 避免了启动时的 CPU 瞬间峰值，确保系统组件一个接一个稳步运行，这对于 2核 的弱 CPU 环境极其重要。

---

### 3.Milvus 内部参数微调 (`environment`)

我们不光在外面（Docker）限制它，还在里面（Milvus 参数）告诉它要省着点花。

* **`CACHE_SIZE: 512M`**:
  * **原理：** Milvus 为了速度，默认会占用大量内存做缓存（Cache）来存储索引。
  * **设置意义：** 手动限制内部缓存为 512MB。这样加上程序运行所需的内存，总占用会很稳地控制在 1GB 左右。

---

###  4.数据持久化 (`volumes`)

```yaml
volumes:
  - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus
```

* **解释：** 这里的配置确保你的向量数据库数据存储在宿主机的硬盘上。
* **安全性：** 即使你执行 `docker compose down` 删除了容器，你的政策文档向量数据也不会丢失，下次启动时会自动挂载。

---

### 5.安全与性能配置 (`security_opt`)

* **`seccomp:unconfined`**:
  * **原理：** Milvus 在执行高性能向量计算时，需要直接调用一些底层系统指令。
  * **设置意义：** 取消默认的安全计算模型限制，可以提升向量检索的性能，并减少由于权限拦截导致的诡异报错。

---

### 6.总结：4GB 内存服务器的“生存法则”

通过这份配置，你的 4GB 内存被精确地切分成了几块“自留地”：

1. **操作系统:** 留存 ~0.4GB。
2. **Milvus 全家桶:** 占用 ~1.2GB（1024M 主程序 + 辅助组件）。
3. **Spring Boot:** 建议限制在 0.6GB (`-Xmx512m`)。
4. **Ollama (模型权重):** 剩余约 **1.8GB - 2.0GB**。

**操作建议：**
应用新配置时，请务必执行以下步骤：

1.  **停止并移除旧容器：**
    ```bash
    docker compose down
    ```
2.  **应用新配置并以后台模式启动：**
    ```bash
    docker compose up -d
    ```
3.  **观察实时内存占用 (关键)：**
    ```bash
    docker stats
    ```
  
    **检查点：** 确保 `milvus-standalone` 的 `MEM USAGE` 稳定在 900MB-1000MB 之间。如果你看到 `milvus-standalone` 的 `MEM USAGE` 稳定在 **900MB - 1000MB** 之间，且 `milvus-minio` 在 **200MB** 左右，说明配置生效，你的系统处于**最稳健**的运行状态。


**执行结果如下图**
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2fbfaa04ccfc47e394eacfc82668b92c.png)


**恭喜！这是教科书级别的 `docker stats` 状态。**

你的 Milvus 全家桶已经完美地运行在了预设的“资源牢笼”中。让我们来看看现在的内存账单：

1.  **Milvus Standalone:** 445MiB（由于刚启动没多久，还没吃满 512M 缓存，远低于 1G 限制）。
2.  **Milvus Minio:** 187MiB（处于极佳的空闲状态）。
3.  **Milvus Etcd:** 14MiB（几乎可以忽略不计）。
4.  **总占用:** 约 **647MiB**。

在 4G (4096MiB) 的服务器上，你现在还有 **约 3.4GB** 的自由内存。这为接下来的 **Ollama (2.2GB)** 和 **Spring Boot (0.6GB)** 留出了极其宽裕的冲刺空间。

---

### 🚀 接下来的“实战部署”三步走

既然数据库已经稳如泰山，请按顺序执行以下操作：

#### 第一步：启动并预热模型 (Ollama)
由于你还有充足空间，现在可以放心让 Ollama 加载高精度模型。请手动触发一次加载，防止 Java 程序第一次请求时超时：

1.  **预热 Embedding 模型 (BGE-M3):**
    ```bash
    curl http://localhost:11434/api/embeddings -d '{"model": "bge-m3", "prompt": "warmup"}'
    ```
2.  **预热 Chat 模型 (Qwen2.5-1.5B):**
    ```bash
    curl http://localhost:11434/api/chat -d '{"model": "qwen2.5:1.5b", "messages": [{"role": "user", "content": "hi"}]}'
    ```

#### 第二步：启动 Spring Boot 应用
使用我们之前调优过的 JVM 参数启动 Java 程序。此时由于 Milvus 已经在 `19530` 端口就绪，Java 程序应该能瞬间连上。

```bash
# 进入你的 Jar 包目录
java -Xms512m -Xmx600m -jar your-rag-project.jar
```
*注：我将 `-Xmx` 稍微收窄到了 600m，这样能给 OS 留出更多余量。*

#### 第三步：最终内存动态监控
打开一个新窗口，输入以下命令实时监控物理内存：
```bash
watch -n 1 free -h
```
**健康指标：**
- **Used:** 3.2G ~ 3.6G（这是最理想的状态，说明资源被充分利用且没有溢出）。
- **Available:** 只要不低于 **150MB**，系统就不会触发 `OOM Killer` 杀掉进程。

---

### ⚠️ 一个关键的排查技巧 (如果 Java 报错)

如果你的 Spring Boot 启动时报 `Connection Refused`（连接 Milvus 失败）：

1.  **检查防火墙：** 阿里云安全组是否放行了 `19530` 端口（如果 Java 和 Docker 在同一台机子，通常不需要，但如果是内网访问则需注意）。
2.  **检查 Host：** 在 `application.yml` 中，`milvus.client.host` 应该填写 `localhost` 或 `127.0.0.1`。

### 🌟 总结
你现在的系统架构非常健壮：
- **容器层：** 严格限额，不会因为数据库索引构建突然吃光内存。
- **模型层：** 动态释放，5 分钟不使用自动退回内存。
- **应用层：** JVM 枷锁，保证 Java 不会无节制膨胀。

**现在，你的私有化 RAG 系统已经完全具备了实战能力。快去上传一份 PDF 文档，试试检索“秒回”的快感吧！**

---

## 六、 应用层调优与对齐

### 1. Spring Boot 启动调优 (JVM 限制)
禁止 Java 无节制占用物理内存：
```bash
# 限制最大堆内存为 1GB，预留空间给 Ollama 推理
java -Xms512m -Xmx1024m -jar policy-rag-system.jar
```

### 2. application.yml 配置对齐
**⚠️ 关键：向量维度 (Dimension) 必须严格匹配！**
```yaml
spring:
  ai:
    ollama:
      chat:
        model: qwen2.5:1.5b    # 逻辑能力与速度的平衡点
      embedding:
        model: bge-m3          # 回归高质量 Embedding 模型
    vectorstore:
      milvus:
        collection-name: policy_docs
        # ⚠️ 如果将 bge-small (512) 换成了 bge-m3 (1024)，此处必须修改
        embedding-dimension: 1024 
```

---
## 七、 Spring AI 核心配置调优 (1.0.0-M6 版)

在 M6 版本中，自动配置存在不稳定性，常导致 `collection-name` 失效（报错寻找默认的 `vector_store`）。

### 1. 强制手动配置方案 (VectorStoreConfig)

**由于 YAML 绑定可能失效，建议通过 Java 配置类强行接管主权。**

* **技术点：** 注入官方源码类 `MilvusVectorStoreProperties`，并开启属性绑定。
* **必杀技：** 显式设置 `.initializeSchema(true)` 解决“集合不存在”导致的搜索崩溃。

```java
@Configuration
@EnableConfigurationProperties(MilvusVectorStoreProperties.class)
public class VectorStoreConfig {
    @Bean
    @Primary
    public MilvusVectorStore vectorStore(MilvusServiceClient client, 
                                         EmbeddingModel model, 
                                         MilvusVectorStoreProperties properties) {
        return MilvusVectorStore.builder(client, model)
            .collectionName(properties.getCollectionName())
            .embeddingDimension(properties.getEmbeddingDimension())
            .initializeSchema(true) // 启动时自动检查/创建集合
            .build();
    }
}
```

### 2. YAML 配置对齐

确保路径严格匹配 `spring.ai.vectorstore.milvus`（注意 M6 以后通常不带横杠）。

```yaml
spring:
  ai:
    ollama:
      base-url: http://服务器IP:11434
    vectorstore:
      milvus:
        client:
          host: 服务器IP
          port: 19530
        collection-name: policy_docs
        embedding-dimension: 1024 # 必须与 BGE-M3 一致
```

---
## 八、 避坑与进阶操作

1.  **维度冲突处理:** 如果你更换了 Embedding 模型，必须**删除并重建** Milvus 的 Collection。512 维与 1024 维不匹配会导致检索时抛出异常。
2.  **解决“首次运行卡顿”：预热加载**
    在启动 Java 程序前，手动触发一次预热，让 Ollama 提前将模型加载至内存：
    `curl http://localhost:11434/api/embeddings -d '{"model": "bge-m3", "prompt": "warmup"}'`
3.  **最终健康度监控:** 执行 `free -h`。
    - **Available:** 只要 **> 200MB**，系统就是安全的。
    - **Swap:** 占用 1G-2G 是正常现象。

**💡 专家建议：** 
后期若业务增加，可考虑将推理 API 托管至 **阿里云 DashScope (通义千问 API)**。届时服务器仅运行 Spring Boot + Milvus，内存占用会降至 1.5G 以下，系统将变得飞速且极其稳定！

---
**✨ 总结：** 从 2G 到 4G，你完成了从“不可用”到“生产级”的跨越。祝你的 RAG 系统运行愉快！