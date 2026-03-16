# 🚀 终极实施指南：基于 Spring AI + Milvus 的 RAG 问答系统

> **文档说明**：本文档为经过深度整理的终极架构指南。全面整合了**Spring Boot 3.4 + JDK 21 环境下的底层依赖纠错**，并针对**企业级生产**与**个人低配实验**场景，提供了两套开箱即用的物理部署方案。

**🎯 核心目标**：构建一个支持 PDF/Word 解析、向量存储、流式响应（SSE）的智能问答系统。
**🛠️ 技术底座**：`Spring Boot 3.4.3（稳定版）` + `Spring AI 1.0.0-M1（里程碑版）` + `Milvus` + `Ollama` + `Vue3`

---

## 🚦 模块零：场景与硬件路线选择 (必读)

请根据你手头的服务器硬件资源，选择对应的实施路线（Path A 或 Path B）：

| 特性对比 | 🌟 Path A：标准企业级方案 (推荐) | 💻 Path B：低配/个人实验方案 |
| :--- | :--- | :--- |
| **适用场景** | 生产环境、演示汇报、高性能要求 | 个人学习、阿里云低配 ECS、无显卡环境 |
| **硬件底线** | 至少 16GB 内存，建议配备 NVIDIA 显卡 | 2核4G / 4核8G，纯 CPU 运算 |
| **对话模型** | `qwen:7b` (4-bit 量化，逻辑严密) | `qwen2.5:1.5b` (极致轻量，速度极快) |
| **向量模型** | `bge-m3` (多语言优异，维度 **1024**) | `bge-small-zh` (极小巧，维度 **512**) |
| **预期性能** | 首字延迟 < 1s，推理顺滑 | 首字延迟 < 2s，勉强流畅 |
| **保命手段** | 模型常驻内存、GPU 加速 | **开启 Swap 虚拟内存**、模型降级、限制 JVM |

---

## ☁️ 模块一：云端环境与保命配置 (基础设施)

### 1. 内存扩容 (Path B 低配机器必做 🚨)
如果你的服务器内存不足 8G（如阿里云 2C4G），**请务必先执行此步**，否则后续启动 Milvus 或编译 Java 时系统会因 OOM（内存溢出）直接崩溃！
```bash
# 1. 创建 8G 虚拟内存文件并赋权
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. 设置开机永久生效
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 3. 验证是否成功 (查看 Swap 栏)
free -h
```

### 2. 启动 Milvus 向量数据库
无论走哪条路线，都推荐使用 Docker Standalone 模式部署。
```bash
# 下载官方 Docker Compose 文件
wget https://github.com/milvus-io/milvus/releases/download/v2.3.0/milvus-standalone-docker-compose.yml -O docker-compose.yml

# Path B 用户注意：建议修改 docker-compose.yml，给 milvus 容器增加 mem_limit: 2g 限制
# 启动服务
sudo docker compose up -d
```

---

## 🧠 模块二：AI 模型层部署 (Ollama)

### 1. 拉取模型 (按路线选择)

**🌟 Path A (标准配置)：**
```bash
ollama pull qwen:7b   # 对话大模型
ollama pull bge-m3    # 向量模型 (维度 1024)
```

**💻 Path B (低配配置)：**
```bash
ollama pull qwen2.5:1.5b
# ⚠️ 注意：bge-small-zh 官方库不存在，推荐直接使用 bge-m3。
# 即使是低配机器，bge-m3 多占的几百MB内存也是值得的，能省去大量手动配置麻烦且精度更高。
```

> 💡 **关于 `bge-small-zh` 拉取报错的解决方案：**
> 若遇到 `Error: pull model manifest: file does not exist`，说明官方库无此镜像。
> **强烈建议：** 直接改为 `ollama pull bge-m3`。（并在后续 yml 配置中将维度改为 `1024`）。
> **备选方案：** 若内存极端受限（如 1C2G），必须手动下载 GGUF 文件并通过 `Modelfile` 构建。
>**手动导入 BGE-Small-ZH (极限低配 Path B)需执行以下步骤：**
>1. **下载 GGUF：** `wget https://huggingface.co/C-K-L/bge-small-zh-v1.5-GGUF/resolve/main/bge-small-zh-v1.5-q4_k_m.gguf -O bge-small.gguf`
>2. **创建 Modelfile：**
  > ```dockerfile
   >FROM ./bge-small.gguf
   >TEMPLATE ""
   >PARAMETER num_ctx 512
   >```
