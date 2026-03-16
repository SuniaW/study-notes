
# Spring AI 实践指南：从脚本模式到生产级架构

## 1. 依赖与环境配置
由于 Spring AI 目前处于快速迭代阶段（Milestone/RC 阶段），请确保 `pom.xml` 中配置了正确的版本及 Maven 仓库。

### 1.1 依赖管理 (BOM)
建议使用 BOM 来管理版本，避免版本冲突。
> **注意：** 目前稳定里程碑版本为 `1.0.0-M5` 或 `1.0.0-RC1`，请勿使用不存在的 `2.0.0-M2`。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0-M5</version>
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

---

## 2. 架构重构：从 CommandLineRunner 到上下文模式
在开发初期，我们常在 `CommandLineRunner` 中编写逻辑。但在生产环境中，应将 `ChatClient` 声明为 Spring Bean，以便在 Controller 或 Service 中复用。

### 2.1 核心配置类 (AiConfig)
通过 `ChatClient.Builder` 创建不同配置的 Bean。

```java
@Configuration
public class AiConfig {

    /**
     * 通用助手：基础配置
     */
    @Bean
    @Primary // 设置为默认注入项
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("You are a helpful assistant.")
                .build();
    }

    /**
     * 气象专家助手：集成工具、评估顾问和日志记录
     */
    @Bean
    public ChatClient weatherChatClient(
            ChatClient.Builder builder, 
            ChatModel chatModel) { // 注入具体的模型实现
        
        return builder
                .defaultSystem("你是一个专业的气象助手。")
                .defaultTools(new WeatherTool()) // 注册天气函数工具
                .defaultAdvisors(
                        // 自我修正评估：使用 chatModel 进行结果校验
                        SelfRefineEvaluationAdvisor.builder()
                                .chatClientBuilder(ChatClient.builder(chatModel))
                                .maxRepeatAttempts(15)
                                .successRating(4)
                                .order(0)
                                .build(),
                        // 日志增强
                        new MyLoggingAdvisor(2)
                )
                .build();
    }
}
```

---

## 3. 多 Bean 注入与冲突处理
当 Spring 上下文中存在多个同类型（`ChatClient`）的 Bean 时，必须告知 Spring 如何选择。

### 3.1 方案一：使用 `@Qualifier` (推荐)
通过指定 Bean 的名称（默认为方法名）进行注入，这是最安全、最明确的方式。

```java
@Service
public class WeatherService {
    private final ChatClient weatherClient;

    public WeatherService(@Qualifier("weatherChatClient") ChatClient weatherClient) {
        this.weatherClient = weatherClient;
    }

    public String askWeather(String city) {
        return weatherClient.prompt()
                .user("告诉我 " + city + " 的天气")
                .call()
                .content();
    }
}
```

### 3.2 方案二：使用 `@Primary`
在配置类中，给最常用的 Bean 加上 `@Primary` 注解。当注入时不加 `@Qualifier` 时，Spring 会自动选择该 Bean。

### 3.3 方案三：参数名自动匹配
Spring 具备一种隐式能力：如果参数名（如 `ChatClient weatherChatClient`）与 Bean 的 ID 一致，它会自动匹配。
*注：该方式受编译参数影响，建议仅在小型项目或快速原型中使用。*

---

## 4. 最佳实践总结

1.  **利用 Builder 的 Prototype 特性**：
    Spring Boot 自动配置提供的 `ChatClient.Builder` 是 **多例 (Prototype)** 的。这意味着每次注入或调用它都会得到一个全新的 Builder 实例，不会污染其他 Client 的配置。

2.  **组件化 Advisors**：
    *   **通用逻辑**（如日志、耗时统计）：应放在通用的 `ChatClient` 中。
    *   **业务逻辑**（如 `SelfRefineEvaluationAdvisor`）：应仅配置在特定的业务 Bean（如 `weatherChatClient`）中。

3.  **解耦 Tool 定义**：
    建议将 `WeatherTool` 定义为 Spring Bean 或使用 `@Tool` 注解的类，而不是在配置类里直接 `new`，这样更方便管理 Tool 的内部依赖（如 API Key 等）。

4.  **环境隔离**：
    在 `weatherChatClient` 中，可以使用本地的 `Ollama` 模型作为评估器（Advisor），而使用昂贵的远程模型（如 Anthropic/OpenAI）作为主生成器，以平衡成本与质量。