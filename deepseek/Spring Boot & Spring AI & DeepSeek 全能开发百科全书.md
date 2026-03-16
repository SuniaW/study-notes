

# Spring Boot & Spring AI & DeepSeek 全能开发百科全书

本指南旨在帮助开发者利用 Spring AI 框架，将 DeepSeek 大模型无缝集成到 Spring Boot 项目中，涵盖从基础对话到生产级架构的完整路径。

---

## 第一部分：核心背景与版本选型

### 1. 为什么选择这个组合？
*   **DeepSeek**：提供极高性价比、且与 OpenAI API **完全兼容**的模型（V3/R1）。
*   **Spring AI**：统一了不同 AI 供应商的 API，避免供应商锁定，支持结构化输出、RAG 和函数调用。
*   **生产级架构**：从简单的脚本调用转向基于 Spring Bean 的可维护架构。

### 2. 关键版本兼容性（避坑指南）
| 组件 | 推荐版本 | 最低要求/注意事项 |
| :--- | :--- | :--- |
| **JDK** | 17 或 21 | 建议使用 LTS 版本 |
| **Spring Boot** | **3.2.5+** 或 **3.4.x** | **必须 >= 3.2.0**（否则会报 `RestClient` 缺失错误） |
| **Spring AI** | **1.0.0-M5** | 建议使用 Milestone 5，稳定性优于 M1 |
| **DeepSeek API** | - | 需从 [DeepSeek 开放平台](https://platform.deepseek.com/) 获取 Key |

---

## 第二部分：项目初始化与配置

### 1. Maven 依赖管理 (`pom.xml`)
必须配置 Spring Milestone 仓库，因为 Spring AI 尚未发布到 Maven 中央仓库。

```xml
<properties>
    <java.version>21</java.version>
    <spring-ai.version>1.0.0-M5</spring-ai.version>
</properties>

<dependencies>
    <!-- 核心 Web 支持 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- OpenAI Starter (DeepSeek 兼容此协议) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
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
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>
```

### 2. 核心配置 (`application.yml`)
```yaml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      chat:
        options:
          # deepseek-chat: V3模型，支持函数调用，适合常规任务
          # deepseek-reasoner: R1模型，擅长逻辑推理，暂不支持函数调用
          model: deepseek-chat
          temperature: 0.7
```

---

## 第三部分：核心功能实现

### 1. 基础对话与流式输出 (SSE)
推荐使用 `ChatClient` 流畅 API 替代旧版的 `ChatModel`。

```java
@RestController
public class ChatController {
    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        // 在 Builder 中可以配置默认系统提示词
        this.chatClient = builder
                .defaultSystem("你是一个专业的技术顾问。")
                .build();
    }

    // 1. 普通阻塞式对话
    @GetMapping("/ai/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt(message).call().content();
    }

    // 2. 流式响应 (SSE)，实现打字机效果
    @GetMapping(value = "/ai/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String message) {
        return chatClient.prompt(message)
                .stream()
                .content();
    }
}
```

### 2. 解决时效性问题
AI 的知识有截止日期。可以通过 `System Prompt` 动态注入服务器时间。

```java
public String chatWithDate(String message) {
    return chatClient.prompt()
            .system(s -> s.text("当前日期是 {date}").param("date", LocalDate.now().toString()))
            .user(message)
            .call()
            .content();
}
```

---

## 第四部分：高级进阶——Function Calling (函数调用)

允许 AI 调用你本地的 Java 业务逻辑（如查数据库、调 API）。

### 1. 定义函数 Bean
必须使用 `@Description` 描述函数，这是 AI 判断是否调用的唯一依据。

```java
@Configuration
public class ToolConfig {
    public record OrderRequest(String orderId) {}
    public record OrderResponse(String status) {}

    @Bean
    @Description("根据订单ID查询订单的实时物流状态") 
    public Function<OrderRequest, OrderResponse> getOrderStatus() {
        return request -> new OrderResponse("订单 [" + request.orderId() + "] 正在上海分拨中心派送中...");
    }
}
```

### 2. 开启函数调用
**注意**：必须使用 `deepseek-chat` 模型。

```java
@GetMapping("/ai/order")
public String orderChat(@RequestParam String message) {
    return chatClient.prompt(message)
            .functions("getOrderStatus") // 这里的名字必须与 @Bean 方法名一致
            .call()
            .content();
}
```

---

## 第五部分：生产级架构与 Advisor 模式

### 1. 多模型 Bean 管理
当项目中有多个用途的 AI 客户端时，使用 `@Qualifier` 区分。

```java
@Configuration
public class AiConfig {
    @Bean
    @Primary
    public ChatClient defaultAssistant(ChatClient.Builder builder) {
        return builder.defaultSystem("普通助手").build();
    }

    @Bean
    public ChatClient expertAssistant(ChatClient.Builder builder) {
        return builder.defaultSystem("专家助手").build();
    }
}
```

### 2. 自我修正评估器 (SelfRefineEvaluationAdvisor)
这是一种 **"LLM-as-a-Judge"** 模式，通过反馈循环提升回答质量。

*   **原理**：模型生成回答 -> 裁判模型评分 -> 若评分低则带上反馈重试 -> 直到达标或达到上限。
*   **适用场景**：公文写作、复杂逻辑校验。

```java
@Bean
public ChatClient refinedClient(ChatClient.Builder builder, ChatModel judgeModel) {
    return builder
        .defaultAdvisors(
            SelfRefineEvaluationAdvisor.builder()
                .chatClientBuilder(ChatClient.builder(judgeModel)) // 指定裁判模型
                .successRating(4) // 满分4分，要求达到4分才停止重试
                .maxRepeatAttempts(3)
                .build()
        )
        .build();
}
```

---

## 第六部分：故障排查 (Troubleshooting)

### 1. 启动报错：`RestClientAutoConfiguration not present`
*   **根本原因**：Spring AI 1.0.0-M1+ 使用了 `RestClient`，这需要 **Spring Boot 3.2.0+**。
*   **解决**：检查 `pom.xml` 中的 `spring-boot-starter-parent` 版本，确保 `>= 3.2.5`。

### 2. 依赖找不到：`Dependency 'org.springframework.ai:...' not found`
*   **根本原因**：Maven 默认不从 Milestone 仓库下载。
*   **解决**：检查是否配置了 `<repositories>` 中的 `spring-milestones` 地址，并确保没有写错版本号（如写成 2.0.0 或 4.0.0）。

### 3. Function Calling 无效/报错
*   **检查 1**：确保模型是 `deepseek-chat` 而非 `deepseek-reasoner`。
*   **检查 2**：确保 Bean 上有 `@Description`。
*   **检查 3**：Bean 名称与 `.functions("name")` 必须完全匹配。

### 4. 响应乱码或超时
*   **解决**：
    *   流式输出确保设置 `produces = MediaType.TEXT_EVENT_STREAM_VALUE`。
    *   在 `application.yml` 增加异步超时：`spring.mvc.async.request-timeout=60000`。

---

## 第七部分：附录——开发者小贴士

### 1. GitHub 仓库清理
如果您需要删除废弃的 AI 实验仓库：
*   **网页端**：`Settings` -> `Danger Zone` -> `Delete this repository`。
*   **命令行**：使用 GitHub CLI 执行 `gh repo delete 用户名/仓库名 --confirm`。

### 2. 模型对比
*   **DeepSeek-V3**：全能型，适合对话、代码、Function Calling。
*   **DeepSeek-R1**：推理型，适合数学、复杂逻辑预测，不支持工具调用。

