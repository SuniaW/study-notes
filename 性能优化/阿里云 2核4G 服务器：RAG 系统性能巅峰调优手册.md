这份技术手册详尽汇编了我们在 **阿里云 2核4G** 环境下，针对私有化 RAG（检索增强生成）系统进行的**系统级瘦身、Ollama 性能压榨、Java 代码重构及自动化运维**的全套实战方案。

---

# 🚀 阿里云 2核4G 服务器：RAG 系统全栈性能调优与实战手册

> **项目环境：** 2核 CPU / 4GB 内存 / 8GB Swap
> **核心组件：** Spring Boot 3.4.3 + Spring AI 1.0.0-M6 + Ollama (Qwen2.5) + Milvus 2.4.0
> **核心痛点：** CPU 长期 100% 满载、内存触底（仅剩 100MB 可用）、Swap 交换频繁导致 I/O 假死、首字响应时间超过 15 秒。

---

## 第一阶段：系统级“深层排毒”（OS 运维优化）

在资源受限环境下，必须清理所有非必要的系统开销，将物理内存留给 AI 核心任务。

### 1. 移除云厂商冗余监控 (释放约 150-200MB)
阿里云盾（AliYunDun）在资源告急时会产生明显的 CPU 采样开销和内存占用。
*   **执行指令：**
    ```bash
    wget http://update.aegis.aliyun.com/download/uninstall.sh && chmod +x uninstall.sh && ./uninstall.sh
    wget http://update.aegis.aliyun.com/download/quartz_uninstall.sh && chmod +x quartz_uninstall.sh && ./quartz_uninstall.sh
    pkill aliyun-service && rm -rf /usr/local/aegis /usr/sbin/aliyun-service
    ```

### 2. 内核参数与日志调优
*   **Swappiness (10)**：强制系统优先使用物理内存，减少磁盘 I/O 等待。
    `echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p`
*   **日志限额**：防止 `systemd-journald` 占用过多内存。
    `sudo journalctl --vacuum-size=100M`
*   **清理 Docker 冗余**：释放无用镜像、挂载容器产生的隐形缓存。
    `docker system prune -a -f`

### 3. 手动释放内核缓存 (buff/cache)
在内存负载极高（`avail` 低于 500MB）时，手动回收 PageCache、dentries 和 inodes。
*   **操作指令：**
    ```bash
    sync; echo 3 > /proc/sys/vm/drop_caches
    ```

---

## 第二阶段：中间件组件性能压榨（Ollama 专项调优）

针对 2核 CPU，默认设置会导致推理时系统调度完全卡死。

### 1. 强制限制推理线程 (关键优化)
默认 Ollama 会尝试占满所有核心（100% CPU）。限制其只使用一个核心，留出另一个核心处理 OS 调度、Milvus 检索和 Java 业务。
*   **操作步骤：** 执行 `sudo systemctl edit ollama.service`，添加：
    ```ini
    [Service]
    Environment="OLLAMA_NUM_THREAD=1"      # 强制模型只使用 1 个线程，防止把 CPU 撑爆
    Environment="OLLAMA_KEEP_ALIVE=5m"     # 模型闲置 5 分钟自动释放物理内存
    Environment="OLLAMA_NUM_PARALLEL=1"    # 串行处理，严防并发推理导致 OOM
    Environment="OLLAMA_MAX_LOADED_MODELS=1" # 显存受限，内存中只允许存在一个模型
    ```
*   **激活：** `sudo systemctl daemon-reload && sudo systemctl restart ollama`

### 2. 模型架构降级：1.5b -> 0.5b (性能质变)
*   **Qwen2.5:1.5b**：占用约 1.1GB 内存，推理时 CPU 压力极大。
*   **Qwen2.5:0.5b**：占用仅约 **400MB** 内存。能够完全驻留在物理内存中，响应速度提升 **3 倍以上**，且在 RAG 场景下表现已足够。
*   **操作指令：**
    ```bash
    ollama pull qwen2.5:0.5b
    ollama rm qwen2.5:1.5b
    ```

---

## 第三阶段：Java 代码深度重构（Ingestion & Rag Service）

