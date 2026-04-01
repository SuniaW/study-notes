# 专题五：《Spring AI 后端开发实战：30 行代码实现 RAG 核心逻辑》

## 📚 企业级 RAG 智能问答系统全栈实施指南

### 第五部分：Spring AI 后端开发

---

## 第 1 章 Spring AI 核心概念与架构

### 1.1 什么是 Spring AI？

Spring AI 是 Spring 官方推出的 AI 应用开发框架，旨在简化人工智能应用的开发流程，提供统一的 API 接口和开箱即用的集成能力。

```
┌─────────────────────────────────────────────────────────────┐
│                    Spring AI 架构                            │
├─────────────────────────────────────────────────────────────┤
│  应用层 → ChatClient/EmbeddingClient → AI Model → 响应       │
│           ↓              ↓                                   │
│  向量存储 ← VectorStore ← 文档处理 ← 数据源                   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Spring AI 核心组件

| 组件 | 接口 | 功能 | 使用场景 |
|------|------|------|----------|
| **ChatClient** | `org.springframework.ai.chat.client` | 与大模型对话 | RAG 问答、聊天机器人 |
| **EmbeddingClient** | `org.springframework.ai.embedding` | 文本向量化 | 文档索引、语义检索 |
| **VectorStore** | `org.springframework.ai.vectorstore` | 向量存储管理 | 知识库检索、相似搜索 |
| **ToolExecution** | `org.springframework.ai.tool` | 工具调用 | Function Calling、Agent |
| **ChatMemory** | `org.springframework.ai.chat.memory` | 会话记忆 | 多轮对话、上下文保持 |

### 1.3 Spring AI 版本演进（2026 最新版）

| 版本 | 发布时间 | 状态 | 建议 | 主要变化 |
|------|----------|------|------|----------|
| 1.0.0-M1~M8 | 2024-2025 | 里程碑版 | ❌ 不推荐生产 | API 不稳定 |
| 1.0.0 GA | 2025-05-20 | 正式版 | ⚠️ 可用 | API 稳定 |
| 1.0.2 | 2026-01-15 | 最新稳定版 | ✅ 强烈推荐 | Bug 修复、性能优化 |
| 1.1.0 | 2026-06 (计划) | 开发中 | 🔜 期待 | Agents 框架增强 |

> 📢 **重要提醒**：本书代码基于 Spring AI 1.0.2 版本编写，这是 2026 年 1 月发布的最新稳定版。

### 1.4 Spring AI vs 其他 AI 框架对比

| 特性 | Spring AI | LangChain4j | LlamaIndex | Haystack |
|------|-----------|-------------|------------|----------|
| 学习曲线 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| Spring 集成 | 原生 | 需适配 | 需适配 | 需适配 |
| 向量存储支持 | 10+ | 15+ | 8+ | 12+ |
| 文档质量 | 官方完善 | 社区驱动 | 官方完善 | 官方完善 |
| 生产就绪 | ✅ 是 | ✅ 是 | ✅ 是 | ✅ 是 |
| 中文社区 | 活跃 | 非常活跃 | 一般 | 一般 |

---

## 第 2 章 项目初始化与依赖配置

### 2.1 Maven 依赖配置（pom.xml）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.3</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>rag-system</artifactId>
    <version>1.0.0</version>
    <name>RAG 智能问答系统</name>
    
    <properties>
        <java.version>21</java.version>
        <spring-ai.version>1.0.2</spring-ai.version>
        <milvus-sdk.version>2.4.0</milvus-sdk.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <!-- Spring AI BOM -->
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
        <!-- Spring Boot 核心 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Spring AI 核心 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-model-ollama</artifactId>
        </dependency>
        
        <!-- Spring AI 向量存储（Milvus） -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-milvus-store</artifactId>
        </dependency>
        
        <!-- Milvus Java SDK -->
        <dependency>
            <groupId>io.milvus</groupId>
            <artifactId>milvus-sdk-java</artifactId>
            <version>${milvus-sdk.version}</version>
        </dependency>
        
        <!-- 文档解析（Apache Tika） -->
        <dependency>
            <groupId>org.apache.tika</groupId>
            <artifactId>tika-core</artifactId>
            <version>2.9.1</version>
        </dependency>
        
        <!-- SSE 流式响应 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        
        <!-- Lombok（简化代码） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- 测试依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
    
    <!-- Spring AI Milestone 仓库（1.0.2 不需要，仅旧版本需要） -->
    <!-- 
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
        </repository>
    </repositories>
    -->
</project>
```

