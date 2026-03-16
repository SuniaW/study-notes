

# 🏛️ 政务政策智能问答系统 (RAG) 完整落地指南

## 📑 1. 系统概述与技术栈
本系统旨在通过 RAG（检索增强生成）架构，实现对政务政策文档（PDF/Word）的自动解析、向量化入库，并提供基于本地大模型的流式精准问答与引用溯源功能。

*   **后端框架**: Spring Boot 3.4.3 + Spring AI 1.0.0-M1 + Java 21
*   **前端框架**: Vue 3 + TypeScript + Vite + markdown-it
*   **大语言模型 (LLM)**: Ollama + Qwen-7B (或 Qwen2.5-1.5B)
*   **向量模型 (Embedding)**: Ollama + BGE-M3 (维度: 1024)
*   **向量数据库**: Milvus (Docker 单机版)

---

## 🚦 2. 硬件与部署策略规划

请根据实际服务器资源选择部署路线：

| 维度 | 🏢 **企业高配版 (推荐)** | 💻 **个人低配版 (无 GPU)** |
| :--- | :--- | :--- |
| **服务器规格** | 16G 内存 + GPU 显卡 | 2核 4G / 4核 8G (纯 CPU) |
| **LLM 模型** | `qwen:7b` (回答严谨，逻辑强) | `qwen2.5:1.5b` (极速，资源占用低) |
| **关键系统设置**| Ollama 常驻内存保活 | **必须开启 8G Swap 虚拟内存**防崩溃 |
| **预期性能** | 首字延迟 < 1s | 首字延迟 < 2s |

---

## 🛠️ 3. 基础设施部署 (Infrastructure)

### 3.1 启动 Milvus 向量数据库
使用 Docker 快速拉取并启动 Milvus：
```bash
wget https://github.com/milvus-io/milvus/releases/download/v2.3.0/milvus-standalone-docker-compose.yml -O docker-compose.yml
sudo docker compose up -d
```
*(注：确保服务器防火墙已放行 `19530` 端口)*

### 3.2 部署 Ollama 与模型
在服务器上安装 Ollama 并拉取所需模型（此处以标准版为例）：
```bash
# 1. 安装 Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 2. 拉取对话模型与向量模型
ollama pull qwen:7b
ollama pull bge-m3

# 3. (可选) 设置模型常驻内存，消除冷启动延迟
ollama run qwen:7b
```

---

## ⚙️ 4. 后端研发 (Spring Boot + Spring AI)

### 4.1 `pom.xml` 核心依赖
*注意：已修复 JDK 21 下的 Lombok 兼容性问题，并配置了 Spring 里程碑仓库。*

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.3</version>
</parent>

<properties>
    <java.version>21</java.version>
    <spring-ai.version>1.0.0-M1</spring-ai.version>
    <lombok.version>1.18.36</lombok.version> <!-- 必须升级以支持 JDK21 -->
</properties>

<dependencies>
    <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
    <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-webflux</artifactId></dependency>
    <dependency><groupId>org.springframework.ai</groupId><artifactId>spring-ai-ollama-spring-boot-starter</artifactId></dependency>
    <dependency><groupId>org.springframework.ai</groupId><artifactId>spring-ai-milvus-spring-boot-starter</artifactId></dependency>
    <dependency><groupId>org.springframework.ai</groupId><artifactId>spring-ai-tika-document-reader</artifactId></dependency>
    <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId><optional>true</optional></dependency>
</dependencies>

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

<repositories>
    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
        <snapshots><enabled>false</enabled></snapshots>
    </repository>
</repositories>
```

### 4.2 `application.yml` 配置
```yaml
server:
  port: 8080
spring:
  application:
    name: policy-rag-system
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        model: qwen:7b
        options:
          temperature: 0.3 # 政务问答需降低发散性
      embedding:
        model: bge-m3
    vectorstore:
      milvus:
        client:
          host: localhost
          port: 19530
        collection-name: policy_docs
        embedding-dimension: 1024  # ⚠️ 必须与 bge-m3 的 1024 维保持一致
        index-type: HNSW           # 高性能向量索引
        metric-type: COSINE