>3. **构建：** `ollama create bge-small-zh -f Modelfile`
---

### 2. 开启 Ollama 远程访问访问 (解决 Connection refused)
默认 Ollama 仅限 `127.0.0.1` 访问，前后端分离或云端部署必须放开 IP 限制。

```bash
# 1. 强制写入覆盖配置 (绕过 Vim 复杂操作，直接搞定)
sudo mkdir -p /etc/systemd/system/ollama.service.d/
echo -e '[Service]\nEnvironment="OLLAMA_HOST=0.0.0.0"' | sudo tee /etc/systemd/system/ollama.service.d/override.conf

# 2. 重载系统配置并重启 Ollama
sudo systemctl daemon-reload
sudo systemctl restart ollama

# 3. 终极验证 (看到 0.0.0.0:11434 或 :::11434 即代表成功)
netstat -tulnp | grep 11434
```
验证：浏览器中输入http://阿里云公网ip:11434/api/tags显示如下

```json
{
  "models": [
    {
      "name": "bge-m3:latest",
      "model": "bge-m3:latest",
      "modified_at": "2026-02-28T20:37:02.169994841+08:00",
      "size": 1157672605,
      "digest": "7907646426070047a77226ac3e684fbbe8410524f7b4a74d02837e43f2146bab",
      "details": {
        "parent_model": "",
        "format": "gguf",
        "family": "bert",
        "families": [
          "bert"
        ],
        "parameter_size": "566.70M",
        "quantization_level": "F16"
      }
    },
    {
      "name": "qwen2.5:1.5b",
      "model": "qwen2.5:1.5b",
      "modified_at": "2026-02-27T20:15:37.660182142+08:00",
      "size": 986061892,
      "digest": "65ec06548149b04c096a120e4a6da9d4017ea809c91734ea5631e89f96ddc57b",
      "details": {
        "parent_model": "",
        "format": "gguf",
        "family": "qwen2",
        "families": [
          "qwen2"
        ],
        "parameter_size": "1.5B",
        "quantization_level": "Q4_K_M"
      }
    }
  ]
}
```

> 🛡️ **阿里云/腾讯云安全组提醒**：请务必在云控制台的“安全组-入方向规则”中，放行 **`11434` (Ollama)** 和 **`19530` (Milvus)** 端口。

---

## ☕ 模块三：后端架构与核心代码 (Spring Boot)

### 1. 终极兼容版 `pom.xml` (解决 JDK21 + Lombok 报错)
🚨 **避坑解析**：之前出现的 `TypeTag :: UNKNOWN` 报错，根本原因是 Maven 3.9+ 对 XML 标签闭合要求极其严格，且旧版 Lombok 无法解析 JDK21 字节码。请**完全替换**你的 POM 文件，并在 IDEA 中 Reload。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.3</version>
        <relativePath/>
    </parent>

    <groupId>com.ailearn.governmentaffairsrag</groupId>
    <artifactId>spring-ai-policy-rag-system</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <java.version>21</java.version>
        <!-- ⚠️ 核心修复：锁定兼容 JDK 21 的 Lombok 版本 -->
        <lombok.version>1.18.36</lombok.version> 
        <spring-ai.version>1.0.0-M6</spring-ai.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Web & 流式响应 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>

        <!-- Spring AI 核心组件 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-milvus-store-spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-tika-document-reader</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <!-- ⚠️ 必加：Spring AI M版依赖的里程碑仓库 -->
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots><enabled>false</enabled></snapshots>
        </repository>
    </repositories>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
> 🧹 **替换 POM 后的关键操作：** 
> Windows 用户请执行 `rmdir /s /q "D:\Program Files\maven\repository\org\projectlombok\lombok"` 清除旧缓存，然后执行 `mvn clean install -U` 强制刷新。

