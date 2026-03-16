## 💎 Spring AI 最新版本概览
截至 **2026 年 2 月**，Spring AI 的最新里程碑版本特性如下：

### ⚙️ 技术栈要求
*   **Java 21**：强制要求，不再支持 Java 17（利用虚拟线程提升并发）。
*   **Spring Boot 4.0.1**：基于最新的微服务基座。
*   **Spring Framework 7.0**：核心框架深度集成。

### ✨ 主要更新亮点
*   **API 空安全**：引入 `JSpecify` 规范，大幅提升 API 健壮性。
*   **存储扩展**：新增对 **Amazon S3**、**Infinispan** 等向量存储的支持。
*   **缓存增强**：支持基于 **Redis** 的语义缓存，显著降低重复查询成本。
*   **模型能力升级**：
    *   Mistral AI 与 Ollama 支持 **结构化输出**（JSON Schema）。
    *   **工具调用**（Tool Calling）机制全面增强。

---

## 🚀 Spring AI 2.x 有哪些重大升级？
Spring AI 2.x 是一次面向 AI 原生时代的架构重构，旨在解决企业级 AI 应用的性能、成本与复杂度问题。

### 1. 底层架构全面升级：拥抱 Java 21
| 项目 | 说明 |
| :--- | :--- |
| **最低 JDK** | Java 21（虚拟线程、Pattern Matching 等特性可用） |
| **生态基座** | Spring Boot 4.0 + Spring Framework 7.0 |
| **性能红利** | 利用 **虚拟线程** 提升高并发 LLM 调用效率；支持 **AOT 编译** 加速启动 |

### 2. 🗄️ Redis “史诗级”增强：从缓存到智能记忆引擎
*   **Redis Chat Memory**：跨会话上下文持久化，支持全文检索。
*   **Redis Vector Store**：
    *   新增文本搜索 + 范围查询。
    *   暴露 **HNSW 索引参数**（如 M, efConstruction），可精细调优召回率 vs 延迟。
*   **语义缓存顾问**：自动识别语义相似请求，复用结果，节省 API 调用成本。

### 3. 🤖 模型能力与工具调用全面进化
| 厂商 | 升级亮点 |
| :--- | :--- |
| **OpenAI** | 集成官方 Java SDK；默认模型升级为 `gpt-5-mini` |
| **Anthropic Claude** | 支持 4.5 版本：Citations API（回答可溯源）、Files API（生成可下载文件） |
| **Google Gemini** | 新增 `ThinkingConfig`，动态调节“思考深度” |
| **Mistral / Ollama** | 原生支持结构化 JSON 输出，确保类型安全 |

> **工具调用增强**：支持运行时动态修改工具 Schema；新增 `conversationHistoryEnabled` 选项，精细控制上下文。

### 4. 🛡️ API 设计规范与空安全
*   全面采用 **JSpecify** 空安全规范，编译期即可检测空指针风险。
*   **Kotlin 开发者受益**：获得真正的可空/非空类型支持。

### 5. ⚠️ 破坏性变更（迁移注意）
*   **temperature 配置**：移除默认值，**必须显式配置**，否则报错。
*   **默认模型**：OpenAI 从 `gpt-4` → `gpt-5-mini`，行为可能变化。
*   **注解体系**：`@Function` 演进为 `@Tool`。

---

<a name="实战示例"></a>
## 🛠️ 实战示例：Spring AI 2.x + DeepSeek + Tool + Advisor

### 1. Maven 依赖配置 (`pom.xml`)
```xml
<properties>
    <java.version>21</java.version>
    <spring-ai.version>2.0.0-M2</spring-ai.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
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
```

### 2. 工具调用示例（`@Tool` 替代 `@Function`）

#### 定义工具类
```java
@Component
public class WeatherTool {
    private static final Logger LOGGER = LoggerFactory.getLogger(WeatherTool.class);
    final int[] temperatures = {-125, 15, -255};
    private final Random random = new Random();

    @Tool(description = "Get the current weather for a given location")
    public String weather(String location) {
        LOGGER.info("WeatherTool called with location: {}", location);
        int temperature = temperatures[random.nextInt(temperatures.length)];
        return "The current weather in " + location + " is sunny with a temperature of " + temperature + "°C.";
    }
}
```

#### Tool 注册与 ChatClient 配置
```java
@Configuration
public class AiConfig {
    @Bean
    @Primary
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder.defaultSystem("You are a helpful assistant.").defaultAdvisors().build();
    }

    @Bean
    public ChatClient weatherChatClient(ChatClient.Builder builder, ChatModel chatModel) {
        return builder.defaultSystem("你是一个专业的气象助手。")
                .defaultTools(new WeatherTool()) // 注册工具类
                .defaultAdvisors(
                        // 1. 自我修正评估顾问
                        SelfRefineEvaluationAdvisor.builder()
                            .chatClientBuilder(ChatClient.builder(chatModel))
                            .maxRepeatAttempts(15).successRating(4).order(0).build(),
                        // 2. 自定义日志顾问
                        new MyLoggingAdvisor(2))
                .build();
    }
}
```