```

### 4.3 核心 Java 业务代码

**1. 知识库入库服务 (`IngestionService.java`)**
```java
@Service
@RequiredArgsConstructor
public class IngestionService {
    private final VectorStore vectorStore;

    public void processDocument(MultipartFile file) throws IOException {
        TikaDocumentReader loader = new TikaDocumentReader(new InputStreamResource(file.getInputStream()));
        // 分块处理，保留上下文重叠区域
        TextSplitter splitter = new TokenTextSplitter(500, 100, 10, 10000, true);
        List<Document> docs = splitter.apply(loader.get());
        
        docs.forEach(doc -> doc.getMetadata().put("filename", file.getOriginalFilename()));
        vectorStore.add(docs);
    }
}
```

**2. RAG 问答服务 (`RagService.java`)**
```java
@Service
@RequiredArgsConstructor
public class RagService {
    private final ChatClient.Builder chatClientBuilder;
    private final VectorStore vectorStore;

    public Flux<String> streamAnswer(String query) {
        ChatClient client = chatClientBuilder
            .defaultSystem("你是一个专业的政务政策解读助手。请基于提供的【政策上下文】回答问题，不要编造。")
            .build();

        // 检索 Milvus 中 Top 3 的相关文档片段
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(3).withSimilarityThreshold(0.6));
            
        String context = docs.stream().map(Document::getContent).collect(Collectors.joining("\n\n"));
        String refs = docs.stream().map(d -> (String) d.getMetadata().get("filename")).distinct().collect(Collectors.joining(", "));

        String prompt = "【政策上下文】\n" + context + "\n\n【用户问题】\n" + query;

        // 发起流式请求并在流末尾追加引用来源
        return client.prompt(prompt).stream().content()
                .concatWith(Flux.just("\n\n---\n📚 **引用来源**: " + (refs.isEmpty() ? "知识库暂未收录" : refs)));
    }
}
```

**3. 控制层接口 (`ChatController.java`)**
```java
@RestController
@RequestMapping("/api")
@CrossOrigin(origins = "*") // 开发期允许跨域
@RequiredArgsConstructor
public class ChatController {
    private final RagService ragService;
    private final IngestionService ingestionService;

    @GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chat(@RequestParam String query) { return ragService.streamAnswer(query); }

    @PostMapping("/upload")
    public String upload(@RequestParam("file") MultipartFile file) throws IOException {
        ingestionService.processDocument(file);
        return "上传成功";
    }
}
```

---

## 💻 5. 前端研发 (Vue 3 + TS)

### 5.1 环境准备
```bash
npm install markdown-it
npm install @types/markdown-it -D
```

### 5.2 核心页面代码 (`App.vue`)
*集成了左侧导航、流式对话呈现、Markdown 自动渲染与文件拖拽上传。已解决 TS 类型警告。*

```vue
<template>
  <div class="app-container">
    <aside class="sidebar">
      <div class="logo">🏛️ 政务智脑</div>
      <nav>
        <div :class="['nav-item', currentTab === 'chat' ? 'active' : '']" @click="currentTab = 'chat'">💬 政策问答</div>
        <div :class="['nav-item', currentTab === 'upload' ? 'active' : '']" @click="currentTab = 'upload'">📂 知识库上传</div>
      </nav>
    </aside>

    <main class="main-content">
      <!-- 问答视图 -->
      <div v-if="currentTab === 'chat'" class="chat-view">
        <header class="view-header">
          <h2>政策咨询助手</h2><span class="status-badge">🟢 模型在线</span>
        </header>

        <div class="messages-container" ref="messagesContainer">
          <div v-for="(msg, index) in messages" :key="index" :class="['message-row', msg.role]">
            <div class="avatar">{{ msg.role === 'user' ? '🧑‍💻' : '⚖️' }}</div>
            <div class="bubble">
              <div class="markdown-body" v-html="md.render(msg.content)"></div>
            </div>
          </div>
        </div>

        <div class="input-area">
          <div class="input-wrapper">
            <input v-model="userInput" @keyup.enter="sendMessage" placeholder="请输入政策问题..." :disabled="loading"/>
            <button @click="sendMessage" :disabled="loading || !userInput.trim()">发送</button>
          </div>
        </div>
      </div>

      <!-- 上传视图 -->
      <div v-else class="upload-view">
        <div class="upload-card">
          <div class="upload-zone" @dragover.prevent @drop.prevent="handleDrop">
            <h3>点击或拖拽上传政策文件</h3>
            <p>支持 PDF, Word (.docx, .doc)</p>
            <input type="file" @change="handleFileSelect" accept=".pdf,.doc,.docx" />
          </div>
          <div v-if="uploadStatus" class="status-msg">{{ uploadStatus }}</div>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick, watch } from 'vue';
