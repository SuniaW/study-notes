# 🚀 阿里云 2核4G 服务器：私有化 RAG 全链路全栈调优与落地指南

本手册旨在指导开发者在资源极度受限的 **阿里云 2核4G** 环境下，从底层系统优化到上层应用实现，构建一套高性能、高可靠的私有化 RAG（检索增强生成）系统。

---

## 一、 核心生存法则：资源资产负债表 (4GB 内存分配)

在 4G 内存中运行 **Milvus + Ollama + Spring Boot**，必须执行“资源精细化管理”，严防 OOM（内存溢出）导致系统假死。

| 组件 | 内存分配上限 | 调优核心手段 |
| :--- | :--- | :--- |
| **Milvus 数据库** | **1.0 GB** | Docker `deploy.resources.limits` 硬限制 + 内部缓存限额 |
| **Ollama (LLM+Embed)** | **2.2 GB** | 设置 `KEEP_ALIVE` 动态释放 + 限制并发线程 |
| **Spring Boot (JVM)** | **0.6 GB** | 参数 `-Xmx600m -XX:MaxMetaspaceSize=256m` |
| **OS 预留** | **0.2 GB** | 卸载冗余插件，维持基础响应 |

---

## 二、 第一阶段：系统级“深层脱水”（释放物理资源）

### 1. 移除云厂商冗余监控 (回血 ~200MB)
阿里云盾（AliYunDun）及其监控插件在内存告急时会产生明显的资源争抢和 CPU 采样开销。
```bash
# 运行官方卸载脚本
wget http://update.aegis.aliyun.com/download/uninstall.sh && chmod +x uninstall.sh && ./uninstall.sh
wget http://update.aegis.aliyun.com/download/quartz_uninstall.sh && chmod +x quartz_uninstall.sh && ./quartz_uninstall.sh
# 强制清理残留进程
pkill aliyun-service && rm -rf /usr/local/aegis /usr/sbin/aliyun-service
```

### 2. 开启 Swap 虚拟内存与内核优化
防止内存瞬间触底导致 OOM 杀掉核心进程。
```bash
# 1. 创建 8G 虚拟内存
sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 2. 调低 Swappiness (建议 10)
# 强制系统优先驻留物理内存，减少 I/O Wait (wa)
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# 3. 限制系统日志大小
sudo journalctl --vacuum-size=100M
```

---

## 三、 第二阶段：数据库层（Milvus 高可靠部署）

### 1. 解决端口冲突 (卸载 Portainer)
如果遇到 `9000 failed: port is already allocated`，通常是 Portainer 占用了端口。建议卸载以节省约 100MB 内存：
```bash
docker stop portainer && docker rm portainer
docker volume rm portainer_data
```

### 2. 精化版 `docker-compose.yml`
```yaml
services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    deploy:
      resources:
        limits: { memory: 256M }

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    command: minio server /minio_data
    deploy:
      resources:
        limits: { memory: 512M }

  standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.4.0
    environment:
      CACHE_SIZE: 512M # 核心优化：限制内部索引缓存
    ports: ["19530:19530"]
    deploy:
      resources:
        limits: { memory: 1024M } # 严控 1G 线
```

---

## 四、 第三阶段：模型层（Ollama 推理压榨）

### 1. 强制限制推理线程 (关键优化)
默认 Ollama 会尝试占满所有核心。限制其使用单线程，留出核心处理业务。
```bash
sudo systemctl edit ollama.service
# 添加以下内容
[Service]
Environment="OLLAMA_NUM_THREAD=1"      # 强制模型只使用 1 个线程
Environment="OLLAMA_KEEP_ALIVE=5m"     # 5分钟不使用自动释放内存
Environment="OLLAMA_NUM_PARALLEL=1"    # 串行处理防止瞬时崩溃
Environment="OLLAMA_HOST=0.0.0.0"      # 允许内网跨容器访问
```
`sudo systemctl daemon-reload && sudo systemctl restart ollama`

### 2. 模型选型与预热
*   **推荐模型**：`qwen2.5:0.5b`（推理极快，400MB 内存占用）或 `qwen2.5:1.5b`（1.1GB 占用）。
*   **预热技巧**：防止首次搜索超时。
    ```bash
    curl http://127.0.0.1:11434/api/embeddings -d '{"model": "bge-m3", "prompt": "warmup"}'
    ```