### 3. 业务层调用
```java
@Service
public class WeatherService {
    private final ChatClient genericClient;
    private final ChatClient weatherClient;

    public WeatherService(@Qualifier("chatClient") ChatClient genericClient,
                          @Qualifier("weatherChatClient") ChatClient weatherClient) {
        this.genericClient = genericClient;
        this.weatherClient = weatherClient;
    }

    public String doWork(String city) {
        String common = genericClient.prompt("你好").call().content();
        String weather = this.weatherClient
                .prompt("请查询 " + city + " 的天气，并友好地回复用户。")
                .call().content();
        return common + weather;
    }
}
```

---

<a name="advisor-核心机制"></a>
## 🧠 Advisor 机制详解

### ✅ 核心价值
*   **无侵入性**：业务代码无需感知。
*   **解耦**：将日志、记忆、RAG、安全等横切关注点抽离。
*   **责任链模式**：支持可插拔、可排序。

### 📋 内置 Advisor 类型
| 类型 | 说明 |
| :--- | :--- |
| **MessageChatMemoryAdvisor** | 保留完整消息结构，适合多轮对话 |
| **VectorStoreChatMemoryAdvisor** | 超长历史对话支持（通过向量检索突破 Token 限制） |
| **QuestionAnswerAdvisor** | 自动 RAG 检索增强 |
| **SimpleLoggerAdvisor** | 请求/响应日志记录 |
| **SelfRefineEvaluationAdvisor** | LLM-as-a-Judge 闭环评估 |

### 💡 使用建议
1.  **合理设置 Order**：安全校验（Security）优先，日志（Logging）最后。
2.  **避免过度堆叠**：建议链路长度 ≤ 5 个。
3.  **流式处理**：流式响应中慎用阻塞操作。
4.  **持久化**：生产环境务必使用 Redis/DB 替代 `InMemoryChatMemory`。

---

## ⚙️ 配置文件 (`application.yml`)
```yaml
spring:
  application:
    name: ai-learn
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.deepseek.com  # 👈 关键！对接 DeepSeek 等兼容代理
      chat:
        options:
          model: deepseek-chat
          temperature: 0.7
```

---

## 🔄 迁移与识别机制

### 1. 迁移注意事项
| 旧版 (1.x) | 新版 (2.x) |
| :--- | :--- |
| `@Function` | **`@Tool`** |
| `FunctionCallback` | **废弃**，推荐直接使用注解方法 |
| 配置前缀 | 统一至 `spring.ai.*` |

### 2. Spring AI 2.x 中工具方法的识别机制
采用 **“声明 + 自动发现”** 模式，流程如下：
1.  **扫描**：Spring 扫描带有 `@Tool` 注解的 Bean 方法。
2.  **解析**：提取方法签名、参数类型及注解中的 `description`。
3.  **转换**：自动生成符合大模型规范的 **JSON Schema**。
4.  **推送**：在 API 请求时将 Tools 定义发送给 LLM。

---

<a name="全球模型支持矩阵"></a>
## 🌐 Spring AI 2.x：全球模型支持矩阵

### 一、国际主流商业大模型
*   **OpenAI**: `gpt-5-mini`, `gpt-4-turbo` (官方 SDK + 工具调用)
*   **Anthropic**: `Claude 4.5` (Citations API + Files API)
*   **Google**: `Gemini 1.5 Pro` (ThinkingConfig + 多模态)
*   **Mistral**: `Mixtral`, `Mistral Large` (结构化 JSON 输出)

### 二、国产大模型（重点支持）
*   **DeepSeek**: `DeepSeek-V2/Coder` (OpenAI 兼容代理)
*   **阿里云**: `Qwen-Max/Plus/VL` (灵积平台)
*   **智谱 AI**: `GLM-4`, `GLM-3-Turbo` (原生 API)
*   **月之暗面**: `Moonshot` (128K+ 超长上下文)

### 三、开源与本地化
*   **Ollama**: 本地运行 Llama 3、Qwen、Mistral。
*   **高性能平台**: Groq（极低延迟）、NVIDIA NIM（GPU 加速）。

---

<a name="总结"></a>
## ✅ 总结
Spring AI 2.x 是面向未来的 AI 应用基础设施，通过：
1.  **统一抽象接口**：`ChatClient`, `EmbeddingModel` 等。
2.  **深度模型集成**：涵盖国内外、开源及多模态模型。
3.  **企业级能力增强**：Redis 记忆、语义缓存、结构化输出。

**最终目标：** 帮助开发者**专注业务逻辑**，自由选择模型，避免厂商锁定，实现从原型到生产的平滑演进。