# Spring Boot 3.4 + Spring AI + Milvus：企业级 RAG 智能问答系统全栈实施指南

## 📖 1. 架构总览与方案选型

本系统基于 **Spring Boot 3.4**、**Spring AI**、**Milvus** 向量数据库及 **Ollama** 本地大模型构建，旨在实现企业私有知识库的高效检索与智能问答。

### 1.1 RAG 系统核心流转
1.  **翻译官（Embedding 模型）**：将非结构化文档（PDF/Word）转化为高维向量。
2.  **记忆库（Milvus 向量数据库）**：负责海量向量的存储、索引及相似度毫秒级检索。
3.  **决策大脑（LLM 大模型）**：结合检索出的上下文背景生成精准回答。

### 1.2 硬件与模型选型建议
| 维度 | 🌟 路线 A：标准方案 (推荐) | 💻 路线 B：低配方案 (无 GPU/4G 内存) |
| :--- | :--- | :--- |
| **硬件底线** | ≥ 16GB 内存，显存 ≥ 8GB | 2核 4G / 4核 8G，纯 CPU |
| **对话模型** | `qwen:7b` (或 `moe-7b` 混合专家) | `qwen2.5:1.5b` 或 `0.5b` |
| **向量模型** | **`bge-m3`** (维度 **1024**) | `bge-small-zh` (维度 **512**) |
| **核心挑战** | 性能调优、高并发 | **必须开启 Swap**、精简依赖 |

---

## 🛠️ 2. 基础设施部署 (Docker & Milvus & Ollama)

### 2.1 环境准备 (以阿里云 Ubuntu 22.04 为例)
在云服务器环境，首要解决网络和资源限制问题。

#### 2.1.1 内存救赎：开启 8GB Swap (低配机器必做)
```bash
sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h # 验证 Swap 不为 0
```

#### 2.1.2 Docker 一键极速安装与镜像加速
```bash
# 安装 Docker
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
sudo systemctl enable --now docker

# 配置国内镜像加速 (解决 i/o timeout)
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://你的阿里云加速器地址",
    "https://docker.m.daocloud.io",
    "https://docker.nju.edu.cn"
  ]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### 2.2 部署 Milvus (v2.4.0+)
推荐使用 Docker Compose 独立部署。
1. **下载配置**：
   ```bash
   mkdir -p /usr/milvus && cd /usr/milvus
   wget https://github.com/milvus-io/milvus/releases/download/v2.4.0/milvus-standalone-docker-compose.yml -O docker-compose.yml
   sed -i '/version:/d' docker-compose.yml # 移除过时标签
   ```
2. **启动服务**：
   ```bash
   docker compose up -d
   ```
3. **检查状态**：`docker compose ps`。确保端口 `19530` 已开启。

### 2.3 Ollama 配置与远程访问
1. **拉取模型**：
   ```bash
   ollama pull qwen2.5:1.5b
   ollama pull bge-m3  # 强烈推荐 BGE-M3 作为向量模型
   ```
2. **解锁跨域与远程监听**：
   ```bash
   sudo mkdir -p /etc/systemd/system/ollama.service.d/
   echo -e '[Service]\nEnvironment="OLLAMA_HOST=0.0.0.0"' | sudo tee /etc/systemd/system/ollama.service.d/override.conf
   sudo systemctl daemon-reload && sudo systemctl restart ollama
   ```

---

## 💻 3. 后端工程开发 (Spring Boot 3.4 + Spring AI)

### 3.1 核心依赖与编译排障 (`pom.xml`)
**警告：** Java 25 EA 编译器与旧版 Lombok 不兼容（报错 `TypeTag :: UNKNOWN`）。建议降级至 **Java 21 LTS** 并锁定 Lombok 版本。

```xml
<properties>
    <java.version>21</java.version>
    <spring-ai.version>1.0.0-M6</spring-ai.version>
    <lombok.version>1.18.36</lombok.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-milvus-store-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
    </dependency>
</dependencies>

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

### 3.2 配置文件 (`application.yml`)
**注意：** `embedding-dimension` 必须与模型输出维度（BGE-M3 为 1024）严格一致，否则会触发“维度陷阱”。

