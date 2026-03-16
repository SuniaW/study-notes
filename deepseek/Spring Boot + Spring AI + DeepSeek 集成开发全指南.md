

# Spring Boot + Spring AI + DeepSeek 集成开发全指南

## 1. 概述
DeepSeek 提供了高性能且具有极高性价比的大模型能力。由于 DeepSeek 的 API **完全兼容 OpenAI 接口规范**，Java 开发者可以利用 **Spring AI** 框架实现极简集成，无需手动编写复杂的 HTTP 请求。

### 核心价值
*   **低成本切换**：通过 Spring AI 的抽象，只需修改配置即可在 OpenAI、DeepSeek、阿里云通义千问等模型间无缝切换。
*   **企业级能力**：支持结构化输出、RAG（知识库增强）以及 Function Calling（工具调用）。

---

## 2. 环境与依赖准备

### 2.1 版本要求
*   **JDK**: 17 或以上
*   **Spring Boot**: 3.2.x 或 3.3.x
*   **Spring AI**: 推荐使用 **1.0.0-M1** 及以上版本（旧版 API 已废弃）

### 2.2 Maven 依赖配置
在 `pom.xml` 中添加 Spring AI 的 OpenAI Starter。由于 Spring AI 处于 Milestone 阶段，必须配置 Spring 官方仓库。

```xml
<dependencies>
    <!-- Spring AI OpenAI Starter (兼容 DeepSeek) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>
```

---

## 3. 核心配置 (application.yml)

在配置文件中将 `base-url` 指向 DeepSeek 官方接口。

```yaml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY} # 替换为你的 DeepSeek Key
      base-url: https://api.deepseek.com
      chat:
        options:
          # deepseek-chat (V3): 速度快，支持 Function Calling
          # deepseek-reasoner (R1): 深度推理，暂不支持 Function Calling
          model: deepseek-chat
          temperature: 0.7
```

---

## 4. 基础对话实现

### 4.1 推荐使用 ChatClient API
在最新版 Spring AI 中，推荐通过 `ChatClient` 进行流式调用，其 API 更加直观。

**配置类：**
```java
@Configuration
public class AiConfig {
    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder.build();
    }
}
```

**Controller 调用：**
```java
@RestController
public class ChatController {
    private final ChatClient chatClient;

    public ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // 简单文本回复
    @GetMapping("/chat")
    public String chat(String message) {
        return chatClient.prompt(message).call().content();
    }

    // 流式回复 (SSE)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream(String message) {
        return chatClient.prompt(message).stream().content();
    }
}
```

---

## 5. 进阶核心：Function Calling (函数调用)

### 5.1 什么是 Function Calling？
Function Calling 是让 AI 调用你本地 Java 方法的能力。AI 会分析用户意图，如果发现需要获取实时数据（如查物流、调数据库），它会要求程序执行对应的函数，并将执行结果交给 AI 总结。

### 5.2 实现步骤
1. **定义函数 Bean**：使用 `@Description` 描述函数用途。
2. **在请求中申明函数**：告知 AI 可用的工具名。

**代码示例：**
```java
@Configuration
public class ToolConfig {

    // 1. 定义业务函数
    @Bean
    @Description("根据订单号获取物流状态")
    public Function<OrderRequest, OrderResponse> getOrderStatus() {
        return request -> {
            // 模拟业务查询
            return new OrderResponse("订单 [" + request.orderId() + "] 已到达上海分拨中心");
        };
    }

    public record OrderRequest(String orderId) {}
    public record OrderResponse(String status) {}
}

// 2. 在调用时启用
@GetMapping("/order-chat")
public String orderChat(String message) {
    return chatClient.prompt(message)
            .functions("getOrderStatus") // 传入定义的 Bean 名称
            .call()
            .content();
}
```

---

## 6. Java AI 框架横向对比

| 框架 | 适用场景 | 特点 |
| :--- | :--- | :--- |
| **Spring AI** | Spring 生态企业级应用 | 官方背景，统一 API，支持 RAG 和函数调用，适合快速集成。 |
| **LangChain4j** | 复杂 AI Agent 构建 | 功能最强（类似于 Python 的 LangChain），集成广泛，支持 50+ 供应商。 |
| **DJL (Deep Java Library)** | 深度学习推理 | 亚马逊开源，在 JVM 上运行 PyTorch/TensorFlow 模型。 |
| **DeepLearning4j** | 分布式训练 | 适合在 Hadoop/Spark 环境下进行大规模神经网络训练。 |

---

## 7. 常见问题排查 (Troubleshooting)

### Q1: 提示 `No candidates found for method call ...`
*   **原因**：通常是因为使用了旧版 `ChatModel` 或 Spring AI 版本低于 `1.0.0-M1`。
*   **解决**：
    1. 检查版本是否为 `1.0.0-M1`。
    2. 检查 Import 是否为 `org.springframework.ai.chat.client.ChatClient`。
    3. 确保是通过 `ChatClient.Builder` 注入的。

### Q2: 启用 Function Calling 后 AI 报错或无响应
*   **原因**：使用了 `deepseek-reasoner` (R1) 模型。
*   **解决**：DeepSeek 官方目前明确 R1 推理模型不支持函数调用。请在配置中切换为 `deepseek-chat`。

### Q3: 接口响应时间太长
*   **原因**：AI 生成文本较慢，HTTP 阻塞。
*   **解决**：
    1. 增加超时时间：`spring.mvc.async.request-timeout=60000`。
    2. 使用 `stream()` 方式进行响应式输出。

---
**文档说明**：本指南基于 Spring AI 2024/2025 最新趋势编写，建议持续关注 Spring AI 官方文档的更新。