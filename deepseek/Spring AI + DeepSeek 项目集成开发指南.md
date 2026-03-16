
# Spring AI + DeepSeek 项目集成开发指南

本手册涵盖了从项目初始化、实时流式对话、时效性增强到 Function Calling（工具调用）的完整实现。

## 1. 项目准备

### 1.1 核心依赖 (Maven)
在 `pom.xml` 中引入 Spring AI BOM 和 OpenAI Starter（DeepSeek 兼容 OpenAI 协议）。

```xml
<dependencies>
    <!-- Web Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Spring AI OpenAI Starter -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <!-- 响应式编程支持 -->
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-core</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0-M5</version> <!-- 建议版本 -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 1.2 核心配置 (application.yml)
```yaml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat # V3模型支持工具调用；deepseek-reasoner为R1模型
```

---

## 2. 核心功能实现

### 2.1 实时流式响应 (SSE)
**场景**：解决 AI 回答缓慢、用户等待时间长的问题，实现打字机效果。
**关键点**：使用 `Flux<String>` 配合 `MediaType.TEXT_EVENT_STREAM_VALUE`。

```java
@GetMapping(value = "/ai/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String message) {
    return chatClient.prompt()
            .user(message)
            .stream() // 开启流式响应
            .content();
}
```

### 2.2 解决日期/时效性问题
**原因**：AI 的知识截止于训练时刻，无法感知当前系统时间。
**方案**：在 `System Prompt` 中动态注入服务器当前日期。

```java
@GetMapping("/ai/chat")
public String chat(@RequestParam String message) {
    return chatClient.prompt()
            .system(s -> s.text("当前日期是 {date}").param("date", LocalDate.now().toString()))
            .user(message)
            .call()
            .content();
}
```

---

## 3. Function Calling (函数调用)

允许 AI 调用你本地的 Java 业务逻辑（如查询数据库、物流信息等）。

### 3.1 定义业务函数
```java
@Configuration
public class AiToolsConfig {

    public record OrderRequest(String orderId) {}
    public record OrderResponse(String status) {}

    @Bean
    @Description("根据订单ID查询订单状态") // 描述非常重要，AI靠此判断是否调用
    public Function<OrderRequest, OrderResponse> queryOrderService() {
        return request -> {
            // 业务逻辑：例如数据库查询
            return new OrderResponse("订单 " + request.orderId() + " 已发货");
        };
    }
}
```

### 3.2 调用方式及报错处理
如果在调用 `.functions()` 时遇到 `Cannot resolve method` 报错，请参考以下三种方案：

#### 方案 A：在 Builder 中全局注册（推荐）
```java
public ChatController(ChatClient.Builder builder) {
    this.chatClient = builder
            .defaultFunctions("queryOrderService") // 全局默认开启
            .build();
}
```

#### 方案 B：通过 Options 显式调用（最稳定）
```java
import org.springframework.ai.openai.OpenAiChatOptions;

@GetMapping("/ai/chat")
public String chat(@RequestParam String message) {
    return chatClient.prompt()
            .user(message)
            .options(OpenAiChatOptions.builder()
                    .withFunction("queryOrderService")
                    .build())
            .call()
            .content();
}
```

---

## 4. 常见问题排查 (FAQ)

| 问题描述 | 原因分析 | 解决方案 |
| :--- | :--- | :--- |
| **返回日期是 2024 年** | AI 知识库截止日期限制 | 在 System Prompt 中注入 `LocalDate.now()` |
| **无法识别 .functions()** | Spring AI 版本 API 变动 | 使用 `ChatClient.Builder` 注册或通过 `Options` 传入 |
| **R1 模型无法调用函数** | DeepSeek R1 协议限制 | 函数调用建议切换至 `deepseek-chat` 模型 |
| **响应结果乱码** | HTTP Header 未设置正确的 Content-Type | 确保 produces 设置为 `text/event-stream` |

## 5. 开发建议
1. **模型选择**：普通对话和需要 Function Calling 的场景使用 `deepseek-chat`；需要深度逻辑推理的场景使用 `deepseek-reasoner`。
2. **超时设置**：AI 响应较慢，建议将 HTTP 客户端的超时时间设置为 60s 以上。
3. **安全提示**：不要将 API Key 硬编码在代码中，建议使用环境变量 `${DEEPSEEK_API_KEY}` 注入。