### 2.2 应用配置文件（application.yml）

```yaml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  application:
    name: rag-system
  
  # Spring AI 配置
  ai:
    # Ollama 大模型配置
    ollama:
      base-url: http://localhost:11434
      chat:
        model: qwen2.5:1.5b
        options:
          temperature: 0.7
          top-p: 0.9
          num-predict: 2048
      embedding:
        model: bge-m3
    
    # 向量存储配置（Milvus）
    vectorstore:
      milvus:
        host: localhost
        port: 19530
        username: 
        password: 
        database: default
        collection-name: rag_documents
        index-type: HNSW
        metric-type: COSINE
        embedding-dimension: 1024
        initialize-schema: true
    
    # 文档分块配置
    document:
      splitter:
        chunk-size: 500
        chunk-overlap: 100

  # 数据源配置（可选，用于 ChatMemory）
  datasource:
    url: jdbc:h2:mem:ragdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 

# 日志配置
logging:
  level:
    root: INFO
    com.example.rag: DEBUG
    org.springframework.ai: DEBUG
    io.milvus: INFO

# 自定义配置
rag:
  system:
    # 检索配置
    retrieval:
      top-k: 5
      similarity-threshold: 0.6
    # 流式响应配置
    streaming:
      enabled: true
      chunk-size: 1
```

### 2.3 依赖冲突解决方案

| 冲突类型 | 症状 | 解决方案 | 优先级 |
|----------|------|----------|--------|
| Spring AI 版本冲突 | `NoSuchMethodError` | 统一使用 BOM 管理版本 | 🔴 高 |
| Milvus SDK 版本冲突 | `ClassNotFoundException` | 锁定 milvus-sdk-java 版本 | 🔴 高 |
| Netty 版本冲突 | `IncompatibleClassChangeError` | 强制指定 netty 版本 | 🟡 中 |
| SLF4J 冲突 | `Multiple bindings` | 排除多余日志实现 | 🟡 中 |
| Jackson 版本冲突 | `JsonMappingException` | 统一 spring-boot 管理 | 🟢 低 |

```xml
<!-- 依赖冲突解决示例 -->
<dependency>
    <groupId>io.milvus</groupId>
    <artifactId>milvus-sdk-java</artifactId>
    <version>2.4.0</version>
    <exclusions>
        <!-- 排除冲突的日志依赖 -->
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 强制指定版本 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>4.1.100.Final</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 第 3 章 30 行代码实现 RAG 核心逻辑

### 3.1 核心服务类（RagService.java）

```java
package com.example.rag.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.search.SearchRequest;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

import java.util.List;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class RagService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    
    // 核心方法：RAG 检索增强生成
    public String query(String question) {
        // 1. 向量检索（Top 5 相关文档）
        List<Document> docs = vectorStore.similaritySearch(question);
        
        // 2. 组装上下文
        String context = buildContext(docs);
        
        // 3. 构建 Prompt
        String prompt = buildPrompt(question, context);
        
        // 4. 调用 LLM 生成回答
        return chatClient.prompt(prompt).call().content();
    }
    
    // 流式响应方法
    public Flux<String> queryStream(String question) {
        List<Document> docs = vectorStore.similaritySearch(question);
        String context = buildContext(docs);
        String prompt = buildPrompt(question, context);
        
        return chatClient.prompt(prompt).stream().content();
    }
    
    // 文档入库方法
    public void addDocument(String text, Map<String, Object> metadata) {
        Document doc = new Document(text, metadata);
        vectorStore.add(List.of(doc));
        log.info("✅ 文档已向量化并存储");
    }
    
    // 构建上下文
    private String buildContext(List<Document> docs) {
        return docs.stream()
            .map(Document::getContent)
            .collect(java.util.stream.Collectors.joining("\n\n"));
    }
    
    // 构建 Prompt
    private String buildPrompt(String question, String context) {
        return """
        你是一名政务政策问答助手。请基于以下政策文档片段回答问题。
        如果文档中没有相关信息，请如实告知用户。
        
        【政策文档】
        %s
        
        【用户问题】
        %s
        
        【回答】
        """.formatted(context, question);
    }
}
```

> 💡 **代码解析**：以上核心服务类仅 30 行代码（不含注释和空行），实现了 RAG 系统的核心逻辑：检索→组装→生成。

### 3.2 控制器层（RagController.java）

```java
package com.example.rag.controller;

