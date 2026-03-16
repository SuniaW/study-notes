
# 🏛️ Spring AI + Milvus + Ollama 离线 RAG 系统全栈指南
**（阿里云 4GB 内存环境专项优化版）**

本指南旨在指导开发者在资源受限的环境下，构建一套稳定、流畅、支持 PDF 检索和流式响应的 AI 政策问答系统。

---

## 一、 环境基座：基础设施调优

### 1. 内存救赎：开启 8GB Swap
在 4GB 物理内存运行大模型（2.2G）+ Milvus（1G）+ Spring Boot（0.5G）时，必须开启虚拟内存防止系统由于 OOM 崩溃。
```bash
sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 2. Ollama 远程访问与资源限制
*   **解锁监听**：`sudo SYSTEMD_EDITOR=vim systemctl edit ollama.service`
*   **添加配置**：在顶部写入：
    ```ini
    [Service]
    Environment="OLLAMA_HOST=0.0.0.0"
    Environment="OLLAMA_KEEP_ALIVE=5m"
    ```
*   **重载**：`sudo systemctl daemon-reload && sudo systemctl restart ollama`

---

## 二、 后端构建：Spring Boot 3.4 深度修复

### 1. 解决 JDK 21 / Lombok 兼容性
针对 `TypeTag :: UNKNOWN` 报错，必须在 `pom.xml` 中强制指定编译路径。

```xml
<properties>
    <java.version>21</java.version>
    <lombok.version>1.18.36</lombok.version>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>${lombok.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 2. 核心配置文件 (`application.yml`)
**关键点**：延长异步超时，防止 AI 检索慢导致连接断开。

```yaml
spring:
  ai:
    ollama:
      base-url: http://127.0.0.1:11434 # 同台机器建议用 127.0.0.1
      chat:
        model: qwen2.5:1.5b
      embedding:
        model: bge-m3 # 维度 1024
    vectorstore:
      milvus:
        client:
          host: 127.0.0.1
          port: 19530
        embedding-dimension: 1024
  mvc:
    async:
      request-timeout: 300000 # 5分钟异步超时，解决 IOException 核心配置
```

### 3. 优化后的 `RagService`
使用弹性线程池处理阻塞检索，并增加“首字激活”和“心跳检测”。

```java
@Service
public class RagService {
    // ... 注入 Builder 和 Store
    public Flux<String> streamAnswer(String query) {
        Flux<String> answerFlux = Flux.defer(() -> {
            // 向量检索：使用 boundedElastic 线程池防止阻塞 Netty
            List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder().query(query).topK(3).similarityThreshold(0.5).build()
            );
            if (docs.isEmpty()) return Flux.just("未找到相关政策。");

            String context = docs.stream().map(Document::getText).collect(Collectors.joining("\n"));
            String refs = docs.stream().map(d -> (String) d.getMetadata().get("filename")).distinct().collect(Collectors.joining(", "));

            return chatClient.prompt().user("上下文："+context+"\n问题："+query)
                    .stream().content()
                    .concatWith(Flux.just("\n\n---\n📚 来源: " + refs));
        }).subscribeOn(Schedulers.boundedElastic());

        // 合并流：首个空格激活前端 -> 业务流 -> 每15秒心跳
        return Flux.concat(Flux.just(" "), answerFlux)
                   .keepAlive(Duration.ofSeconds(15), s -> " ");
    }
}
```

---

## 三、 前端构建：Vue 3 + 专业流解析

### 1. 解决 SSE 协议解析 (消除 `data:` 字段)
使用 `@microsoft/fetch-event-source` 库，并显式禁止无限重连。

**安装：** `npm install @microsoft/fetch-event-source`

**核心逻辑：**
```typescript
const sendMessage = async () => {
  const ctrl = new AbortController();
  const assistantMsg = { role: 'assistant', content: '' };
  messages.value.push(assistantMsg);

  await fetchEventSource('/api/chat?query=...', {
    method: 'GET',
    signal: ctrl.signal,
    onmessage(ev) {
      if (ev.data) assistantMsg.content += ev.data; // 库已自动过滤 data: 前缀
    },
    onclose() {
      ctrl.abort(); // 正常关闭，切断库的自动重连
      loading.value = false;
    },
    onerror(err) {
      ctrl.abort(); // 异常关闭，切断重连循环
      loading.value = false;
      throw err; // 抛出异常以终止
    }
  });
}
```

---

## 四、 常见问题及排障 (Troubleshooting)

### 1. 现象：后端一直报错 `IOException` 且前端无限重连
*   **原因**：Spring 异步超时（默认 10s）比 AI 出字慢。
*   **修复**：修改 `application.yml` 中的 `spring.mvc.async.request-timeout=300000`。

### 2. 现象：Milvus 报 `UNAVAILABLE: io exception`
*   **原因**：Milvus 进程被 OOM Killer 杀死，或者连接地址用了公网 IP 导致不稳定。
*   **修复**：
    1.  `docker ps` 检查容器。
    2.  `application.yml` 中 host 改为 `127.0.0.1`。
    3.  Docker 限制 Milvus 内存为 `800M`。

### 3. 现象：回答中有 `data:硕士` 类似的字样
*   **原因**：前端手动解析 `reader.read()` 却没过滤 SSE 协议头。
*   **修复**：改用 `@microsoft/fetch-event-source` 库。

### 4. 现象：升级 4G 内存后依然很卡
*   **原因**：`bge-m3` 模型推理吃 CPU，且 4G 内存运行两个模型仍会触发大量磁盘交换（Swap）。
*   **优化**：
    1.  将 Embedding 模型换成 `shaw/dpoint-bge-small-zh-v1.5` (512维)。
    2.  将 Chat 模型换成 `qwen2.5:0.5b` 测试。

---

## 五、 最终上线检查清单
1.  [ ] **安全组**：放行 8081 (Web), 11434 (Ollama), 19530 (Milvus)。
2.  [ ] **JVM 参数**：启动时增加 `-Xms256m -Xmx512m` 预留内存给系统。
3.  [ ] **Nginx** (如有)：务必配置 `proxy_buffering off;`。
4.  [ ] **维度对齐**：`bge-m3` 对应 1024，`bge-small` 对应 512，不匹配会报 Search 失败。