### 2. `application.yml` 配置文件
⚠️ **极其重要**：`embedding-dimension` 必须与你拉取的模型对应！`bge-m3` = **1024**，`bge-small` = **512**。

```yaml
spring:
  application:
    name: policy-rag-system
  ai:
    ollama:
      base-url: http://<你的阿里云公网IP>:11434  # 本地开发替换为云端IP
      chat:
        model: qwen:7b   # Path B 用户改为 qwen2.5:1.5b
        options:
          temperature: 0.3 # 政策问答需严谨，降低发散性
      embedding:
        model: bge-m3    # 强烈推荐统一使用 bge-m3
    vectorstore:
      milvus:
        client:
          host: <你的阿里云公网IP>
          port: 19530
        collection-name: policy_docs
        embedding-dimension: 1024  # ⚠️ 匹配 bge-m3 的维度
        index-type: HNSW           # 使用 HNSW 保证检索速度
```

### 3. 核心业务代码

**A. 文档解析与入库服务 (`IngestionService.java`)**
```java
@Service
@RequiredArgsConstructor
public class IngestionService {
    private final VectorStore vectorStore;

    public void processDocument(MultipartFile file) throws IOException {
        // 1. Tika 万能文档解析
        TikaDocumentReader loader = new TikaDocumentReader(new InputStreamResource(file.getInputStream()));
        
        // 2. Token 智能切片 (保留上下文重叠度，防止语义截断)
        TextSplitter splitter = new TokenTextSplitter(500, 100, 10, 10000, true);
        List<Document> splitDocuments = splitter.apply(loader.get());
        
        // 3. 注入文件名元数据，用于追溯来源
        splitDocuments.forEach(doc -> doc.getMetadata().put("filename", file.getOriginalFilename()));
        
        // 4. 向量化并持久化至 Milvus
        vectorStore.add(splitDocuments);
    }
}
```

**B. RAG 流式问答服务 (`RagService.java`)**
```java
package com.wx.policyragsystem.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.document.Document; // 确保导包正确！
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    // 推荐在构造时直接构建好 ChatClient 实例
    public RagService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.chatClient = chatClientBuilder.build();
        this.vectorStore = vectorStore;
    }

    public Flux<String> streamAnswer(String query) {

        // 1. 向量数据库相似度检索 (Top 3，阈值 0.6)
        // 使用新版的 Builder 模式
        List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(query)
                        .topK(3)
                        .similarityThreshold(0.6)
                        .build()
        );

        // 2. 组装上下文与引用来源
        String context = docs.stream()
                .map(Document::getText)    // ✅ 改为 getText() 即可
                .collect(Collectors.joining("\n\n"));

        String refs = docs.stream()
                .map(d -> (String) d.getMetadata().getOrDefault("filename", "未知来源"))
                .distinct()
                .collect(Collectors.joining(", "));

        // 3. 构建 Prompt
        String prompt = "基于以下政策上下文回答问题：\n" + context + "\n\n问题：" + query;

        // 4. 流式(SSE)返回内容，并在末尾拼接来源参考
        // 新版 ChatClient 推荐使用 .prompt().user(prompt) 语法
        return chatClient.prompt()
                .user(prompt)
                .stream()
                .content()
                .concatWith(Flux.just("\n\n---\n📚 来源: " + refs));
    }
}
```

**C. 控制层 (`ChatController.java`)**

```java
@RestController
@RequestMapping("/api")
@CrossOrigin(origins = "*") // 允许前端跨域
@RequiredArgsConstructor
public class ChatController {

    private final RagService ragService;
    private final IngestionService ingestionService;

    // 流式问答接口 (SSE)
    @GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chat(@RequestParam String query) {
        return ragService.streamAnswer(query);
    }

    // 文档上传接口
    @PostMapping("/upload")
    public String upload(@RequestParam("file") MultipartFile file) throws IOException {
        ingestionService.processDocument(file);
        return "上传并解析成功: " + file.getOriginalFilename();
    }
}
```
**Spring AI 核心配置调优 (1.0.0-M6 版)**
>解决报错
io.milvus.exception.ServerException: collection not found[database=default][collection=vector_store]
在 M6 版本中，自动配置存在不稳定性，常导致 `collection-name` 失效（报错寻找默认的 `vector_store`）。

 **1. 强制手动配置方案 (VectorStoreConfig)**

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