import com.example.rag.service.RagService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

import java.util.Map;

@RestController
@RequestMapping("/rag")
@RequiredArgsConstructor
public class RagController {
    
    private final RagService ragService;
    
    // 普通问答接口
    @PostMapping("/query")
    public Map<String, String> query(@RequestBody Map<String, String> request) {
        String question = request.get("question");
        String answer = ragService.query(question);
        return Map.of("answer", answer);
    }
    
    // 流式问答接口（SSE）
    @PostMapping(value = "/query/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> queryStream(@RequestBody Map<String, String> request) {
        String question = request.get("question");
        return ragService.queryStream(question);
    }
    
    // 文档入库接口
    @PostMapping("/document")
    public Map<String, String> addDocument(@RequestBody Map<String, Object> request) {
        String text = (String) request.get("text");
        @SuppressWarnings("unchecked")
        Map<String, Object> metadata = (Map<String, Object>) request.get("metadata");
        ragService.addDocument(text, metadata);
        return Map.of("status", "success");
    }
}
```

### 3.3 配置类（AiConfig.java）

```java
package com.example.rag.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {
    
    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder.build();
    }
    
    @Bean
    public VectorStore vectorStore(VectorStore vectorStore) {
        return vectorStore;
    }
    
    @Bean
    public EmbeddingModel embeddingModel(EmbeddingModel embeddingModel) {
        return embeddingModel;
    }
}
```

---

## 第 4 章 SSE 流式响应实现

### 4.1 SSE 协议详解

| 特性 | SSE | WebSocket | HTTP 轮询 |
|------|-----|-----------|----------|
| 通信方向 | 单向（服务器→客户端） | 双向 | 单向 |
| 协议 | HTTP/HTTPS | WebSocket | HTTP |
| 重连机制 | 自动 | 需手动 | 需手动 |
| 浏览器支持 | 现代浏览器 | 现代浏览器 | 所有浏览器 |
| 适用场景 | 流式文本、新闻推送 | 聊天、游戏 | 简单场景 |

### 4.2 后端 SSE 实现（完整代码）

```java
package com.example.rag.controller;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;
import reactor.core.publisher.Flux;

import java.io.IOException;
import java.util.concurrent.CompletableFuture;

@Slf4j
@RestController
@RequestMapping("/sse")
@RequiredArgsConstructor
public class SseController {
    
    private final RagService ragService;
    
    /**
     * SSE 流式问答接口
     * 数据格式：data:{"content":"第","type":"content"}
     */
    @PostMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter chatStream(@RequestBody Map<String, String> request) {
        String question = request.get("question");
        
        // 创建 SSE 发射器（超时时间 300 秒）
        SseEmitter emitter = new SseEmitter(300_000L);
        
        // 异步处理流式响应
        CompletableFuture.runAsync(() -> {
            try {
                Flux<String> flux = ragService.queryStream(question);
                
                flux.subscribe(
                    // 收到数据
                    content -> {
                        try {
                            emitter.send(SseEmitter.event()
                                .name("message")
                                .data(Map.of("content", content, "type", "content")));
                        } catch (IOException e) {
                            log.error("SSE 发送失败", e);
                        }
                    },
                    // 错误处理
                    error -> {
                        log.error("流式生成错误", error);
                        try {
                            emitter.send(SseEmitter.event()
                                .name("error")
                                .data(Map.of("message", error.getMessage(), "type", "error")));
                            emitter.complete();
                        } catch (IOException e) {
                            log.error("SSE 错误发送失败", e);
                        }
                    },
                    // 完成处理
                    () -> {
                        try {
                            emitter.send(SseEmitter.event()
                                .name("message")
                                .data(Map.of("content", "", "type", "done")));
                            emitter.complete();
                        } catch (IOException e) {
                            log.error("SSE 完成发送失败", e);
                        }
                    }
                );
            } catch (Exception e) {
                log.error("SSE 处理异常", e);
                emitter.completeWithError(e);
            }
        });
        
        // 连接关闭回调
        emitter.onCompletion(() -> log.info("SSE 连接完成"));
        emitter.onTimeout(() -> {
            log.warn("SSE 连接超时");
            emitter.complete();
        });
        emitter.onError(e -> log.error("SSE 连接错误", e));
        
        return emitter;
    }
    
