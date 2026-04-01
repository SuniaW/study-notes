# Spring AI + DeepSeek 完整技术指南

> **文档说明**
> 本文档合并了以下技术文档的全部内容，确保无内容丢失：
> - Spring AI + DeepSeek 项目快速开发指南.md
> - Spring AI 2.x + DeepSeek + Tool + Advisor.md
> - Spring AI 函数调用（Function Calling）配置与实现指南.md

> **合并时间**: 2026-03-31 15:20:00
> **文档数量**: 3 个
> **总页数**: 约 9 页

---

## 目录
1. [Spring AI + DeepSeek 项目快速开发指南](#part1)
2. [Spring AI 2.x + DeepSeek + Tool + Advisor 指南](#part2)  
3. [Spring AI 函数调用配置与实现指南](#part3)
4. [综合总结与最佳实践](#summary)

---

<div id="part1"></div>

## 1. Spring AI + DeepSeek 项目快速开发指南

> **来源文件**: Spring AI + DeepSeek 项目快速开发指南.md
> **最后修改**: 2026-03-10 09:34:00

# Spring AI + DeepSeek 项目快速开发指南

本手册涵盖了从项目开始到实现实时流式对话、时效性增强、 Function Calling、工具调用等完整实现。

## 1. 项目准备

### 1.1 依赖配置 (Maven)
在 `pom.xml` 中添加 Spring AI BOM 和 OpenAI Starter（DeepSeek 遵循 OpenAI 协议）。

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
    <!-- Lombok (可选) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <!-- Spring AI BOM -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0-M5</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 1.2 配置文件
在 `application.yml` 或 `application.properties` 中配置 DeepSeek API 密钥和基础 URL。

```yaml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY}  # 你的 DeepSeek API 密钥
      base-url: https://api.deepseek.com  # DeepSeek API 地址
      chat:
        options:
          model: deepseek-chat  # 使用的模型
          temperature: 0.7
```

## 2. 基础对话实现

### 2.1 创建 Controller
```java
@RestController
@RequestMapping("/api/chat")
@RequiredArgsConstructor
public class ChatController {
    
    private final OpenAiChatClient chatClient;
    
    @PostMapping("/simple")
    public String simpleChat(@RequestBody ChatRequest request) {
        return chatClient.call(request.getMessage());
    }
    
    @GetMapping("/stream")
    public Flux<String> streamChat(@RequestParam String message) {
        return chatClient.stream(message);
    }
}
```

### 2.2 请求响应对象
```java
@Data
public class ChatRequest {
    private String message;
    private String model;
    private Double temperature;
}

@Data
public class ChatResponse {
    private String content;
    private String model;
    private LocalDateTime timestamp;
}
```

## 3. 流式响应处理

### 3.1 Server-Sent Events (SSE)
```java
@GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> sseChat(@RequestParam String message) {
    return chatClient.stream(message)
        .map(content -> ServerSentEvent.builder(content).build());
}
```

### 3.2 WebSocket 实现
```java
@Controller
public class ChatWebSocketController {
    
    @MessageMapping("/chat")
    @SendTo("/topic/messages")
    public Flux<String> handleChat(String message) {
        return chatClient.stream(message);
    }
}
```

## 4. Function Calling 集成

### 4.1 定义函数
```java
@Bean
public FunctionCallback weatherFunction() {
    return FunctionCallback.builder()
        .name("getWeather")
        .description("获取指定城市的天气信息")
        .inputType(WeatherRequest.class)
        .function(weatherService::getWeather)
        .build();
}

@Data
public class WeatherRequest {
    private String city;
    private String unit = "celsius";
}
```

### 4.2 服务实现
```java
@Service
public class WeatherService {
    
    public String getWeather(WeatherRequest request) {
        // 调用天气API或返回模拟数据
        return String.format("%s的天气：晴，温度25°C", request.getCity());
    }
}
```

## 5. 工具调用与 Advisor 模式

### 5.1 工具定义
```java
@Bean
public ToolSpecification calculatorTool() {
    return ToolSpecification.builder()
        .name("calculator")
        .description("执行数学计算")
        .parameters(Map.of(
            "expression", Map.of("type", "string", "description", "数学表达式")
        ))
        .build();
}
```

### 5.2 Advisor 配置
```java
@Bean
public Advisor chatAdvisor() {
    return Advisor.builder()
        .name("contextEnhancer")
        .description("增强对话上下文")
        .preProcess((request, context) -> {
            // 预处理：添加上下文信息
            context.put("timestamp", LocalDateTime.now().toString());
            return request;
        })
        .postProcess((response, context) -> {
            // 后处理：添加元数据
            response.setMetadata(context);
            return response;
        })
        .build();
}
```

## 6. 时效性增强

### 6.1 实时数据集成
```java
@Service
public class RealTimeDataService {
    
    public String enhanceWithRealTimeData(String query) {
        // 添加实时信息：时间、新闻、股票等
        String timeInfo = "当前时间：" + LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        return query + "\n\n上下文信息：" + timeInfo;
    }
}
```

### 6.2 缓存策略
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("chatResponses", "functionResults");
    }
}
```

## 7. 错误处理与监控

### 7.1 全局异常处理
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ErrorResponse> handleApiException(ApiException ex) {
        return ResponseEntity.status(ex.getStatus())
            .body(new ErrorResponse(ex.getMessage(), ex.getCode()));
    }
}
```

### 7.2 监控指标
```java
@Component
public class ChatMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordChatRequest(String model, long duration) {
        meterRegistry.timer("chat.requests", "model", model)
            .record(duration, TimeUnit.MILLISECONDS);
    }
}
```

## 8. 部署与优化

### 8.1 Docker 配置
```dockerfile
FROM openjdk:21-jdk-slim
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 8.2 性能优化
- 启用响应式编程
- 配置连接池
- 实现请求限流
- 添加健康检查

---

<div id="part2"></div>

## 2. Spring AI 2.x + DeepSeek + Tool + Advisor 指南

> **来源文件**: Spring AI 2.x + DeepSeek + Tool + Advisor.md
> **最后修改**: 2026-03-12 09:18:00

## 🔄 Spring AI 最新版本更新
截至 **2026 年 2 月**，Spring AI 发布了里程碑版本，更新如下：

### 🛠️ 基础栈要求
*   **Java 21**：强制要求，不再支持 Java 17，充分利用虚拟线程等新特性。
*   **Spring Boot 4.0.1**：拥抱最新的微服务架构。
*   **Spring Framework 7.0**：核心框架全面升级。

### ⚡ 主要改进点
*   **API 空安全**：遵循 `JSpecify` 规范，增强 API 健壮性。
*   **响应式优先**：全面拥抱 Project Reactor，支持高并发场景。
*   **模块化重构**：按功能拆分为独立模块，减少依赖冲突。
*   **性能优化**：底层通信层重构，提升吞吐量 30%+。

## 🎯 DeepSeek 集成要点

### 1. 配置更新
```yaml
spring:
  ai:
    openai:
      # DeepSeek 专用配置
      api-key: ${DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat
          max-tokens: 4096
          temperature: 0.7
          top-p: 0.9
      # 新增：流式响应超时设置
      streaming:
        read-timeout: 30s
        connect-timeout: 10s
```

### 2. 多模型支持
```java
@Configuration
public class ModelConfig {
    
    @Bean
    @Primary
    public ChatClient deepseekChatClient(OpenAiChatClient openAiChatClient) {
        return openAiChatClient;
    }
    
    @Bean
    public ChatClient deepseekCoderClient(OpenAiChatProperties properties) {
        // 专用代码模型
        return new OpenAiChatClient(
            properties.getApiKey(),
            properties.getBaseUrl(),
            ChatOptions.builder()
                .model("deepseek-coder")
                .temperature(0.2)
                .build()
        );
    }
}
```

## 🛠️ Tool Calling 深度集成

### 1. 工具注册机制
```java
@Configuration
public class ToolRegistryConfig {
    
    @Bean
    public ToolRegistry toolRegistry(
        List<ToolSpecification> toolSpecs,
        List<FunctionCallback> functionCallbacks) {
        
        ToolRegistry registry = new DefaultToolRegistry();
        
        // 注册声明式工具
        toolSpecs.forEach(registry::register);
        
        // 注册函数式工具
        functionCallbacks.forEach(fc -> 
            registry.register(fc.getName(), fc.getFunction()));
        
        return registry;
    }
}
```

### 2. 动态工具发现
```java
@Service
public class DynamicToolService {
    
    private final ToolRegistry toolRegistry;
    
    public void registerDynamicTool(String name, String description, 
                                   Function<Map<String, Object>, String> function) {
        ToolSpecification spec = ToolSpecification.builder()
            .name(name)
            .description(description)
            .parameters(createParameterSchema())
            .build();
            
        toolRegistry.register(spec, function);
    }
    
    private Map<String, Object> createParameterSchema() {
        return Map.of(
            "type", "object",
            "properties", Map.of(
                "input", Map.of("type", "string", "description", "输入参数")
            ),
            "required", List.of("input")
        );
    }
}
```

### 3. 工具链调用
```java
@Component
public class ToolChainExecutor {
    
    public Mono<String> executeToolChain(String query, List<String> toolSequence) {
        return Flux.fromIterable(toolSequence)
            .concatMap(toolName -> executeTool(toolName, query))
            .reduce((acc, result) -> acc + "\n" + result);
    }
    
    private Mono<String> executeTool(String toolName, String input) {
        // 执行单个工具
        return Mono.fromCallable(() -> toolRegistry.execute(toolName, input));
    }
}
```

## 🧠 Advisor 模式高级应用

### 1. 上下文感知 Advisor
```java
@Bean
public Advisor contextAwareAdvisor() {
    return Advisor.builder()
        .name("contextEnricher")
        .description("根据对话历史丰富上下文")
        .condition((request, context) -> 
            context.containsKey("conversationId"))  // 条件执行
        .preProcess((request, context) -> {
            // 从数据库加载历史对话
            String history = loadConversationHistory(
                context.get("conversationId").toString());
            context.put("history", history);
            return enhanceRequest(request, history);
        })
        .postProcess((response, context) -> {
            // 保存对话记录
            saveConversationTurn(
                context.get("conversationId").toString(),
                response.getContent());
            return response;
        })
        .build();
}
```

### 2. 链式 Advisor
```java
@Configuration
public class AdvisorChainConfig {
    
    @Bean
    public AdvisorChain mainAdvisorChain(
        Advisor validationAdvisor,
        Advisor enrichmentAdvisor,
        Advisor loggingAdvisor) {
        
        return AdvisorChain.builder()
            .addAdvisor(validationAdvisor)     // 1. 输入验证
            .addAdvisor(enrichmentAdvisor)     // 2. 上下文丰富
            .addAdvisor(loggingAdvisor)        // 3. 日志记录
            .build();
    }
    
    @Bean
    public Advisor validationAdvisor() {
        return Advisor.builder()
            .name("inputValidator")
            .preProcess((request, context) -> {
                if (request == null || request.getMessage() == null) {
                    throw new IllegalArgumentException("请求内容不能为空");
                }
                if (request.getMessage().length() > 1000) {
                    throw new IllegalArgumentException("消息过长");
                }
                return request;
            })
            .build();
    }
}
```

### 3. 可观测性 Advisor
```java
@Bean
public Advisor observabilityAdvisor(MeterRegistry meterRegistry) {
    return Advisor.builder()
        .name("metricsCollector")
        .preProcess((request, context) -> {
            context.put("startTime", System.currentTimeMillis());
            meterRegistry.counter("chat.requests.total").increment();
            return request;
        })
        .postProcess((response, context) -> {
            long duration = System.currentTimeMillis() - 
                (Long) context.get("startTime");
            meterRegistry.timer("chat.response.time")
                .record(duration, TimeUnit.MILLISECONDS);
            return response;
        })
        .build();
}
```

## 🚀 性能优化策略

### 1. 异步处理流水线
```java
@Component
public class AsyncChatPipeline {
    
    private final Scheduler boundedElastic = Schedulers.boundedElastic();
    
    public Mono<ChatResponse> processAsync(ChatRequest request) {
        return Mono.just(request)
            .publishOn(boundedElastic)
            .flatMap(this::validateRequest)
            .flatMap(this::enrichContext)
            .flatMap(this::callAI)
            .flatMap(this::postProcess)
            .timeout(Duration.ofSeconds(30))
            .onErrorResume(this::handleError);
    }
}
```

### 2. 缓存策略
```java
@Configuration
@EnableCaching
public class CacheConfiguration {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .maximumSize(1000)
            .recordStats());
        return cacheManager;
    }
    
    @Bean
    public CacheResolver toolCacheResolver() {
        return new SimpleCacheResolver(cacheManager());
    }
}
```

### 3. 连接池管理
```java
@Configuration
public class HttpClientConfig {
    
    @Bean
    public ClientHttpConnector clientHttpConnector() {
        return new ReactorClientHttpConnector(HttpClient.create()
            .responseTimeout(Duration.ofSeconds(30))
            .compress(true)
            .metrics(true, () -> MicrometerHttpClientMetricsRecorder.builder()
                .meterRegistry(meterRegistry)
                .build()));
    }
}
```

## 📊 监控与告警

### 1. Micrometer 集成
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
  tracing:
    sampling:
      probability: 1.0
```

### 2. 自定义指标
```java
@Component
public class ChatMetrics {
    
    private final DistributionSummary responseLengthSummary;
    private final Counter errorCounter;
    
    public ChatMetrics(MeterRegistry registry) {
        responseLengthSummary = DistributionSummary
            .builder("chat.response.length")
            .description("响应内容长度分布")
            .register(registry);
            
        errorCounter = Counter
            .builder("chat.errors")
            .description("聊天错误计数")
            .tag("type", "api")
            .register(registry);
    }
    
    public void recordResponse(String content) {
        responseLengthSummary.record(content.length());
    }
    
    public void recordError(String errorType) {
        errorCounter.increment();
    }
}
```

### 3. 健康检查
```java
@Component
public class DeepSeekHealthIndicator implements HealthIndicator {
    
    private final OpenAiChatClient chatClient;
    
    @Override
    public Health health() {
        try {
            String response = chatClient.call("ping");
            return Health.up()
                .withDetail("model", "deepseek-chat")
                .withDetail("response", response.substring(0, Math.min(50, response.length())))
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

## 🔧 调试与故障排除

### 1. 详细日志配置
```yaml
logging:
  level:
    org.springframework.ai: DEBUG
    org.springframework.web: DEBUG
    reactor.netty: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

### 2. 请求响应追踪
```java
@Bean
public ExchangeFilterFunction loggingFilter() {
    return ExchangeFilterFunction.ofRequestProcessor(clientRequest -> {
        log.debug("Request: {} {}", clientRequest.method(), clientRequest.url());
        clientRequest.headers().forEach((name, values) -> 
            values.forEach(value -> log.debug("{}: {}", name, value)));
        return Mono.just(clientRequest);
    });
}
```

## 🎨 最佳实践总结

### 1. 代码组织
```
src/main/java/com/example/ai/
├── config/                    # 配置类
├── controller/               # 控制器
├── service/                  # 业务服务
├── advisor/                  # Advisor 实现
├── tool/                     # 工具定义
├── model/                    # 数据模型
└── client/                   # 客户端封装
```

### 2. 配置管理
- 使用 `@ConfigurationProperties` 集中管理配置
- 环境变量覆盖配置文件
- 敏感信息使用 Secret Manager

### 3. 错误处理
- 全局异常处理器
- 自定义业务异常
- 友好错误消息
- 错误码体系

### 4. 测试策略
```java
@SpringBootTest
class ChatServiceTest {
    
    @MockBean
    private OpenAiChatClient chatClient;
    
    @Test
    void testChatWithMock() {
        when(chatClient.call(anyString()))
            .thenReturn("Mock response");
        
        // 测试逻辑
    }
}
```

---

<div id="part3"></div>

## 3. Spring AI 函数调用配置与实现指南

> **来源文件**: Spring AI 函数调用（Function Calling）配置与实现指南.md
> **最后修改**: 2026-03-10 09:46:00

# Spring AI 函数调用（Function Calling）配置与实现指南

## 1. 问题排查：为什么你的 Function 无法使用？

通过您提供的 `pom.xml`，我们发现了几个关键问题：

### 1.1 错误的 Spring Boot 版本 (4.0.0)
*   **问题**：您使用了 `<version>4.0.0</version>`。
*   **原因**：目前 Spring Boot 官方尚未发布 4.0.0 版本，当前稳定版为 3.4.x。Spring AI 1.0.0-M5 需要 Spring Boot 3.3.x 或 3.4.x。
*   **解决方案**：将 Spring Boot 版本降级到 3.4.3（当前最新稳定版）。

### 1.2 缺少必要的依赖
*   **问题**：您的依赖中缺少 `spring-ai-openai-spring-boot-starter`。
*   **原因**：Function Calling 功能由 OpenAI Starter 提供。
*   **解决方案**：添加 OpenAI Starter 依赖。

### 1.3 不完整的 BOM 配置
*   **问题**：BOM 版本可能不正确或缺失。
*   **解决方案**：使用正确的 Spring AI BOM 版本。

## 2. 正确的 pom.xml 配置

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
        <version>3.4.3</version>  <!-- 正确的 Spring Boot 版本 -->
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>spring-ai-demo</artifactId>
    <version>1.0.0</version>
    
    <properties>
        <java.version>21</java.version>
        <spring-ai.version>1.0.0-M5</spring-ai.version>
    </properties>
    
    <dependencies>
        <!-- Web 支持 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Spring AI OpenAI Starter（关键依赖） -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        </dependency>
        
        <!-- 测试依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
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
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## 3. 完整的 Function Calling 实现步骤

### 3.1 步骤 1：配置 DeepSeek API

在 `application.yml` 中配置：

```yaml
spring:
  ai:
    openai:
      api-key: ${DEEPSEEK_API_KEY}  # 你的 DeepSeek API 密钥
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat
          temperature: 0.7
```

### 3.2 步骤 2：定义 Function Bean

创建你的第一个 Function：

```java
import org.springframework.ai.model.function.FunctionCallback;
import org.springframework.ai.model.function.FunctionCallbackWrapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;
import java.util.function.Function;

@Configuration
public class FunctionConfig {
    
    /**
     * 天气查询 Function
     */
    @Bean
    public FunctionCallback weatherFunction() {
        return FunctionCallbackWrapper.builder()
            .withName("getWeather")  // Function 名称
            .withDescription("获取指定城市的天气信息")  // 描述
            .withInputType(WeatherRequest.class)  // 输入类型
            .withFunction(new WeatherService())  // 实际执行函数
            .build();
    }
    
    /**
     * 计算器 Function
     */
    @Bean  
    public FunctionCallback calculatorFunction() {
        return FunctionCallbackWrapper.builder()
            .withName("calculator")
            .withDescription("执行数学计算，支持加减乘除")
            .withInputType(CalculatorRequest.class)
            .withFunction(new CalculatorService())
            .build();
    }
}
```

### 3.3 步骤 3：定义请求模型

```java
import lombok.Data;

@Data
public class WeatherRequest {
    private String city;      // 城市名称
    private String unit;      // 温度单位：celsius 或 fahrenheit
    private String language;  // 返回语言：zh 或 en
}

@Data
public class CalculatorRequest {
    private String expression;  // 数学表达式，如 "2 + 3 * 4"
    private Integer precision;  // 精度（小数位数）
}
```

### 3.4 步骤 4：实现服务逻辑

```java
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class WeatherService implements Function<WeatherRequest, String> {
    
    // 模拟天气数据
    private static final Map<String, String> WEATHER_DATA = new ConcurrentHashMap<>();
    
    static {
        WEATHER_DATA.put("北京", "晴，温度 25°C，湿度 45%，东南风 2级");
        WEATHER_DATA.put("上海", "多云，温度 28°C，湿度 65%，东风 3级");
        WEATHER_DATA.put("广州", "阵雨，温度 30°C，湿度 80%，南风 2级");
        WEATHER_DATA.put("深圳", "晴转多云，温度 29°C，湿度 70%，东南风 3级");
    }
    
    @Override
    public String apply(WeatherRequest request) {
        String city = request.getCity();
        String unit = request.getUnit() != null ? request.getUnit() : "celsius";
        String language = request.getLanguage() != null ? request.getLanguage() : "zh";
        
        String weather = WEATHER_DATA.getOrDefault(city, 
            "未找到该城市的天气信息，请检查城市名称是否正确");
        
        // 单位转换逻辑（简化版）
        if ("fahrenheit".equals(unit) && weather.contains("°C")) {
            weather = weather.replace("°C", "°F");
        }
        
        return weather;
    }
}

@Service
public class CalculatorService implements Function<CalculatorRequest, String> {
    
    @Override
    public String apply(CalculatorRequest request) {
        try {
            String expression = request.getExpression();
            Integer precision = request.getPrecision() != null ? request.getPrecision() : 2;
            
            // 简单的表达式计算（实际项目应使用更安全的计算引擎）
            double result = evaluateExpression(expression);
            
            // 格式化结果
            return String.format("表达式: %s = %." + precision + "f", expression, result);
            
        } catch (Exception e) {
            return "计算失败: " + e.getMessage() + 
                   "。请确保表达式格式正确，例如：2 + 3 * 4";
        }
    }
    
    private double evaluateExpression(String expression) {
        // 简化实现，实际项目应使用 ScriptEngine 或第三方库
        expression = expression.replaceAll("\\s+", "");
        
        // 处理基本运算
        if (expression.contains("+")) {
            String[] parts = expression.split("\\+");
            return Double.parseDouble(parts[0]) + Double.parseDouble(parts[1]);
        } else if (expression.contains("-")) {
            String[] parts = expression.split("-");
            return Double.parseDouble(parts[0]) - Double.parseDouble(parts[1]);
        } else if (expression.contains("*")) {
            String[] parts = expression.split("\\*");
            return Double.parseDouble(parts[0]) * Double.parseDouble(parts[1]);
        } else if (expression.contains("/")) {
            String[] parts = expression.split("/");
            return Double.parseDouble(parts[0]) / Double.parseDouble(parts[1]);
        } else {
            return Double.parseDouble(expression);
        }
    }
}
```

### 3.5 步骤 5：创建 Controller

```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/chat")
public class ChatController {
    
    @Autowired
    private ChatClient chatClient;
    
    /**
     * 普通聊天（自动触发 Function Calling）
     */
    @PostMapping("/function")
    public ChatResponse chatWithFunctions(@RequestBody ChatRequest request) {
        Prompt prompt = new Prompt(request.getMessage());
        
        return chatClient.prompt(prompt)
            .call()
            .chatResponse();
    }
    
    /**
     * 指定使用特定 Function
     */
    @PostMapping("/weather")
    public String getWeather(@RequestParam String city) {
        String promptText = "请告诉我" + city + "的天气情况";
        
        Prompt prompt = new Prompt(promptText);
        
        ChatResponse response = chatClient.prompt(prompt)
            .functions("getWeather")  // 指定使用 weather function
            .call()
            .chatResponse();
        
        return response.getResult().getOutput().getContent();
    }
    
    /**
     * 复杂的多 Function 调用
     */
    @PostMapping("/complex")
    public String complexQuery(@RequestBody ComplexQuery query) {
        String promptText = query.getQuestion();
        
        // 使用 PromptTemplate 构建更复杂的提示
        PromptTemplate template = new PromptTemplate("""
            用户问题：{question}
            
            请根据问题内容，选择合适的工具来回答。
            如果涉及天气查询，请使用 getWeather 函数。
            如果涉及数学计算，请使用 calculator 函数。
            如果涉及多个方面，请按顺序调用相关函数。
            """);
        
        Map<String, Object> model = Map.of("question", promptText);
        Prompt prompt = template.create(model);
        
        ChatResponse response = chatClient.prompt(prompt)
            .functions("getWeather", "calculator")  // 允许多个函数
            .call()
            .chatResponse();
        
        return response.getResult().getOutput().getContent();
    }
}

// 请求对象
@Data
class ChatRequest {
    private String message;
    private String model;
    private Double temperature;
}

@Data  
class ComplexQuery {
    private String question;
    private List<String> preferredFunctions;
}
```

### 3.6 步骤 6：测试 Function Calling

#### 测试 1：直接调用
```bash
curl -X POST http://localhost:8080/api/chat/function \
  -H "Content-Type: application/json" \
  -d '{
    "message": "北京今天天气怎么样？",
    "model": "deepseek-chat",
    "temperature": 0.7
  }'
```

#### 测试 2：指定 Function
```bash
curl -X POST "http://localhost:8080/api/chat/weather?city=上海"
```

#### 测试 3：复杂查询
```bash
curl -X POST http://localhost:8080/api/chat/complex \
  -H "Content-Type: application/json" \
  -d '{
    "question": "先计算一下 15 * 28 等于多少，然后告诉我深圳的天气情况"
  }'
```

## 4. 高级配置与优化

### 4.1 配置 Function Calling 参数

```yaml
spring:
  ai:
    openai:
      chat:
        options:
          function-call: auto  # 可选：none, auto, 或指定函数名
          functions:           # 全局函数配置
            - name: getWeather
              description: 获取天气信息
            - name: calculator  
              description: 数学计算器
```

### 4.2 自定义 Function 注册器

```java
@Component
public class DynamicFunctionRegistry {
    
    private final Map<String, FunctionCallback> functionMap = new ConcurrentHashMap<>();
    
    /**
     * 动态注册 Function
     */
    public void registerFunction(String name, String description, 
                                Class<?> inputType, Function<?, ?> function) {
        FunctionCallback callback = FunctionCallbackWrapper.builder()
            .withName(name)
            .withDescription(description)
            .withInputType(inputType)
            .withFunction(function)
            .build();
        
        functionMap.put(name, callback);
    }
    
    /**
     * 获取所有 Function
     */
    public Collection<FunctionCallback> getAllFunctions() {
        return functionMap.values();
    }
    
    /**
     * 根据条件筛选 Function
     */
    public List<FunctionCallback> getFunctionsByCategory(String category) {
        return functionMap.values().stream()
            .filter(fc -> fc.getDescription().contains(category))
            .collect(Collectors.toList());
    }
}
```

### 4.3 Function 调用链

```java
@Service
public class FunctionChainService {
    
    public String executeFunctionChain(String query, List<String> functionNames) {
        StringBuilder result = new StringBuilder();
        
        for (String functionName : functionNames) {
            try {
                String functionResult = executeSingleFunction(functionName, query);
                result.append("【").append(functionName).append("】\n")
                      .append(functionResult).append("\n\n");
                
                // 将上一步结果作为下一步的输入（可选）
                query = functionResult;
                
            } catch (Exception e) {
                result.append("【").append(functionName).append("】执行失败: ")
                      .append(e.getMessage()).append("\n\n");
            }
        }
        
        return result.toString();
    }
}
```

### 4.4 错误处理与降级

```java
@ControllerAdvice
public class FunctionExceptionHandler {
    
    @ExceptionHandler(FunctionExecutionException.class)
    public ResponseEntity<ErrorResponse> handleFunctionError(FunctionExecutionException ex) {
        ErrorResponse error = new ErrorResponse(
            "函数执行失败",
            "FUNC_" + ex.getFunctionName().toUpperCase() + "_ERROR",
            Map.of(
                "function", ex.getFunctionName(),
                "input", ex.getInput(),
                "suggestion", getSuggestion(ex.getFunctionName())
            )
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
    
    private String getSuggestion(String functionName) {
        switch (functionName) {
            case "getWeather":
                return "请检查城市名称是否正确，或稍后重试";
            case "calculator":
                return "请确保数学表达式格式正确，如：2 + 3 * 4";
            default:
                return "请检查函数参数格式";
        }
    }
}

@Data
class FunctionExecutionException extends RuntimeException {
    private final String functionName;
    private final Object input;
    
    public FunctionExecutionException(String functionName, Object input, String message) {
        super(message);
        this.functionName = functionName;
        this.input = input;
    }
}
```

## 5. 监控与日志

### 5.1 Function 调用日志

```java
@Aspect
@Component
@Slf4j
public class FunctionLoggingAspect {
    
    @Around("@within(org.springframework.stereotype.Service) && execution(* *.*(..))")
    public Object logFunctionExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        String functionName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        long startTime = System.currentTimeMillis();
        log.info("Function {} 开始执行，参数: {}", functionName, Arrays.toString(args));
        
        try {
            Object result = joinPoint.proceed();
            long duration = System.currentTimeMillis() - startTime;
            
            log.info("Function {} 执行成功，耗时: {}ms，结果: {}", 
                    functionName, duration, 
                    result != null ? result.toString().substring(0, Math.min(100, result.toString().length())) : "null");
            
            return result;
            
        } catch (Exception e) {
            log.error("Function {} 执行失败: {}", functionName, e.getMessage(), e);
            throw e;
        }
    }
}
```

### 5.2 性能指标收集

```java
@Component
public class FunctionMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Map<String, Timer> timerMap = new ConcurrentHashMap<>();
    
    public FunctionMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void recordFunctionCall(String functionName, long duration, boolean success) {
        Timer timer = timerMap.computeIfAbsent(functionName, name -> 
            Timer.builder("function.execution.time")
                .tag("function", name)
                .register(meterRegistry));
        
        timer.record(duration, TimeUnit.MILLISECONDS);
        
        Counter counter = Counter.builder("function.calls.total")
            .tag("function", functionName)
            .tag("success", String.valueOf(success))
            .register(meterRegistry);
        
        counter.increment();
    }
}
```

## 6. 测试策略

### 6.1 单元测试

```java
@SpringBootTest
class WeatherServiceTest {
    
    @Autowired
    private WeatherService weatherService;
    
    @Test
    void testGetWeatherWithValidCity() {
        WeatherRequest request = new WeatherRequest();
        request.setCity("北京");
        request.setUnit("celsius");
        
        String result = weatherService.apply(request);
        
        assertNotNull(result);
        assertTrue(result.contains("北京"));
        assertTrue(result.contains("°C"));
    }
    
    @Test
    void testGetWeatherWithInvalidCity() {
        WeatherRequest request = new WeatherRequest();
        request.setCity("未知城市");
        
        String result = weatherService.apply(request);
        
        assertNotNull(result);
        assertTrue(result.contains("未找到"));
    }
}
```

### 6.2 集成测试

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class ChatControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private ChatClient chatClient;
    
    @Test
    void testChatWithFunctions() throws Exception {
        // 模拟 AI 响应
        ChatResponse mockResponse = new ChatResponse();
        // ... 设置模拟响应
        
        when(chatClient.prompt(any(Prompt.class)))
            .thenReturn(new ChatClientCallSpec(chatClient));
        
        mockMvc.perform(post("/api/chat/function")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"message\": \"北京天气怎么样？\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content").exists());
    }
}
```

## 7. 部署与配置

### 7.1 环境特定配置

```yaml
# application-dev.yml
spring:
  ai:
    openai:
      api-key: ${DEV_DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat
          temperature: 0.8  # 开发环境更高的创造性

# application-prod.yml  
spring:
  ai:
    openai:
      api-key: ${PROD_DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat
          temperature: 0.3  # 生产环境更稳定的输出
          max-tokens: 2048
```

### 7.2 Docker 部署

```dockerfile
# 多阶段构建
FROM openjdk:21-jdk-slim as builder
WORKDIR /app
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline
COPY src src
RUN ./mvnw package -DskipTests

FROM openjdk:21-jdk-slim
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

## 8. 常见问题与解决方案

### 问题 1：Function 不被识别
**症状**：AI 不调用定义的 Function
**解决方案**：
1. 检查 Function 名称和描述是否清晰
2. 确保 Function 已正确注册为 Spring Bean
3. 检查提示词是否明确要求使用 Function

### 问题 2：参数解析失败
**症状**：Function 被调用但参数错误
**解决方案**：
1. 检查输入类型定义是否正确
2. 确保 JSON 序列化/反序列化配置正确
3. 添加参数验证逻辑

### 问题 3：性能问题
**症状**：Function 调用缓慢
**解决方案**：
1. 实现缓存机制
2. 优化 Function 实现逻辑
3. 考虑异步执行

### 问题 4：安全性问题
**症状**：敏感信息泄露或未授权访问
**解决方案**：
1. 实现 Function 访问控制
2. 验证输入参数
3. 记录所有 Function 调用

## 9. 最佳实践总结

1. **清晰的 Function 定义**
   - 使用有意义的名称
   - 提供详细的描述
   - 定义明确的输入输出类型

2. **错误处理**
   - 友好的错误消息
   - 适当的降级策略
   - 完整的日志记录

3. **性能优化**
   - 缓存频繁调用的结果
   - 异步执行耗时操作
   - 监控和告警

4. **安全性**
   - 输入验证
   - 访问控制
   - 审计日志

5. **可维护性**
   - 模块化设计
   - 单元测试覆盖
   - 文档齐全

---

<div id="summary"></div>

## 4. 综合总结与最佳实践

### 4.1 技术栈总结
基于以上文档，Spring AI + DeepSeek 完整技术栈包括：

1. **核心框架**
   - Spring Boot 3.4+ / 4.0+
   - Spring AI 2.x+
   - Java 21+

2. **AI集成**
   - DeepSeek API（兼容OpenAI协议）
   - Function Calling 支持
   - 流式响应处理
   - Tool 和 Advisor 模式

3. **开发工具**
   - Maven/Gradle 构建工具
   - Docker 容器化
   - 单元测试框架

### 4.2 开发流程最佳实践

1. **环境准备阶段**
   - 确认正确的Spring Boot和Spring AI版本兼容性
   - 配置DeepSeek API密钥和基础URL
   - 设置开发环境变量

2. **项目初始化**
   - 使用正确的pom.xml配置
   - 配置多环境配置文件
   - 设置日志和监控

3. **核心功能开发**
   - 先实现基础对话功能
   - 逐步添加流式响应支持
   - 集成Function Calling
   - 实现Tool和Advisor模式

4. **高级功能**
   - 添加缓存机制
   - 实现错误处理和降级
   - 集成监控和告警
   - 优化性能

### 4.3 架构设计建议

1. **分层架构**
   ```
   Controller层 - 处理HTTP请求
   Service层 - 业务逻辑实现
   Repository层 - 数据访问
   Client层 - 外部API调用
   Config层 - 配置管理
   ```

2. **模块化设计**
   - 按功能划分模块
   - 清晰的包结构
   - 松耦合设计

3. **配置管理**
   - 环境特定配置
   - 敏感信息安全管理
   - 动态配置更新

### 4.4 性能优化策略

1. **响应时间优化**
   - 异步处理耗时操作
   - 缓存频繁访问的数据
   - 连接池优化

2. **资源利用**
   - 合理设置线程池
   - 内存使用监控
   - 垃圾回收调优

3. **可扩展性**
   - 水平扩展设计
   - 无状态服务
   - 负载均衡

### 4.5 监控与运维

1. **健康检查**
   - 应用健康状态
   - 依赖服务健康
   - 自定义健康指标

2. **指标监控**
   - 请求响应时间
   - 错误率监控
   - 资源使用情况

3. **日志管理**
   - 结构化日志
   - 日志级别控制
   - 日志聚合和分析

### 4.6 安全考虑

1. **API安全**
   - API密钥管理
   - 请求限流
   - 输入验证

2. **数据安全**
   - 敏感信息加密
   - 数据传输安全
   - 访问控制

3. **审计日志**
   - 操作记录
   - 异常追踪
   - 合规性要求

### 4.7 测试策略

1. **单元测试**
   - 业务逻辑测试
   - 工具函数测试
   - 边界条件测试

2. **集成测试**
   - API接口测试
   - 数据库集成测试
   - 外部服务集成测试

3. **端到端测试**
   - 完整业务流程测试
   - 性能测试
   - 安全测试

### 4.8 部署与持续集成

1. **CI/CD流程**
   - 自动化测试
   - 代码质量检查
   - 自动化部署

2. **容器化部署**
   - Docker镜像构建
   - Kubernetes编排
   - 服务网格集成

3. **环境管理**
   - 开发环境
   - 测试环境
   - 生产环境

### 4.9 后续学习建议

1. **深入学习方向**
   - Spring AI官方文档和源码
   - DeepSeek API最新特性
   - 微服务架构设计模式
   - 云原生技术栈

2. **项目实践建议**
   - 从简单项目开始，逐步增加复杂度
   - 参与开源项目贡献
   - 构建完整的AI应用案例
   - 分享经验和最佳实践

3. **社区资源**
   - Spring官方社区
   - DeepSeek开发者社区
   - 技术博客和论坛
   - 开源项目参考

---

## 附录

### 文档信息
- **合并版本**: 1.0
- **包含文档**: 3 个
- **总字符数**: 约 25,000 字符
- **维护建议**: 定期更新各源文档，重新合并

### 更新记录
| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-03-31 | 1.0 | 初始合并版本，包含3个核心文档 |

### 使用说明
1. 本文档为技术参考文档，建议结合实际项目使用
2. 各章节可独立参考，也可整体学习
3. 遇到问题时，可参考对应章节的详细说明
4. 建议定期检查源文档是否有更新

### 贡献指南
如需更新本文档：
1. 更新对应的源文档
2. 重新运行合并脚本
3. 更新版本号和更新记录
4. 验证内容完整性和准确性

---
*文档合并完成 - 保留所有原始内容，无删减*
*最后更新: 2026-03-31 15:25:00*