```yaml
spring:
  ai:
    ollama:
      base-url: http://<服务器IP>:11434
      chat:
        model: qwen2.5:1.5b
        options:
          temperature: 0.3
      embedding:
        model: bge-m3
    vectorstore:
      milvus:
        client:
          host: <服务器IP>
          port: 19530
        collection-name: policy_docs
        embedding-dimension: 1024
  mvc:
    async:
      request-timeout: 300000 # 5分钟异步超时，防止 AI 响应慢断开

logging:
  level:
    com.ailearn.rag: DEBUG
    org.springframework.ai: DEBUG # 开启日志以查看真实 Prompt
  file:
    name: logs/rag.log
  logback:
    rollingpolicy:
      file-name-pattern: logs/rag-%d{yyyy-MM-dd}.%i.log
      max-file-size: 100MB
```

### 3.3 核心业务逻辑 (适配 Spring AI 1.0.0+ API)
新版本中 `SearchRequest` 使用 **Builder 模式**，`Document` 获取文本使用 **`getText()`**。

```java
@Service
public class RagService {
    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.chatClient = chatClientBuilder.build();
        this.vectorStore = vectorStore;
    }

    public Flux<String> streamAnswer(String query) {
        // 1. 向量相似度检索 (Top 3)
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(3)
                .similarityThreshold(0.6)
                .build()
        );

        // 2. 组装上下文与来源
        String context = docs.stream().map(Document::getText).collect(Collectors.joining("\n\n"));
        String refs = docs.stream()
            .map(d -> (String) d.getMetadata().getOrDefault("filename", "未知来源"))
            .distinct().collect(Collectors.joining(", "));

        // 3. 构建 Prompt 并流式返回结果
        String prompt = "基于上下文回答问题：\n" + context + "\n\n问题：" + query;
        return chatClient.prompt().user(prompt).stream().content()
                .concatWith(Flux.just("\n\n---\n📚 来源: " + refs));
    }
}
```

---

## 🖥️ 4. 前端交互实现 (Vue 3)

### 4.1 项目创建
```bash
npm create vue@latest # 推荐 Vite
cd <project-name> && npm install
npm install @microsoft/fetch-event-source # 专业流解析库
```

### 4.2 专业流式解析逻辑
避免手动解析 `reader.read()` 产生的 SSE 协议头（如 `data:` 字符）。

```javascript
import { fetchEventSource } from '@microsoft/fetch-event-source';

const sendMessage = async (query) => {
    const assistantMsg = { role: 'assistant', content: '' };
    messages.value.push(assistantMsg);

    await fetchEventSource('/api/chat/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query }),
        onmessage(ev) {
            if (ev.data) assistantMsg.content += ev.data; 
        },
        onclose() { /* 处理关闭 */ },
        onerror(err) { throw err; }
    });
};
```

---

## 🚨 5. 核心避坑指南与常见故障排查

### 5.1 编译与版本报错
*   **`RestClientAutoConfiguration not present`**: 
    *   **原因**：Spring AI 1.0.0+ 依赖 `RestClient`。
    *   **解决**：确保 Spring Boot 版本 ≥ 3.2.0。
*   **`TypeTag :: UNKNOWN`**: 
    *   **原因**：JDK 25/Lombok 冲突。
    *   **解决**：降级至 JDK 21 并在 `maven-compiler-plugin` 显式配置 Lombok 路径。

### 5.2 Milvus 连接与性能问题
*   **`Connection Refused (19530)`**:
    1. 检查阿里云安全组是否开放 19530。
    2. 检查 Milvus 容器是否 OOM（查看 `docker stats`）。
    3. 如果是容器间通信，Host 请使用容器名 `milvus-standalone` 而非 `localhost`。
*   **“维度陷阱”**:
    *   如果 `application.yml` 的维度（如 1024）与模型（如 bge-small 为 512）不符，入库或检索会报错。更换模型必须删除旧的 Collection。

### 5.3 生产维护建议
*   **Portainer 超时报错**: 若安装后 5 分钟未设密码被锁定，执行 `docker restart portainer` 重置计时。
*   **日志归档**: 使用 `logback-spring.xml` 将 ERROR 日志与常规日志分离，方便快速定位 RAG 检索失败的原因。
*   **网络隔离**: 生产环境下，Etcd、MinIO、Milvus 的内部端口（除 19530 外）尽量不要暴露给公网。

---
**结语**：构建 RAG 系统是一项系统工程，从底层的 Swap 配置到前端的流解析库选择，每一环都决定了最终的体验。在资源受限环境，优先保证 **Swap 空间** 和 **维度匹配** 是成功的关键。