    /**
     * 心跳检测接口（防止连接断开）
     */
    @GetMapping(value = "/heartbeat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter heartbeat() {
        SseEmitter emitter = new SseEmitter(0L); // 永不过期
        
        // 每 30 秒发送心跳
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
        scheduler.scheduleAtFixedRate(() -> {
            try {
                emitter.send(SseEmitter.event()
                    .name("heartbeat")
                    .data(Map.of("timestamp", System.currentTimeMillis())));
            } catch (IOException e) {
                scheduler.shutdown();
                emitter.complete();
            }
        }, 30, 30, TimeUnit.SECONDS);
        
        emitter.onCompletion(() -> scheduler.shutdown());
        
        return emitter;
    }
}
```

### 4.3 前端 Vue3 对接示例

```vue
<!-- ChatComponent.vue -->
<template>
  <div class="chat-container">
    <div class="messages">
      <div v-for="(msg, index) in messages" :key="index" 
           :class="['message', msg.role]">
        {{ msg.content }}
      </div>
      <div v-if="isStreaming" class="message assistant streaming">
        {{ streamingContent }}<span class="cursor">|</span>
      </div>
    </div>
    
    <div class="input-area">
      <input v-model="question" @keyup.enter="sendMessage" 
             placeholder="请输入问题..." />
      <button @click="sendMessage" :disabled="isStreaming">发送</button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const question = ref('')
const messages = ref([])
const isStreaming = ref(false)
const streamingContent = ref('')

const sendMessage = () => {
  if (!question.value.trim() || isStreaming.value) return
  
  // 添加用户消息
  messages.value.push({ role: 'user', content: question.value })
  
  // 创建 SSE 连接
  const eventSource = new EventSource('/api/sse/chat', {
    headers: { 'Content-Type': 'application/json' }
  })
  
  // 由于 EventSource 不支持 POST，改用 fetch + ReadableStream
  fetch('/api/sse/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ question: question.value })
  })
  .then(response => {
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    isStreaming.value = true
    streamingContent.value = ''
    
    const read = () => {
      reader.read().then(({ done, value }) => {
        if (done) {
          isStreaming.value = false
          messages.value.push({ role: 'assistant', content: streamingContent.value })
          question.value = ''
          return
        }
        
        const chunk = decoder.decode(value)
        const lines = chunk.split('\n')
        
        lines.forEach(line => {
          if (line.startsWith('data:')) {
            const data = JSON.parse(line.substring(5))
            if (data.type === 'content') {
              streamingContent.value += data.content
            } else if (data.type === 'done') {
              isStreaming.value = false
              messages.value.push({ role: 'assistant', content: streamingContent.value })
              question.value = ''
            }
          }
        })
        
        read()
      })
    }
    
    read()
  })
  .catch(error => {
    console.error('SSE 错误:', error)
    isStreaming.value = false
  })
}
</script>

<style scoped>
.chat-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.messages {
  max-height: 500px;
  overflow-y: auto;
  margin-bottom: 20px;
}

.message {
  padding: 10px 15px;
  margin: 10px 0;
  border-radius: 8px;
  max-width: 80%;
}

.message.user {
  background: #007bff;
  color: white;
  margin-left: auto;
}

.message.assistant {
  background: #f0f0f0;
  color: #333;
}

.streaming .cursor {
  animation: blink 1s infinite;
}

@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0; }
}

.input-area {
  display: flex;
  gap: 10px;
}