**2. YAML 配置对齐**

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
## 🎨 模块四：前端打字机效果实现 (Vue 3)

使用原生的 `fetch` API 和 `ReadableStream` 接收后端的 `Flux` 流，实现丝滑的打字机输出体验。

```vue
<template>
  <div class="chat-container">
    <div class="messages" ref="msgBox">
      <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
        <div class="content" v-html="renderMarkdown(msg.content)"></div>
      </div>
    </div>

    <div class="input-area">
      <input
        v-model="userInput"
        @keyup.enter="sendMessage"
        placeholder="请输入政策问题..."
        :disabled="loading"
      />
      <button @click="sendMessage" :disabled="loading">发送</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import MarkdownIt from 'markdown-it'

const md = new MarkdownIt()
const userInput = ref('')
const messages = ref<{ role: string; content: string }[]>([])
const loading = ref(false)

const renderMarkdown = (text: string) => {
  return md.render(text)
}

const sendMessage = async () => {
  if (!userInput.value.trim()) return

  const query = userInput.value
  messages.value.push({ role: 'user', content: query })
  userInput.value = ''
  loading.value = true

  // 预占位 Assistant 的回复
  const assistantMsg = { role: 'assistant', content: '' }
  messages.value.push(assistantMsg)

  try {
    const response = await fetch(
      `http://localhost:8080/api/chat/stream?query=${encodeURIComponent(query)}`,
    )
    const reader = response.body!.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      const chunk = decoder.decode(value, { stream: true })
      // 实时追加内容
      assistantMsg.content += chunk
    }
  } catch (error) {
    assistantMsg.content += `[系统错误: 响应中断${error}`
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
/* 简单的 CSS 样式 */
.chat-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}
.message {
  margin-bottom: 15px;
  padding: 10px;
  border-radius: 8px;
}
.user {
  background-color: #e3f2fd;
  text-align: right;
}
.assistant {
  background-color: #f5f5f5;
}
</style>

```

---

## ✅ 最终上线检查清单 (Checklist)

在全面启动测试前，请依次核对：
- [ ]  **Maven 仓库**：确认 `pom.xml` 中已包含 `<repositories>` 且能够访问 `repo.spring.io`。
- [ ]  **Lombok 版本**：确认 Maven 下载的 Lombok jar 包版本为 `1.18.36` (若 IDE 报错，请开启注解处理并清理缓存)。
- [ ]  **Milvus 维度**：确认 `application.yml` 中 `embedding-dimension: 1024`，与 `bge-m3` 模型一致。
- [ ] **Ollama 模型**：确认 `ollama list` 中存在 `qwen:7b` 和 `bge-m3`。
- [ ]  **性能优化**：
    *   Milvus 索引类型设为 `HNSW`。
    *   生产环境建议设置 Ollama 环境变量 `OLLAMA_KEEP_ALIVE=-1` 防止模型卸载导致下次请求卡顿。


- [ ] **编译正常**：执行 `mvn clean compile` 不再报 `TypeTag` 或 `No plugin found` 错误。
- [ ] **公网打通**：在本地电脑终端执行 `curl http://<阿里云IP>:11434/api/tags`，能成功返回 JSON 格式的模型列表。
- [ ] **维度匹配**：仔细确认 `application.yml` 中的 `embedding-dimension` 是 `1024`，且模型指定为 `bge-m3`。
- [ ] **内存安全** (仅 Path B)：`free -h` 确认 Swap 分区已挂载，启动后端时添加了参数限制：`java -Xms1g -Xmx2g -jar app.jar`。

🎉 **按照这份全景指南操作，你的 RAG 系统将彻底扫除本地 JDK、Maven 依赖以及云端网络、内存 OOM 等一切障碍，实现平稳运行！**