---

## 五、 第四阶段：应用层（Spring AI 实现与避坑）

### 1. 解决 `collection-name` 配置失效 Bug
在 Spring AI 早期版本（如 1.0.0-M6）中，YAML 配置可能被忽略。必须使用 Java 配置类强制接管。

```java
@Configuration
@EnableConfigurationProperties(MilvusVectorStoreProperties.class)
public class VectorStoreConfig {
    @Bean
    @Primary
    public MilvusVectorStore vectorStore(MilvusServiceClient client, 
                                         EmbeddingModel model, 
                                         MilvusVectorStoreProperties properties) {
        MilvusVectorStore.MilvusVectorStoreConfig config = MilvusVectorStore.MilvusVectorStoreConfig.builder()
            .withCollectionName(properties.getCollectionName())
            .withEmbeddingDimension(1024) // BGE-M3 对应 1024 维
            .build();
        return MilvusVectorStore.builder(client, model)
            .collectionName(properties.getCollectionName())
            .initializeSchema(true) // 关键：启动时自动创建集合
            .build();
    }
}
```

### 2. 高效 Ingestion（分批入库逻辑）
避免一次性处理大文件导致 OOM。
```java
public void processDocument(MultipartFile file) throws IOException {
    TikaDocumentReader loader = new TikaDocumentReader(new InputStreamResource(file.getInputStream()));
    TokenTextSplitter splitter = new TokenTextSplitter(400, 100, 5, 10000, true);
    List<Document> docs = splitter.apply(loader.get());
    
    // 每批 32 条最稳健
    int batchSize = 32;
    for (int i = 0; i < docs.size(); i += batchSize) {
        vectorStore.add(docs.subList(i, Math.min(i + batchSize, docs.size())));
    }
}
```

---

## 六、 第五阶段：前端表现层（Vue 3 + SSE）

### 1. 流式响应解析 (原生 Fetch)
避免第三方库的自动重连机制在 500 错误时压死服务器。
```typescript
const response = await fetch(`${API_BASE}/chat?query=${encodeURIComponent(text)}`, {
  signal: abortController.signal
});
const reader = response.body.getReader();
const decoder = new TextDecoder();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  const chunk = decoder.decode(value, { stream: true });
  // 处理 SSE 格式中的 "data:" 前缀
  assistantMsg.content += chunk.replace(/^data:/gm, '');
}
```

### 2. UI/UX 优化
*   **Markdown 渲染**：使用 `markdown-it` 处理标题、表格。
*   **思考时长计时**：通过首字下行（TTFT）计算 RAG 检索效率。

---

## 七、 第六阶段：自动化运维“生存法则”

### 1. 严禁本地编译
绝对不要在 4G 服务器运行 `mvn clean install`，这会诱发系统 OOM。请在本地编译 JAR 包后上传。

### 2. 每日凌晨例行维护脚本
创建 `daily_maint.sh` 并加入 crontab：
```bash
#!/bin/bash
# 1. 重启 Ollama 释放内存碎片
systemctl restart ollama
# 2. 清理系统 PageCache
sync && echo 3 > /proc/sys/vm/drop_caches
# 3. 模型预热
curl http://127.0.0.1:11434/api/generate -d '{"model": "qwen2.5:0.5b", "prompt": "hi"}'
```

### 3. 可视化管理 (Attu) 避坑
**严禁在服务器上额外部署 Attu 可视化镜像**。请下载 Attu 桌面客户端，通过阿里云公网 IP 远程连接 Milvus (19530)。

---

## 💡 总结：运行健康指标 (Checklist)

*   **内存监控**：执行 `free -m`，`avail` 需保持在 200MB 以上。
*   **内网对齐**：`application.yml` 中的 `host` 必须使用 `127.0.0.1` 以绕过公网防火墙并降低延迟。
*   **维度一致性**：`BGE-M3` 必须对应 `1024` 维度，否则 `SimilaritySearch` 会报错。
*   **I/O 等待**：执行 `top` 查看 `%wa`。若长期高于 5%，说明 Swap 交换过度，建议降级使用 `0.5b` 模型。

通过上述 **“系统瘦身 + 线程硬限 + 异步分批 + 内存置换”** 的组合拳，你的阿里云 2核4G 服务器将能够平稳运行一套完整的私有化政策智库系统。