.input-area input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.input-area button {
  padding: 10px 20px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.input-area button:disabled {
  background: #ccc;
  cursor: not-allowed;
}
</style>
```

### 4.4 SSE 常见问题排查

| 问题 | 症状 | 原因 | 解决方案 | 优先级 |
|------|------|------|----------|--------|
| 连接立即断开 | 收不到任何数据 | 响应类型错误 | 设置 `produces = TEXT_EVENT_STREAM_VALUE` | 🔴 高 |
| 数据不显示 | 前端收不到数据 | 数据格式错误 | 使用 `data:` 前缀 | 🔴 高 |
| 中文乱码 | 显示乱码 | 编码问题 | 设置 UTF-8 编码 | 🟡 中 |
| 连接超时 | 300 秒后断开 | 超时设置过短 | 增加 `SseEmitter` 超时时间 | 🟡 中 |
| 内存泄漏 | 服务器内存增长 | 连接未关闭 | 实现 `onCompletion` 回调 | 🔴 高 |

---

## 第 5 章 向量存储服务集成

### 5.1 VectorStore 接口详解

```java
// Spring AI VectorStore 核心接口
public interface VectorStore {
    
    // 添加文档
    void add(List<Document> documents);
    
    // 相似性检索
    List<Document> similaritySearch(String query);
    
    // 带参数的相似性检索
    List<Document> similaritySearch(SearchRequest request);
    
    // 删除文档
    boolean delete(List<String> idList);
}
```

### 5.2 自定义 VectorStore 实现

```java
package com.example.rag.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class CustomVectorStoreService {
    
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    
    /**
     * 批量添加文档（带重试机制）
     */
    public void addDocumentsBatch(List<Document> documents, int batchSize) {
        for (int i = 0; i < documents.size(); i += batchSize) {
            int end = Math.min(i + batchSize, documents.size());
            List<Document> batch = documents.subList(i, end);
            
            try {
                vectorStore.add(batch);
                log.info("✅ 批次 {}/{} 完成", end, documents.size());
            } catch (Exception e) {
                log.error("❌ 批次 {} 失败", i, e);
                // 重试逻辑
                retryAdd(batch, 3);
            }
        }
    }
    
    /**
     * 高级检索（带过滤条件）
     */
    public List<Document> advancedSearch(String query, 
                                         Map<String, Object> filterCriteria,
                                         int topK,
                                         double similarityThreshold) {
        SearchRequest request = SearchRequest.query(query)
            .withTopK(topK)
            .withSimilarityThreshold(similarityThreshold);
        
        // 添加过滤条件
        if (filterCriteria != null && !filterCriteria.isEmpty()) {
            request = request.withFilterCriteria(filterCriteria);
        }
        
        return vectorStore.similaritySearch(request);
    }
    
    /**
     * 删除指定条件的文档
     */
    public void deleteByMetadata(String key, Object value) {
        List<Document> docs = advancedSearch("*", 
            Map.of(key, value), 1000, 0.0);
        
        List<String> ids = docs.stream()
            .map(Document::getId)
            .collect(Collectors.toList());
        
        if (!ids.isEmpty()) {
            vectorStore.delete(ids);
            log.info("✅ 删除 {} 条文档", ids.size());
        }
    }
    
    /**
     * 重试添加逻辑
     */
    private void retryAdd(List<Document> batch, int maxRetries) {
        for (int i = 0; i < maxRetries; i++) {
            try {
                Thread.sleep(1000 * (i + 1)); // 递增延迟
                vectorStore.add(batch);
                log.info("✅ 重试成功");
                return;
            } catch (Exception e) {
                log.warn("⚠️ 重试 {} 失败", i + 1, e);
            }
        }
        log.error("❌ 重试 {} 次后仍失败", maxRetries);
    }
}
```

### 5.3 文档元数据管理

```java
// Document 元数据示例
Document doc = new Document(
    "政策内容文本...",
    Map.of(
        "doc_id", "POLICY_2025_001",
        "doc_type", "政策文件",
        "department", "发改委",
        "publish_date", "2025-01-15",
        "effective_date", "2025-02-01",
        "region", "全国",
        "keywords", Arrays.asList("人工智能", "产业发展", "扶持政策")
    )
);

// 元数据过滤检索
SearchRequest request = SearchRequest.query("人工智能扶持政策")
    .withTopK(5)
    .withFilterExpression("department == '发改委' AND publish_date > '2024-01-01'");

List<Document> results = vectorStore.similaritySearch(request);
```

---

## 第 6 章 依赖冲突与版本管理

### 6.1 常见依赖冲突场景

| 冲突类型 | 冲突依赖 | 症状 | 解决方案 |
|----------|----------|------|----------|
| Spring 版本冲突 | spring-core | `NoSuchMethodError` | 使用 BOM 统一管理 |
| Netty 版本冲突 | netty-all | `IncompatibleClassChangeError` | 强制指定版本 |
| SLF4J 冲突 | slf4j-simple/slf4j-log4j | `Multiple bindings` | 排除多余实现 |
| Jackson 冲突 | jackson-databind | `JsonMappingException` | 统一 spring-boot 管理 |
| Milvus SDK 冲突 | milvus-sdk-java | `ClassNotFoundException` | 锁定版本 |

### 6.2 依赖树分析命令

```bash
# 查看完整依赖树
mvn dependency:tree

# 查看特定依赖的冲突
mvn dependency:tree -Dincludes=io.milvus:*

# 查看重复依赖
mvn dependency:tree -Dverbose

# 生成依赖报告
mvn dependency:analyze-report
```

### 6.3 依赖冲突解决模板

```xml
<!-- 统一版本管理 -->
<properties>
    <spring-ai.version>1.0.2</spring-ai.version>
    <milvus-sdk.version>2.4.0</milvus-sdk.version>
    <netty.version>4.1.100.Final</netty.version>
    <jackson.version>2.16.1</jackson.version>
</properties>

<dependencyManagement>
    <dependencies>
        <!-- Spring AI BOM -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        
        <!-- 强制指定 Netty 版本 -->
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>${netty.version}</version>
        </dependency>
        
        <!-- 强制指定 Jackson 版本 -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Milvus SDK（排除冲突依赖） -->
    <dependency>
        <groupId>io.milvus</groupId>
        <artifactId>milvus-sdk-java</artifactId>
        <version>${milvus-sdk.version}</version>
        <exclusions>
            <exclusion>
                <groupId>org.slf4j</groupId>
                <artifactId>slf4j-simple</artifactId>
            </exclusion>
            <exclusion>
                <groupId>io.netty</groupId>
                <artifactId>*</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
```

---

## 第 7 章 生产环境部署最佳实践

### 7.1 应用打包与部署

```bash
# 1. 清理并打包
mvn clean package -DskipTests

# 2. 查看生成的 JAR 文件
ls -lh target/*.jar

# 3. 运行应用
java -jar target/rag-system-1.0.0.jar

# 4. 生产环境运行（指定配置文件）
java -jar target/rag-system-1.0.0.jar \
  --spring.profiles.active=prod \
  --server.port=8080 \
  --spring.ai.ollama.base-url=http://ollama:11434 \
  --spring.ai.vectorstore.milvus.host=milvus

# 5. 后台运行
nohup java -jar target/rag-system-1.0.0.jar > app.log 2>&1 &

# 6. 查看日志
tail -f app.log
```

### 7.2 Docker 部署配置

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# 复制 JAR 文件
COPY target/rag-system-1.0.0.jar app.jar

# 创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 暴露端口
EXPOSE 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/api/actuator/health || exit 1

# 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  rag-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_AI_OLLAMA_BASE_URL=http://ollama:11434
      - SPRING_AI_VECTORSTORE_MILVUS_HOST=milvus
    depends_on:
      - ollama
      - milvus
    networks:
      - rag-network
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/api/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ./ollama-data:/root/.ollama
    networks:
      - rag-network

  milvus:
    image: milvusdb/milvus:v2.6.0
    ports:
      - "19530:19530"
    volumes:
      - ./milvus-data:/var/lib/milvus
    networks:
      - rag-network

networks:
  rag-network:
    driver: bridge
```

### 7.3 性能优化配置

```yaml
# application-prod.yml
server:
  tomcat:
    threads:
      max: 200
      min-spare: 20
    connection-timeout: 30000
    accept-count: 100

spring:
  ai:
    ollama:
      chat:
        options:
          temperature: 0.7
          top-p: 0.9
          num-predict: 2048
    vectorstore:
      milvus:
        connection-pool:
          max-total: 20
          max-idle: 10
          min-idle: 5

# JVM 启动参数优化
# -Xms512m -Xmx1g -XX:+UseG1GC -XX:MaxGCPauseMillis=200
```

### 7.4 监控与告警

```java
// 添加 Spring Boot Actuator 依赖
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# 监控端点配置
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
```

---

**本专题完**

> 📌 **核心要点总结**：
> 1. Spring AI 1.0.2 是 2026 年最新稳定版，生产环境推荐使用
> 2. RAG 核心逻辑仅需 30 行代码：检索→组装→生成
> 3. SSE 流式响应需设置 `TEXT_EVENT_STREAM_VALUE` 和超时时间
> 4. 使用 BOM 统一管理 Spring AI 依赖版本，避免冲突
> 5. Milvus SDK 需排除冲突的日志和 Netty 依赖
> 6. 生产环境使用 Docker 部署，配置健康检查和资源限制
> 7. 添加 Actuator 监控端点，便于运维管理

> 📖 **下专题预告**：《Vue3 前端开发实战：打造企业级 RAG 问答界面》—— 详解组件设计、SSE 对接、引用来源展示、响应式布局