import MarkdownIt from 'markdown-it';

interface ChatMessage { role: 'user' | 'assistant'; content: string; }

const currentTab = ref<'chat' | 'upload'>('chat');
const md = new MarkdownIt({ html: true, breaks: true });
const userInput = ref('');
const messages = ref<ChatMessage[]>([]);
const loading = ref(false);
const messagesContainer = ref<HTMLElement | null>(null);
const uploadStatus = ref('');

// 自动滚动
watch(() => messages.value, () => {
  nextTick(() => { if (messagesContainer.value) messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight; });
}, { deep: true });

// 流式问答核心逻辑
const sendMessage = async () => {
  if (!userInput.value.trim() || loading.value) return;
  const query = userInput.value;
  messages.value.push({ role: 'user', content: query });
  userInput.value = '';
  loading.value = true;

  const assistantMsg: ChatMessage = { role: 'assistant', content: '' };
  messages.value.push(assistantMsg);

  try {
    const response = await fetch(`http://localhost:8080/api/chat?query=${encodeURIComponent(query)}`);
    if (!response.body) throw new Error("Body is null"); // 修复 TS 警告

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      assistantMsg.content += decoder.decode(value, { stream: true });
      if (messagesContainer.value) messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight;
    }
  } catch (e) {
    assistantMsg.content += "\n[系统错误]";
  } finally {
    loading.value = false;
  }
};

// 文件上传逻辑
const handleFileSelect = (e: Event) => uploadFile((e.target as HTMLInputElement).files![0]);
const handleDrop = (e: DragEvent) => uploadFile(e.dataTransfer!.files[0]);

const uploadFile = async (file: File) => {
  uploadStatus.value = `正在入库: ${file.name}...`;
  const formData = new FormData(); formData.append('file', file);
  const res = await fetch('http://localhost:8080/api/upload', { method: 'POST', body: formData });
  uploadStatus.value = res.ok ? `✅ 上传成功！` : `❌ 上传失败`;
};
</script>

<style scoped>
/* 样式代码较长，请参考上一轮回答中的完整 CSS，包含侧边栏、聊天气泡、拖拽虚线框等设计 */
.app-container { display: flex; height: 100vh; font-family: sans-serif; }
.sidebar { width: 260px; background: #1e293b; color: white; }
.main-content { flex: 1; display: flex; flex-direction: column; }
/* ... 此处省略基础样式 ... */
</style>
```

---

## 🎯 6. 系统避坑与自检清单 (Checklist)

在正式运行或联调前，请对照以下问题排查：

1.  🔴 **应用无法启动 / 报找不到类的错误**
    *   **原因**：Spring AI 目前在里程碑阶段，未发布到中央仓库。
    *   **解决**：检查 `pom.xml` 底部是否包含了 `<repository><id>spring-milestones</id>...` 且 Maven 已重新 Reload。
2.  🔴 **编译报错：`TypeTag :: UNKNOWN`**
    *   **原因**：JDK 21 与旧版 Lombok 冲突。
    *   **解决**：确保 `pom.xml` 强制指定了 `<lombok.version>1.18.36</lombok.version>`。
3.  🔴 **报错：Milvus 插入向量维度不匹配**
    *   **原因**：YAML 配置的维度与拉取的 Embedding 模型不一致。
    *   **解决**：确保 `application.yml` 中 `embedding-dimension` 是 `1024`（对应 `bge-m3`）。如果是 `bge-small` 则是 `512`。
4.  🔴 **内存溢出 (OOM) 或服务器死机**
    *   **原因**：低配服务器（如 2核 4G）内存不足。
    *   **解决**：执行 `sudo fallocate -l 8G /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` 强制开启虚拟内存。