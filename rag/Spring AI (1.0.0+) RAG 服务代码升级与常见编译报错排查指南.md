

# 📚 Spring AI (1.0.0+) RAG 服务代码升级与常见编译报错排查指南

## 📝 背景说明
随着 Spring AI 的快速迭代（尤其是进入 `1.0.0 Milestone` 系列版本后），部分核心 API 发生了破坏性更新。如果在参考早期教程或旧版代码实现 RAG（检索增强生成）功能时，常会遇到一系列编译错误。

本文档汇总了从旧版本向新版本迁移时最常见的报错及其解决方案，并提供了一份经过优化的完整 RAG 服务代码。

---

## 🐞 常见报错分析与解决方案

### 报错一：`SearchRequest` 找不到 `query` 方法
> **错误信息：** `Cannot resolve method 'query' in 'SearchRequest'`

* **原因分析：** 新版 Spring AI 移除了 `SearchRequest.query()` 这种静态工厂方法，全面统一为标准的 **Builder 模式**。
* **修复方法：**
  ```java
  // ❌ 旧版写法：
  SearchRequest.query(query).withTopK(3).withSimilarityThreshold(0.6)
  
  // ✅ 新版写法：
  SearchRequest.builder().query(query).topK(3).similarityThreshold(0.6).build()
  ```

### 报错二：`Document` 找不到 `getMetadata` 方法
> **错误信息：** `Cannot resolve method 'getMetadata' in 'Document'`

* **原因分析：** 90% 的情况是因为 IDE 的自动导包（Auto-Import）引错了类。通常错误地引入了 `org.w3c.dom.Document` 或 `org.bson.Document`。
* **修复方法：** 检查顶部 `import`，确保引入的是 Spring AI 的 `Document` 类。
  ```java
  // ✅ 正确导包：
  import org.springframework.ai.document.Document;
  ```

### 报错三：`Document` 找不到 `getContent` 方法
> **错误信息：** `Cannot resolve method 'getContent'`

* **原因分析：** 在 Spring AI 1.0.0 M2/M3 以后的版本中，`Document` 类表示文本内容的属性名由 `content` 被重命名为了 `text`。
* **修复方法：** 将原本的获取内容方法替换为 `getText()`。
  ```java
  // ❌ 旧版写法：
  docs.stream().map(Document::getContent)
  
  // ✅ 新版写法：
  docs.stream().map(Document::getText)
  ```

---

## 💡 最佳实践与优化点

1. **`ChatClient` 的复用**：不建议在每次方法调用时都执行 `chatClientBuilder.build()`。推荐在类的**构造函数**中 Build 一次，随后复用该实例，以提升性能。
2. **元数据空指针安全**：在获取 `Metadata`（如文件名 `filename`）时，推荐使用 `getOrDefault()`。防止某些文档入库时缺少该元数据导致拼接出 `null` 字符串或强转报错。
3. **流式 API 语法更新**：新版 ChatClient 推荐使用 `.prompt().user(prompt)` 的链式调用语法。

---

## 💻 完整可运行代码 (适配 Spring AI 1.0.0+)

以下是经过修复和优化后的完整 `RagService` 代码：

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

    /**
     * 推荐在构造函数中完成 ChatClient 的构建，避免每次请求重复 build
     */
    public RagService(ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {
        this.chatClient = chatClientBuilder.build();
        this.vectorStore = vectorStore;
    }

    /**
     * 流式响应的 RAG 问答服务
     *
     * @param query 用户提问
     * @return Flux<String> 响应流 (SSE)
     */
    public Flux<String> streamAnswer(String query) {

        // 1. 向量检索：使用 Builder 模式，获取 Top 3 且相似度大于 0.6 的文档
        List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(query)
                        .topK(3)
                        .similarityThreshold(0.6)
                        .build()
        );

        // 2. 组装上下文：注意新版 API 使用 getText() 获取文本
        String context = docs.stream()
                .map(Document::getText) 
                .collect(Collectors.joining("\n\n"));

        // 3. 组装来源参考：使用 getOrDefault 防止 null 异常
        String refs = docs.stream()
                .map(d -> (String) d.getMetadata().getOrDefault("filename", "未知来源"))
                .distinct()
                .collect(Collectors.joining(", "));

        // 4. 构建 Prompt 提示词
        String prompt = "基于以下政策上下文回答问题：\n" + context + "\n\n问题：" + query;

        // 5. 调用大模型并流式返回结果，末尾拼接参考来源
        return chatClient.prompt()
                .user(prompt)
                .stream()
                .content()
                .concatWith(Flux.just("\n\n---\n📚 来源: " + refs));
    }
}
```