# Spring Boot 与 DeepSeek 集成开发指南

本指南介绍了如何利用 Spring AI 框架，将 DeepSeek 大模型无缝集成到 Spring Boot 项目中，并实现高级功能（如 Function Calling）。

---

## 一、 背景与优势
DeepSeek 的 API 接口与 **OpenAI 完全兼容**。这意味着我们可以直接使用 Spring AI 官方提供的 OpenAI Starter，只需更改 `base-url` 和 `model` 即可快速接入。

- **框架选择**：推荐使用 **Spring AI** (1.0.0-M1 及以上版本)。
- **核心模型**：
    - `deepseek-chat`：适用于对话、代码生成、逻辑推理（支持 Function Calling）。
    - `deepseek-reasoner`：即 R1 模型，擅长深度思考、数学和复杂推理。

---

## 二、 环境准备

1. **JDK 版本**：Java 17 或更高版本。
2. **Spring Boot**：3.2.x 或 3.3.x。
3. **DeepSeek API Key**：前往 [DeepSeek 开放平台](https://platform.deepseek.com/) 获取。

---

## 三、 项目配置

### 1. Maven 依赖
在 `pom.xml` 中引入 Spring AI OpenAI Starter。注意：Spring AI 处于 Milestone 阶段，需要配置 Spring 仓库。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
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

### 2. 核心配置 (`application.yml`)
将请求地址指向 DeepSeek 官方 API。

```yaml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat # 常规任务选这个
          # model: deepseek-reasoner # 深度推理选这个
```

---

## 四、 核心集成方案

### 1. 基础对话集成 (使用 ChatClient)
`ChatClient` 是 Spring AI 1.0.0-M1 推出的流畅 API，推荐优先使用。

**配置类定义：**
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

    @GetMapping("/ai/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt(message).call().content();
    }
}
```

---

## 五、 进阶：Function Calling (函数调用)

Function Calling 允许 AI 根据需要主动调用本地的 Java 方法。例如：让 AI 查询实时订单物流。

### 第一步：定义函数 Bean
通过 `@Description` 告知 AI 该函数的作用和参数含义。

```java
@Configuration
public class OrderToolsConfig {

    public record OrderRequest(String orderId) {}
    public record OrderResponse(String status) {}

    @Bean
    @Description("根据订单ID查询订单的实时物流状态") 
    public Function<OrderRequest, OrderResponse> getOrderStatus() {
        return request -> {
            // 模拟数据库查询逻辑
            return new OrderResponse("订单 [" + request.orderId() + "] 正在派送中...");
        };
    }
}
```

### 第二步：在对话中启用函数
在 `ChatClient` 中通过 `.functions()` 方法指定 Bean 名称。

```java
@GetMapping("/ai/order-chat")
public String orderChat(@RequestParam String message) {
    return chatClient.prompt(message)
            .functions("getOrderStatus") // 开启函数调用
            .call()
            .content();
}
```

---

## 六、 常见问题与解决方案 (Troubleshooting)

### 1. 报错：`No candidates found for method call ...`
- **原因**：通常是 Spring AI 版本不匹配。
- **解决**：确保版本为 `1.0.0-M1`。检查 Import 路径是否为 `org.springframework.ai.chat.client.ChatClient`。

### 2. 无法使用 Function Calling
- **原因**：模型选择错误。
- **解决**：DeepSeek 的 `deepseek-reasoner` (R1) 目前**不支持** Function Calling。请务必在配置文件中切换回 `deepseek-chat`。

### 3. 连接超时
- **原因**：网络延迟或 AI 响应耗时较长。
- **解决**：
    - 在 Spring 中增加异步超时时间：`spring.mvc.async.request-timeout=60000`。
    - 使用流式输出（Streaming）：`chatClient.prompt(message).stream().content()`。

---

## 七、 总结：为什么要选这个组合？

| 组件 | 作用 |
| :--- | :--- |
| **Spring Boot** | 提供稳定的企业级后端支撑。 |
| **Spring AI** | 统一了不同 AI 供应商的 API，便于未来从 DeepSeek 切换到其他模型（如国产通义千问等）。 |
| **DeepSeek** | 提供极高性价比、且兼容 OpenAI 协议的大模型能力。 |
| **Function Calling** | 打破“信息茧房”，让 AI 真正连接并操作你的业务数据库和 API。 |