### 1. IngestionService：并行解析与分批入库
针对 4G 环境，避免一次性向远程向量库发送上千条数据导致连接超时。

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class IngestionService {
    private final VectorStore vectorStore;
    private final TokenTextSplitter splitter = new TokenTextSplitter(400, 100, 5, 10000, true);

    public void processDocuments(MultipartFile[] files) {
        // 1. 利用 parallelStream 开启并行文件解析 (提升 Tika 效率)
        List<Document> allSplitDocs = Arrays.stream(files)
                .parallel()
                .flatMap(file -> {
                    try {
                        TikaDocumentReader loader = new TikaDocumentReader(new InputStreamResource(file.getInputStream()));
                        return splitter.apply(loader.get()).stream()
                                .peek(doc -> doc.getMetadata().put("filename", file.getOriginalFilename()));
                    } catch (IOException e) {
                        log.error("解析失败: {}", file.getOriginalFilename(), e);
                        return null;
                    }
                })
                .filter(Objects::nonNull).collect(Collectors.toList());

        // 2. 严控分批入库：每批 32 条最稳健，防止远程请求超时
        int batchSize = 32;
        for (int i = 0; i < allSplitDocs.size(); i += batchSize) {
            int end = Math.min(i + batchSize, allSplitDocs.size());
            vectorStore.add(allSplitDocs.subList(i, end));
        }
    }
}
```

### 2. RagService：缩短首字响应时间 (TTFT)
核心逻辑：**减少 `topK` 以缩短上下文，异步化检索任务。**

```java
@Service
@Slf4j
public class RagService {
    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public Flux<String> streamAnswer(String query, String chatId) {
        long startTime = System.currentTimeMillis();
        return Mono.fromCallable(() -> {
                // 优化检索：topK=2 或 3 是 2核服务器的极限平衡点。
                // 减少 Context 长度能直接缩短 0.5b 模型的预处理时间。
                SearchRequest searchRequest = SearchRequest.builder()
                        .query(query).topK(3).similarityThreshold(0.5).build();
                return vectorStore.similaritySearch(searchRequest);
            })
            .subscribeOn(Schedulers.boundedElastic()) // 异步非阻塞检索
            .flatMapMany(docs -> {
                String context = docs.stream().map(Document::getText).collect(Collectors.joining("\n"));
                String references = docs.stream().map(d -> (String) d.getMetadata().get("filename")).distinct().collect(Collectors.joining(", "));

                return chatClient.prompt()
                        .user(u -> u.text("背景：{context}\n问题：{query}").param("query", query).param("context", context))
                        .advisors(a -> a.param(MessageChatMemoryAdvisor.DEFAULT_CHAT_MEMORY_CONVERSATION_ID, chatId))
                        .stream().content()
                        .concatWith(Flux.just("\n\n---\n📚 来源: " + references));
            });
    }
}
```

---

## 第四阶段：自动化运维与健康维护

### 1. 每日凌晨 3:00 深度维护脚本
创建脚本 `/home/admin/scripts/daily_maint.sh` 解决内存碎片和 Swap 堆积。

```bash
#!/bin/bash
LOG_FILE="/var/log/daily_maint.log"
echo "--- $(date) 开始每日例行维护 ---" >> $LOG_FILE

# 1. 强制重启 Ollama：释放被模型长期占用的物理内存
systemctl restart ollama
sleep 5

# 2. 清理系统内核缓存 (释放 Buff/Cache)
sync && echo 3 > /proc/sys/vm/drop_caches

# 3. 预热模型：模拟一次生成，防止第二天首个用户查询等待加载
curl http://127.0.0.1:11434/api/generate -d '{"model": "qwen2.5:0.5b", "prompt": "hi", "stream": false}' > /dev/null

echo "--- 维护完成 ---" >> $LOG_FILE
```
*   **定时任务配置**：执行 `sudo crontab -e`
    `0 3 * * * /home/admin/scripts/daily_maint.sh`（凌晨3点全量清理）
    `0 * * * * sync && echo 3 > /proc/sys/vm/drop_caches`（每小时释放内核缓存）

---

## 第五阶段：最终运行健康指标 (Checklist)

1.  **Avail Mem (可用内存)**：
    *   **2.5GB+**：清爽态，系统反应极快。
    *   **1.0GB+**：正常态，Spring Boot 启动。
    *   **200MB - 500MB**：运行态，0.5b 推理中。
2.  **I/O Wait (wa)**：
    *   执行 `top` 查看 `wa`。只要 `wa` 接近 **0.0%**，说明没有产生频繁磁盘交换，操作不会卡顿。
3.  **内网调用对齐**：
    *   `base-url` 务必使用 `http://127.0.0.1:11434`，公网调用会平增 200ms 以上的网络抖动。
4.  **Java 内存红线**：
    *   启动参数必须限制：`java -Xms256m -Xmx512m -XX:MaxMetaspaceSize=128m -jar app.jar`。

**✨ 总结：** 通过 **“物理瘦身、线程限制、模型降级、异步重构”** 的四维调优，即便在 2核4G 的极端环境下，私有化 RAG 系统也能实现从 15 秒到 **3-5 秒** 的丝滑响应跨越。