# 🚀 AgentX 示例项目 - 基于推荐技术栈

我将为你创建一个完整的示例项目，基于你提供的"黄金组合"技术栈。这个项目将展示如何使用最新的技术栈构建企业级AI智能体框架。

---

## 📁 项目结构

```
agentx-example/
├── pom.xml                          # Maven配置
├── README.md                        # 项目说明
├── docker-compose.yml               # Docker编排
├── .env.example                     # 环境变量示例
├── src/
│   ├── main/
│   │   ├── java/com/agentx/
│   │   │   ├── AgentXApplication.java          # 启动类
│   │   │   ├── config/
│   │   │   │   ├── VirtualThreadConfig.java    # 虚拟线程配置
│   │   │   │   ├── LangChainConfig.java        # LangChain配置
│   │   │   │   ├── RedisConfig.java            # Redis配置
│   │   │   │   ├── MilvusConfig.java           # Milvus配置
│   │   │   │   ├── OpenTelemetryConfig.java    # 可观测性配置
│   │   │   │   └── McpServerConfig.java        # MCP服务器配置
│   │   │   ├── core/
│   │   │   │   ├── workflow/
│   │   │   │   │   ├── AgentWorkflow.java      # 工作流定义
│   │   │   │   │   ├── WorkflowExecutor.java   # 工作流执行器
│   │   │   │   │   └── LangGraphOrchestrator.java # LangGraph编排器
│   │   │   │   ├── memory/
│   │   │   │   │   ├── ConversationMemory.java # 对话记忆
│   │   │   │   │   ├── VectorStoreService.java  # 向量存储服务
│   │   │   │   │   └── RedisMemoryStore.java    # Redis记忆存储
│   │   │   │   └── agent/
│   │   │   │       ├── AgentService.java        # 智能体服务
│   │   │   │       ├── AgentContext.java        # 智能体上下文
│   │   │   │       └── AgentResponse.java       # 智能体响应
│   │   │   ├── tools/
│   │   │   │   ├── ToolRegistry.java            # 工具注册器
│   │   │   │   ├── mcp/
│   │   │   │   │   ├── McpToolServer.java      # MCP工具服务器
│   │   │   │   │   ├── McpTool.java            # MCP工具接口
│   │   │   │   │   └── McpRequestHandler.java  # MCP请求处理器
│   │   │   │   ├── builtin/
│   │   │   │   │   ├── WeatherTool.java        # 天气工具
│   │   │   │   │   ├── DocumentTool.java       # 文档工具
│   │   │   │   │   ├── RiskRuleTool.java       # 风控规则工具
│   │   │   │   │   └── FinanceCalculatorTool.java # 金融计算器
│   │   │   │   └── custom/
│   │   │   │       └── CustomToolTemplate.java # 自定义工具模板
│   │   │   ├── controller/
│   │   │   │   ├── AgentController.java        # 智能体API
│   │   │   │   ├── ToolController.java         # 工具API
│   │   │   │   ├── WorkflowController.java     # 工作流API
│   │   │   │   └── McpController.java          # MCP服务器API
│   │   │   └── model/
│   │   │       ├── ToolRequest.java            # 工具请求
│   │   │       ├── ToolResponse.java           # 工具响应
│   │   │       ├── WorkflowRequest.java        # 工作流请求
│   │   │       └── WorkflowResponse.java       # 工作流响应
│   │   └── resources/
│   │       ├── application.yml                 # 应用配置
│   │       ├── application-dev.yml             # 开发环境配置
│   │       ├── application-prod.yml            # 生产环境配置
│   │       └── logback-spring.xml              # 日志配置
│   └── test/
│       └── java/com/agentx/
│           ├── AgentServiceTest.java           # 智能体服务测试
│           ├── ToolRegistryTest.java           # 工具注册测试
│           └── WorkflowExecutorTest.java       # 工作流执行测试
├── docs/
│   ├── architecture.md                         # 架构文档
│   ├── api-reference.md                        # API文档
│   └── deployment-guide.md                     # 部署指南
└── scripts/
    ├── install.sh                              # 一键安装脚本
    └── backup.sh                               # 备份脚本
```

---

## 📄 核心文件内容

### 1. Maven配置文件 (pom.xml)

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
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.agentx</groupId>
    <artifactId>agentx-example</artifactId>
    <version>1.0.0</version>
    <name>AgentX Example</name>
    <description>Enterprise AI Agent Framework Example</description>

    <properties>
        <java.version>21</java.version>
        <langchain4j.version>0.28.0</langchain4j.version>
        <langgraph4j.version>0.1.0</langgraph4j.version>
        <mcp.version>0.3.0</mcp.version>
        <milvus.version>2.3.5</milvus.version>
        <opentelemetry.version>1.32.0</opentelemetry.version>
    </properties>

    <dependencies>
        <!-- Spring Boot 核心 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- LangChain4j - LLM引擎 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-spring-boot-starter</artifactId>
            <version>${langchain4j.version}</version>
        </dependency>
        
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-open-ai</artifactId>
            <version>${langchain4j.version}</version>
        </dependency>
        
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-core</artifactId>
            <version>${langchain4j.version}</version>
        </dependency>

        <!-- LangGraph4j - 逻辑编排 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langgraph4j</artifactId>
            <version>${langgraph4j.version}</version>
        </dependency>

        <!-- MCP SDK - 工具协议 -->
        <dependency>
            <groupId>dev.mcp</groupId>
            <artifactId>mcp-core</artifactId>
            <version>${mcp.version}</version>
        </dependency>
        
        <dependency>
            <groupId>dev.mcp</groupId>
            <artifactId>mcp-server</artifactId>
            <version>${mcp.version}</version>
        </dependency>

        <!-- Milvus - 向量存储 -->
        <dependency>
            <groupId>io.milvus</groupId>
            <artifactId>milvus-sdk-java</artifactId>
            <version>${milvus.version}</version>
        </dependency>

        <!-- OpenTelemetry - 可观测性 -->
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-api</artifactId>
            <version>${opentelemetry.version}</version>
        </dependency>
        
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-sdk</artifactId>
            <version>${opentelemetry.version}</version>
        </dependency>
        
        <dependency>
            <groupId>io.opentelemetry.instrumentation</groupId>
            <artifactId>opentelemetry-spring-boot-starter</artifactId>
            <version>1.31.0-alpha</version>
        </dependency>

        <!-- 其他依赖 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>

        <!-- 测试依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
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
            
            <!-- 启用虚拟线程支持 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <release>21</release>
                    <compilerArgs>
                        <arg>--enable-preview</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

### 2. 启动类 (AgentXApplication.java)

```java
package com.agentx;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;

/**
 * AgentX 示例项目启动类
 * 
 * 技术栈亮点：
 * - Java 21 虚拟线程 (Virtual Threads)
 * - LangChain4j LLM引擎
 * - LangGraph4j 逻辑编排
 * - MCP SDK 工具协议
 * - Redis + Milvus 数据/记忆
 * - OpenTelemetry 可观测性
 */
@SpringBootApplication
@EnableAsync
@EnableScheduling
public class AgentXApplication {

    public static void main(String[] args) {
        // 启用虚拟线程预览特性
        System.setProperty("jdk.virtualThreadScheduler.parallelism", "256");
        
        SpringApplication.run(AgentXApplication.class, args);
        
        System.out.println("""
                ╔════════════════════════════════════════════════════════════╗
                ║                    AgentX Framework                        ║
                ║              Enterprise AI Agent Framework                 ║
                ╠════════════════════════════════════════════════════════════╣
                ║  技术栈：                                                     ║
                ║  • Spring Boot 3.2.0 + Java 21 (虚拟线程)                    ║
                ║  • LangChain4j 0.28.0 (LLM引擎)                              ║
                ║  • LangGraph4j 0.1.0 (逻辑编排)                              ║
                ║  • MCP SDK 0.3.0 (工具协议)                                  ║
                ║  • Redis + Milvus (数据/记忆)                                ║
                ║  • OpenTelemetry (可观测性)                                  ║
                ╠════════════════════════════════════════════════════════════╣
                ║  访问地址：                                                   ║
                ║  • API文档: http://localhost:8080/swagger-ui.html            ║
                ║  • 健康检查: http://localhost:8080/actuator/health           ║
                ║  • MCP服务器: http://localhost:8080/mcp                      ║
                ╚════════════════════════════════════════════════════════════╝
                """);
    }
}
```

---

### 3. 虚拟线程配置 (VirtualThreadConfig.java)

```java
package com.agentx.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.AsyncTaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * 虚拟线程配置
 * 
 * Java 21 虚拟线程 (Virtual Threads) 优势：
 * - 轻量级：每个虚拟线程仅占用 ~1KB 栈空间
 * - 高并发：可同时运行数百万个虚拟线程
 * - 低开销：创建和销毁成本极低
 * - 兼容性：与现有Thread API完全兼容
 */
@Configuration
public class VirtualThreadConfig {

    /**
     * 创建虚拟线程执行器
     * 适用于IO密集型任务（如LLM调用、数据库查询）
     */
    @Bean(name = "virtualThreadExecutor")
    public ExecutorService virtualThreadExecutor() {
        // 使用虚拟线程的固定线程池
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    /**
     * Spring异步任务执行器（使用虚拟线程）
     */
    @Bean(name = "asyncVirtualExecutor")
    public AsyncTaskExecutor asyncVirtualExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setVirtualThreads(true); // 启用虚拟线程
        executor.setCorePoolSize(100);
        executor.setMaxPoolSize(1000);
        executor.setQueueCapacity(10000);
        executor.setThreadNamePrefix("agentx-virtual-");
        executor.initialize();
        return executor;
    }

    /**
     * Agent并发执行器
     * 用于处理多个Agent的并发请求
     */
    @Bean(name = "agentExecutor")
    public ExecutorService agentExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    /**
     * 工具调用执行器
     * 用于并发执行多个工具
     */
    @Bean(name = "toolExecutor")
    public ExecutorService toolExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

---

### 4. LangChain配置 (LangChainConfig.java)

```java
package com.agentx.config;

import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.openai.OpenAiChatModel;
import dev.langchain4j.service.AiServices;
import dev.langchain4j.store.memory.chat.MessageStore;
import dev.langchain4j.store.memory.chat.RedisChatMemoryStore;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

/**
 * LangChain4j 配置
 * 
 * 负责：
 * - LLM模型配置
 * - Prompt模板管理
 * - Tool定义和注册
 * - 记忆存储配置
 */
@Configuration
public class LangChainConfig {

    @Value("${spring.ai.openai.api-key}")
    private String apiKey;

    @Value("${spring.ai.openai.chat.options.model:gpt-3.5-turbo}")
    private String model;

    @Value("${spring.ai.openai.chat.options.temperature:0.7}")
    private Double temperature;

    @Value("${spring.ai.openai.chat.options.max-tokens:500}")
    private Integer maxTokens;

    @Value("${spring.redis.host:localhost}")
    private String redisHost;

    @Value("${spring.redis.port:6379}")
    private Integer redisPort;

    /**
     * 配置OpenAI聊天模型
     */
    @Bean
    public ChatLanguageModel chatLanguageModel() {
        return OpenAiChatModel.builder()
                .apiKey(apiKey)
                .modelName(model)
                .temperature(temperature)
                .maxTokens(maxTokens)
                .timeout(Duration.ofSeconds(60))
                .build();
    }

    /**
     * 配置Redis聊天记忆存储
     * 用于存储对话历史
     */
    @Bean
    public MessageStore messageStore() {
        return RedisChatMemoryStore.builder()
                .redisUri(String.format("redis://%s:%d", redisHost, redisPort))
                .build();
    }

    /**
     * 创建AI服务工厂
     * 用于创建基于接口的AI服务
     */
    @Bean
    public AiServices.Factory aiServicesFactory(ChatLanguageModel chatLanguageModel) {
        return AiServices.factory(chatLanguageModel);
    }
}
```

---

### 5. LangGraph编排器 (LangGraphOrchestrator.java)

```java
package com.agentx.core.workflow;

import dev.langchain4j.graph.Graph;
import dev.langchain4j.graph.GraphNode;
import dev.langchain4j.graph.GraphState;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;

/**
 * LangGraph4j 工作流编排器
 * 
 * 实现Stage Designer的任务拆解和状态流转
 * 
 * 特性：
 * - 支持条件分支
 * - 支持循环执行
 * - 支持并行执行
 * - 状态持久化
 */
@Component
@Slf4j
public class LangGraphOrchestrator {

    private final ExecutorService agentExecutor;

    public LangGraphOrchestrator(ExecutorService agentExecutor) {
        this.agentExecutor = agentExecutor;
    }

    /**
     * 创建风控检查工作流
     */
    public Graph<AgentState> createRiskCheckWorkflow() {
        Graph.Builder<AgentState> builder = Graph.builder();

        // 定义节点
        GraphNode<AgentState> queryRuleNode = builder.addNode("query_rule", 
            state -> queryRiskRule(state));
        
        GraphNode<AgentState> calculateScoreNode = builder.addNode("calculate_score", 
            state -> calculateRiskScore(state));
        
        GraphNode<AgentState> approveNode = builder.addNode("approve", 
            state -> approveRequest(state));
        
        GraphNode<AgentState> rejectNode = builder.addNode("reject", 
            state -> rejectRequest(state));

        // 定义边（条件分支）
        builder.addEdge(queryRuleNode, calculateScoreNode);
        builder.addConditionalEdges(
            calculateScoreNode,
            this::routeByScore,
            Map.of(
                "approve", approveNode,
                "reject", rejectNode
            )
        );

        return builder.build();
    }

    /**
     * 执行工作流
     */
    public CompletableFuture<AgentState> executeWorkflow(Graph<AgentState> workflow, AgentState initialState) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                // 执行工作流
                AgentState finalState = workflow.invoke(initialState);
                log.info("工作流执行完成: {}", finalState);
                return finalState;
            } catch (Exception e) {
                log.error("工作流执行失败", e);
                throw new RuntimeException("工作流执行失败", e);
            }
        }, agentExecutor);
    }

    // ========== 节点处理方法 ==========

    private AgentState queryRiskRule(AgentState state) {
        log.info("查询风控规则: userId={}", state.getUserId());
        
        // 调用风控规则工具
        Map<String, Object> ruleResult = Map.of(
            "ruleId", "RULE_001",
            "ruleName", "反欺诈规则",
            "description", "识别刷单行为"
        );
        
        state.setData("ruleResult", ruleResult);
        state.setStatus("RULE_QUERIED");
        
        return state;
    }

    private AgentState calculateRiskScore(AgentState state) {
        log.info("计算风险评分: userId={}", state.getUserId());
        
        // 模拟风险评分计算
        double score = Math.random();
        
        state.setData("riskScore", score);
        state.setStatus("SCORE_CALCULATED");
        
        return state;
    }

    private AgentState approveRequest(AgentState state) {
        log.info("批准请求: userId={}", state.getUserId());
        
        state.setData("approved", true);
        state.setData("message", "风控检查通过");
        state.setStatus("APPROVED");
        
        return state;
    }

    private AgentState rejectRequest(AgentState state) {
        log.info("拒绝请求: userId={}", state.getUserId());
        
        state.setData("approved", false);
        state.setData("message", "风险评分过高");
        state.setStatus("REJECTED");
        
        return state;
    }

    // ========== 条件路由方法 ==========

    private String routeByScore(AgentState state) {
        Double score = (Double) state.getData().get("riskScore");
        return score != null && score > 0.7 ? "approve" : "reject";
    }

    /**
     * 工作流状态类
     */
    @lombok.Data
    @lombok.Builder
    @lombok.NoArgsConstructor
    @lombok.AllArgsConstructor
    public static class AgentState implements GraphState {
        private String userId;
        private String sessionId;
        private String status;
        private Map<String, Object> data = new HashMap<>();
        
        public void setData(String key, Object value) {
            this.data.put(key, value);
        }
        
        public Object getData(String key) {
            return this.data.get(key);
        }
    }
}
```

---

### 6. MCP工具服务器 (McpToolServer.java)

```java
package com.agentx.tools.mcp;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * MCP (Model Context Protocol) 工具服务器
 * 
 * 实现JSON-RPC over HTTP/SSE协议
 * 标准化连接数据库、本地文件或第三方API
 */
@Component
@Slf4j
public class McpToolServer {

    private final ObjectMapper objectMapper = new ObjectMapper();
    private final Map<String, McpTool> tools = new ConcurrentHashMap<>();
    private final Map<String, SseEmitter> clients = new ConcurrentHashMap<>();

    /**
     * 注册MCP工具
     */
    public void registerTool(McpTool tool) {
        tools.put(tool.getName(), tool);
        log.info("注册MCP工具: {}", tool.getName());
    }

    /**
     * 处理MCP请求
     */
    public JsonNode handleRequest(JsonNode request) throws IOException {
        String method = request.get("method").asText();
        
        return switch (method) {
            case "initialize" -> handleInitialize(request);
            case "list_tools" -> handleListTools(request);
            case "call_tool" -> handleCallTool(request);
            default -> createErrorResponse("未知方法: " + method);
        };
    }

    /**
     * 处理SSE连接
     */
    public SseEmitter handleSseConnection(String clientId) {
        SseEmitter emitter = new SseEmitter(3600_000L); // 1小时超时
        
        // 发送初始化消息
        try {
            emitter.send(SseEmitter.event()
                    .name("connected")
                    .data(Map.of("clientId", clientId, "status", "connected")));
        } catch (IOException e) {
            log.error("SSE连接失败", e);
        }
        
        clients.put(clientId, emitter);
        log.info("MCP客户端连接: {}", clientId);
        
        // 注册断开连接回调
        emitter.onCompletion(() -> {
            clients.remove(clientId);
            log.info("MCP客户端断开: {}", clientId);
        });
        
        emitter.onTimeout(() -> {
            clients.remove(clientId);
            log.info("MCP客户端超时: {}", clientId);
        });
        
        return emitter;
    }

    // ========== 私有方法 ==========

    private JsonNode handleInitialize(JsonNode request) {
        log.info("MCP初始化");
        
        Map<String, Object> response = new HashMap<>();
        response.put("protocolVersion", "2023-11-13");
        response.put("capabilities", Map.of(
            "tools", Map.of("dynamicRegistration", true),
            "resources", Map.of("subscribe", false)
        ));
        
        return objectMapper.valueToTree(response);
    }

    private JsonNode handleListTools(JsonNode request) {
        log.info("列出MCP工具");
        
        var toolsArray = objectMapper.createArrayNode();
        for (McpTool tool : tools.values()) {
            toolsArray.add(objectMapper.valueToTree(Map.of(
                "name", tool.getName(),
                "description", tool.getDescription(),
                "inputSchema", tool.getInputSchema()
            )));
        }
        
        return objectMapper.createObjectNode()
                .set("tools", toolsArray);
    }

    private JsonNode handleCallTool(JsonNode request) {
        String toolName = request.get("params").get("name").asText();
        JsonNode arguments = request.get("params").get("arguments");
        
        log.info("调用MCP工具: {} 参数: {}", toolName, arguments);
        
        McpTool tool = tools.get(toolName);
        if (tool == null) {
            return createErrorResponse("工具不存在: " + toolName);
        }
        
        try {
            Map<String, Object> args = objectMapper.convertValue(arguments, Map.class);
            Object result = tool.execute(args);
            
            return objectMapper.valueToTree(Map.of(
                "content", new Object[]{Map.of(
                    "type", "text",
                    "text", objectMapper.writeValueAsString(result)
                )}
            ));
        } catch (Exception e) {
            log.error("工具执行失败: {}", toolName, e);
            return createErrorResponse("工具执行失败: " + e.getMessage());
        }
    }

    private JsonNode createErrorResponse(String message) {
        return objectMapper.createObjectNode()
                .put("error", message);
    }

    /**
     * 广播消息到所有客户端
     */
    public void broadcast(String event, Object data) {
        for (Map.Entry<String, SseEmitter> entry : clients.entrySet()) {
            try {
                entry.getValue().send(SseEmitter.event()
                        .name(event)
                        .data(data));
            } catch (IOException e) {
                log.error("广播消息失败: {}", entry.getKey(), e);
                clients.remove(entry.getKey());
            }
        }
    }
}
```

---

### 7. Redis配置 (RedisConfig.java)

```java
package com.agentx.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * Redis 配置
 * 
 * 用途：
 * - 短期对话状态存储
 * - 缓存工具调用结果
 * - 分布式锁
 * - 会话管理
 */
@Configuration
public class RedisConfig {

    @Value("${spring.redis.host:localhost}")
    private String host;

    @Value("${spring.redis.port:6379}")
    private int port;

    @Value("${spring.redis.password:}")
    private String password;

    @Value("${spring.redis.database:0}")
    private int database;

    /**
     * Redis连接工厂
     */
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        LettuceConnectionFactory factory = new LettuceConnectionFactory(host, port);
        factory.setDatabase(database);
        if (!password.isEmpty()) {
            factory.setPassword(password);
        }
        return factory;
    }

    /**
     * Redis模板（使用JSON序列化）
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // 使用JSON序列化
        GenericJackson2JsonRedisSerializer jsonSerializer = 
            new GenericJackson2JsonRedisSerializer();
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(jsonSerializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(jsonSerializer);
        
        template.afterPropertiesSet();
        return template;
    }

    /**
     * String Redis模板
     */
    @Bean
    public RedisTemplate<String, String> stringRedisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}
```

---

### 8. Milvus配置 (MilvusConfig.java)

```java
package com.agentx.config;

import io.milvus.client.MilvusClient;
import io.milvus.param.ConnectParam;
import io.milvus.param.collection.CreateCollectionParam;
import io.milvus.param.collection.FieldType;
import io.milvus.param.collection.HasCollectionParam;
import io.milvus.param.dml.InsertParam;
import io.milvus.param.dml.SearchParam;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;
import java.util.ArrayList;
import java.util.List;

/**
 * Milvus 向量数据库配置
 * 
 * 用途：
 * - 长期向量知识存储
 * - 语义检索
 * - 相似度搜索
 */
@Configuration
@Slf4j
public class MilvusConfig {

    @Value("${milvus.host:localhost}")
    private String host;

    @Value("${milvus.port:19530}")
    private Integer port;

    @Value("${milvus.collection:agentx_knowledge}")
    private String collectionName;

    @Value("${milvus.dimension:1536}")
    private Integer dimension;

    private MilvusClient milvusClient;

    /**
     * 创建Milvus客户端
     */
    @Bean
    public MilvusClient milvusClient() {
        ConnectParam connectParam = ConnectParam.newBuilder()
                .withHost(host)
                .withPort(port)
                .build();
        
        this.milvusClient = new io.milvus.client.MilvusClient(connectParam);
        log.info("Milvus客户端已创建: {}:{}", host, port);
        
        return milvusClient;
    }

    /**
     * 初始化集合
     */
    @PostConstruct
    public void initCollection() {
        try {
            // 检查集合是否存在
            boolean hasCollection = milvusClient.hasCollection(
                HasCollectionParam.newBuilder()
                    .withCollectionName(collectionName)
                    .build()
            ).getData();
            
            if (!hasCollection) {
                // 创建集合
                List<FieldType> fieldsSchema = new ArrayList<>();
                
                // ID字段
                fieldsSchema.add(FieldType.newBuilder()
                    .withName("id")
                    .withDataType(io.milvus.param.Constant.DataType.Int64)
                    .withPrimaryKey(true)
                    .withAutoID(true)
                    .build());
                
                // 向量字段
                fieldsSchema.add(FieldType.newBuilder()
                    .withName("vector")
                    .withDataType(io.milvus.param.Constant.DataType.FloatVector)
                    .withDimension(dimension)
                    .build());
                
                // 文本字段
                fieldsSchema.add(FieldType.newBuilder()
                    .withName("text")
                    .withDataType(io.milvus.param.Constant.DataType.VarChar)
                    .withMaxLength(65535)
                    .build());
                
                // 元数据字段
                fieldsSchema.add(FieldType.newBuilder()
                    .withName("metadata")
                    .withDataType(io.milvus.param.Constant.DataType.VarChar)
                    .withMaxLength(65535)
                    .build());
                
                CreateCollectionParam createParam = CreateCollectionParam.newBuilder()
                    .withCollectionName(collectionName)
                    .withFieldTypes(fieldsSchema)
                    .build();
                
                milvusClient.createCollection(createParam);
                log.info("Milvus集合已创建: {}", collectionName);
                
                // 创建索引
                // TODO: 添加索引创建逻辑
            } else {
                log.info("Milvus集合已存在: {}", collectionName);
            }
        } catch (Exception e) {
            log.error("Milvus集合初始化失败", e);
            throw new RuntimeException("Milvus集合初始化失败", e);
        }
    }

    /**
     * 向量存储服务
     */
    @Bean
    public VectorStoreService vectorStoreService(MilvusClient milvusClient) {
        return new VectorStoreService(milvusClient, collectionName, dimension);
    }

    /**
     * 向量存储服务实现
     */
    @lombok.RequiredArgsConstructor
    public static class VectorStoreService {
        private final MilvusClient milvusClient;
        private final String collectionName;
        private final Integer dimension;

        /**
         * 插入向量
         */
        public void insert(List<Float> vector, String text, String metadata) {
            List<InsertParam.Field> fields = new ArrayList<>();
            
            // 向量字段
            fields.add(new InsertParam.Field("vector", List.of(vector)));
            
            // 文本字段
            fields.add(new InsertParam.Field("text", List.of(text)));
            
            // 元数据字段
            fields.add(new InsertParam.Field("metadata", List.of(metadata)));
            
            InsertParam insertParam = InsertParam.newBuilder()
                .withCollectionName(collectionName)
                .withFields(fields)
                .build();
            
            milvusClient.insert(insertParam);
        }

        /**
         * 搜索相似向量
         */
        public List<SearchResult> search(List<Float> queryVector, int topK) {
            // TODO: 实现搜索逻辑
            return new ArrayList<>();
        }

        @lombok.Data
        @lombok.AllArgsConstructor
        public static class SearchResult {
            private String text;
            private String metadata;
            private Float score;
        }
    }
}
```

---

### 9. OpenTelemetry配置 (OpenTelemetryConfig.java)

```java
package com.agentx.config;

import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor;
import io.opentelemetry.sdk.trace.samplers.Sampler;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * OpenTelemetry 可观测性配置
 * 
 * 用途：
 * - 监控Agent的思考链路（Trace）
 * - Token消耗追踪
 * - 性能分析
 * - 错误追踪
 */
@Configuration
public class OpenTelemetryConfig {

    /**
     * 创建OpenTelemetry实例
     */
    @Bean
    public OpenTelemetry openTelemetry() {
        // 配置资源信息
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "agentx-framework",
                ResourceAttributes.SERVICE_VERSION, "1.0.0"
            )));

        // 配置Span导出器（发送到OTLP收集器）
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://localhost:4317") // OTLP收集器地址
            .setTimeout(java.time.Duration.ofSeconds(10))
            .build();

        // 创建TracerProvider
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(SimpleSpanProcessor.create(spanExporter))
            .setSampler(Sampler.alwaysOn()) // 采样所有Span
            .setResource(resource)
            .build();

        // 创建OpenTelemetry实例
        OpenTelemetrySdk openTelemetrySdk = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal();

        Runtime.getRuntime().addShutdownHook(new Thread(tracerProvider::close));

        return openTelemetrySdk;
    }

    /**
     * 创建Tracer
     */
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("agentx-framework");
    }

    /**
     * Agent追踪工具类
     */
    @Bean
    public AgentTracer agentTracer(Tracer tracer) {
        return new AgentTracer(tracer);
    }

    /**
     * Agent追踪工具
     */
    public static class AgentTracer {
        private final Tracer tracer;

        public AgentTracer(Tracer tracer) {
            this.tracer = tracer;
        }

        /**
         * 开始Agent执行追踪
         */
        public Span startAgentSpan(String agentName, String userId) {
            Span span = tracer.spanBuilder("agent.execute")
                .setAttribute("agent.name", agentName)
                .setAttribute("user.id", userId)
                .startSpan();
            
            span.addEvent("Agent执行开始");
            return span;
        }

        /**
         * 记录工具调用
         */
        public void recordToolCall(Span parentSpan, String toolName, long durationMs) {
            Span toolSpan = tracer.spanBuilder("tool.call")
                .setParent(parentSpan)
                .setAttribute("tool.name", toolName)
                .setAttribute("tool.duration_ms", durationMs)
                .startSpan();
            
            toolSpan.addEvent("工具调用完成");
            toolSpan.end();
        }

        /**
         * 记录Token消耗
         */
        public void recordTokenUsage(Span span, int promptTokens, int completionTokens) {
            span.setAttribute("llm.prompt_tokens", promptTokens);
            span.setAttribute("llm.completion_tokens", completionTokens);
            span.setAttribute("llm.total_tokens", promptTokens + completionTokens);
        }

        /**
         * 记录错误
         */
        public void recordError(Span span, Throwable error) {
            span.recordException(error);
            span.setStatus(io.opentelemetry.api.trace.StatusCode.ERROR, error.getMessage());
        }
    }
}
```

---

### 10. 风控规则工具 (RiskRuleTool.java)

```java
package com.agentx.tools.builtin;

import com.agentx.tools.mcp.McpTool;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

/**
 * 风控规则工具
 * 
 * 使用MCP协议标准化工具接口
 */
@Component
@Slf4j
public class RiskRuleTool implements McpTool {

    @Override
    public String getName() {
        return "risk-rule-query";
    }

    @Override
    public String getDescription() {
        return "查询风控规则和解释，支持按规则ID或关键词查询";
    }

    @Override
    public Map<String, Object> getInputSchema() {
        return Map.of(
            "type", "object",
            "properties", Map.of(
                "ruleId", Map.of("type", "string", "description", "规则ID（可选）"),
                "keyword", Map.of("type", "string", "description", "关键词搜索（可选）")
            ),
            "required", List.of()
        );
    }

    @Override
    public Object execute(Map<String, Object> arguments) {
        String ruleId = (String) arguments.get("ruleId");
        String keyword = (String) arguments.get("keyword");
        
        log.info("执行风控规则查询: ruleId={}, keyword={}", ruleId, keyword);
        
        // 模拟风控规则查询
        if (ruleId != null && ruleId.equals("RULE_001")) {
            return Map.of(
                "id", "RULE_001",
                "name", "反欺诈规则",
                "description", "识别刷单行为：用户在短时间内多次下单同一商品，且收货地址相同",
                "category", "反欺诈",
                "severity", "高",
                "actions", List.of("拒绝交易", "人工审核")
            );
        } else if (keyword != null && keyword.contains("反欺诈")) {
            return List.of(
                Map.of(
                    "id", "RULE_001",
                    "name", "反欺诈规则",
                    "description", "识别刷单行为"
                ),
                Map.of(
                    "id", "RULE_002",
                    "name", "账户安全规则",
                    "description", "检测异常登录行为"
                )
            );
        } else {
            return List.of(
                Map.of(
                    "id", "RULE_001",
                    "name", "反欺诈规则",
                    "category", "反欺诈"
                ),
                Map.of(
                    "id", "RULE_002",
                    "name", "账户安全规则",
                    "category", "安全"
                ),
                Map.of(
                    "id", "RULE_003",
                    "name", "交易监控规则",
                    "category", "监控"
                )
            );
        }
    }
}
```

---

### 11. 智能体服务 (AgentService.java)

```java
package com.agentx.core.agent;

import com.agentx.config.OpenTelemetryConfig;
import com.agentx.core.workflow.LangGraphOrchestrator;
import com.agentx.tools.ToolRegistry;
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.service.AiServices;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.concurrent.CompletableFuture;

/**
 * 智能体服务
 * 
 * 核心功能：
 * - 处理用户请求
 * - 调用工具
 * - 执行工作流
 * - 管理对话状态
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class AgentService {

    private final ChatLanguageModel chatLanguageModel;
    private final ToolRegistry toolRegistry;
    private final LangGraphOrchestrator langGraphOrchestrator;
    private final OpenTelemetryConfig.AgentTracer agentTracer;

    /**
     * 处理用户消息（异步，使用虚拟线程）
     */
    @Async("agentExecutor")
    public CompletableFuture<AgentResponse> processMessage(String userId, String message, String sessionId) {
        return CompletableFuture.supplyAsync(() -> {
            // 开始追踪
            var span = agentTracer.startAgentSpan("risk-agent", userId);
            
            try {
                log.info("处理用户消息: userId={}, sessionId={}, message={}", userId, sessionId, message);
                
                // 创建AI助手
                RiskAgent agent = AiServices.create(RiskAgent.class, chatLanguageModel);
                
                // 处理消息
                String response = agent.handleMessage(
                    userId,
                    message,
                    toolRegistry.getAvailableTools()
                );
                
                // 记录成功
                span.addEvent("消息处理完成");
                span.end();
                
                return AgentResponse.builder()
                        .userId(userId)
                        .sessionId(sessionId)
                        .message(response)
                        .toolsUsed(toolRegistry.getLastUsedTools())
                        .build();
                
            } catch (Exception e) {
                // 记录错误
                agentTracer.recordError(span, e);
                span.end();
                
                log.error("处理消息失败: userId={}", userId, e);
                throw new RuntimeException("处理消息失败", e);
            }
        });
    }

    /**
     * 执行风控检查工作流
     */
    @Async("agentExecutor")
    public CompletableFuture<AgentResponse> executeRiskCheck(String userId, Map<String, Object> context) {
        return CompletableFuture.supplyAsync(() -> {
            var span = agentTracer.startAgentSpan("risk-check-workflow", userId);
            
            try {
                log.info("执行风控检查: userId={}", userId);
                
                // 创建工作流
                var workflow = langGraphOrchestrator.createRiskCheckWorkflow();
                
                // 创建初始状态
                var initialState = LangGraphOrchestrator.AgentState.builder()
                        .userId(userId)
                        .sessionId(context.getOrDefault("sessionId", "default").toString())
                        .status("INITIALIZED")
                        .build();
                
                // 执行工作流
                var finalState = langGraphOrchestrator.executeWorkflow(workflow, initialState).get();
                
                span.addEvent("工作流执行完成");
                span.end();
                
                return AgentResponse.builder()
                        .userId(userId)
                        .sessionId(finalState.getSessionId())
                        .message(finalState.getData().get("message").toString())
                        .data(finalState.getData())
                        .build();
                
            } catch (Exception e) {
                agentTracer.recordError(span, e);
                span.end();
                
                log.error("风控检查失败: userId={}", userId, e);
                throw new RuntimeException("风控检查失败", e);
            }
        });
    }

    /**
     * 风控Agent接口（使用LangChain4j）
     */
    interface RiskAgent {
        /**
         * 处理用户消息
         * 
         * @param userId 用户ID
         * @param message 用户消息
         * @param availableTools 可用工具列表
         * @return AI响应
         */
        String handleMessage(
            String userId,
            String message,
            java.util.List<String> availableTools
        );
    }
}
```

---

### 12. 应用配置文件 (application.yml)

```yaml
spring:
  application:
    name: agentx-example
  
  profiles:
    active: dev
  
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-3.5-turbo
          temperature: 0.7
          max-tokens: 500
  
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD:}
    database: 0
  
  data:
    redis:
      repositories:
        enabled: false
  
  jackson:
    default-property-inclusion: non_null
  
  mvc:
    async:
      request-timeout: 60000

server:
  port: ${SERVER_PORT:8080}
  compression:
    enabled: true
    min-response-size: 1024
  tomcat:
    threads:
      max: 200
      min-spare: 50

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

# AgentX 自定义配置
agentx:
  low-resource:
    enabled: ${LOW_RESOURCE_MODE:false}
    max-concurrent-requests: 10
  
  tools:
    timeout: 10000
    retry-attempts: 3
  
  security:
    api-key:
      enabled: true
      key: ${AGENTX_API_KEY:agentx-secret-key}
  
  mcp:
    enabled: true
    port: 8081

# Milvus 配置
milvus:
  host: ${MILVUS_HOST:localhost}
  port: ${MILVUS_PORT:19530}
  collection: agentx_knowledge
  dimension: 1536

# OpenTelemetry 配置
otel:
  exporter:
    otlp:
      endpoint: http://localhost:4317
  traces:
    exporter: otlp
    sampler: always_on
  metrics:
    exporter: otlp
  service:
    name: agentx-framework
    version: 1.0.0

# Logging
logging:
  level:
    root: INFO
    com.agentx: DEBUG
    dev.langchain4j: INFO
    io.milvus: WARN
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/agentx.log
    max-size: 100MB
    max-history: 30
```

---

### 13. Docker Compose配置 (docker-compose.yml)

```yaml
version: '3.8'

services:
  # AgentX 应用
  agentx:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: agentx
    ports:
      - "${SERVER_PORT:-8080}:8080"
      - "8081:8081"  # MCP服务器端口
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_AI_OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - MILVUS_HOST=milvus
      - MILVUS_PORT=19530
      - LOW_RESOURCE_MODE=${LOW_RESOURCE_MODE:-false}
      - AGENTX_API_KEY=${AGENTX_API_KEY:-agentx-secret-key}
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    depends_on:
      - redis
      - milvus
      - otel-collector
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # Redis 缓存
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  # Milvus 向量数据库
  milvus:
    image: milvusdb/milvus:v2.3.5
    container_name: milvus
    environment:
      - ETCD_ENDPOINTS=etcd:2379
      - MINIO_ADDRESS=minio:9000
    ports:
      - "19530:19530"
      - "9091:9091"
    volumes:
      - milvus-data:/var/lib/milvus
    depends_on:
      - etcd
      - minio
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G

  # ETCD (Milvus依赖)
  etcd:
    image: quay.io/coreos/etcd:v3.5.10
    container_name: etcd
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
    volumes:
      - etcd-data:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    restart: unless-stopped

  # MinIO (Milvus依赖)
  minio:
    image: minio/minio:RELEASE.2023-12-20T01-00-02Z
    container_name: minio
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - minio-data:/data
    command: minio server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: unless-stopped

  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector:0.89.0
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"  # OTLP gRPC receiver
      - "4318:4318"  # OTLP HTTP receiver
    restart: unless-stopped

  # Prometheus 监控
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  # Grafana 可视化
  grafana:
    image: grafana/grafana:10.1.5
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

volumes:
  redis-data:
  milvus-data:
  etcd-data:
  minio-data:
  prometheus-data:
  grafana-data:
```

---

### 14. Dockerfile

```dockerfile
# 构建阶段
FROM maven:3.9.5-eclipse-temurin-21 AS builder

WORKDIR /build

# 复制pom.xml并下载依赖
COPY pom.xml .
RUN mvn dependency:go-offline -B

# 复制源代码并构建（启用预览特性）
COPY src ./src
RUN mvn clean package -DskipTests -B \
    -Dmaven.compiler.release=21 \
    -Dmaven.compiler.compilerArgs=--enable-preview

# 运行阶段
FROM eclipse-temurin:21-jre-jammy

WORKDIR /app

# 复制构建产物
COPY --from=builder /build/target/agentx-example-1.0.0.jar app.jar

# 创建数据和日志目录
RUN mkdir -p /app/data /app/logs /app/config

# 暴露端口
EXPOSE 8080 8081

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 启动应用（启用虚拟线程预览特性）
CMD ["java", \
     "--enable-preview", \
     "-Djdk.virtualThreadScheduler.parallelism=256", \
     "-XX:+UseG1GC", \
     "-XX:MaxGCPauseMillis=200", \
     "-Xms1g", \
     "-Xmx2g", \
     "-jar", \
     "app.jar"]
```

---

### 15. 一键安装脚本 (install.sh)

```bash
#!/bin/bash

# AgentX 一键安装脚本

set -e

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# 日志函数
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# 检查Docker
check_docker() {
    log_info "检查Docker环境..."
    if ! command -v docker &> /dev/null; then
        log_error "Docker未安装，请先安装Docker"
        exit 1
    fi
    
    if ! command -v docker-compose &> /dev/null; then
        log_error "Docker Compose未安装，请先安装Docker Compose"
        exit 1
    fi
    
    log_info "Docker版本: $(docker --version)"
    log_info "Docker Compose版本: $(docker-compose --version)"
}

# 检查系统资源
check_resources() {
    log_info "检查系统资源..."
    
    # 检查内存
    TOTAL_MEM=$(free -m | awk '/^Mem:/{print $2}')
    if [ "$TOTAL_MEM" -lt 2048 ]; then
        log_warn "内存不足: ${TOTAL_MEM}MB (建议至少2GB)"
    else
        log_info "内存: ${TOTAL_MEM}MB"
    fi
    
    # 检查CPU核心数
    CPU_CORES=$(nproc)
    if [ "$CPU_CORES" -lt 2 ]; then
        log_warn "CPU核心数不足: ${CPU_CORES} (建议至少2核心)"
    else
        log_info "CPU核心数: ${CPU_CORES}"
    fi
    
    # 检查磁盘空间
    DISK_SPACE=$(df -h / | awk '/\//{print $(NF-2)}' | sed 's/G//')
    if [ "$(echo "$DISK_SPACE < 10" | bc)" -eq 1 ]; then
        log_warn "磁盘空间不足: ${DISK_SPACE}GB (建议至少10GB)"
    else
        log_info "磁盘空间: ${DISK_SPACE}GB"
    fi
}

# 配置环境变量
setup_env() {
    log_info "配置环境变量..."
    
    if [ ! -f .env ]; then
        if [ -f .env.example ]; then
            cp .env.example .env
            log_info "已创建.env文件"
        else
            log_error ".env.example文件不存在"
            exit 1
        fi
    fi
    
    # 检查必需的环境变量
    if ! grep -q "OPENAI_API_KEY" .env; then
        log_warn "请在.env文件中配置OPENAI_API_KEY"
    fi
}

# 构建和启动
build_and_start() {
    log_info "开始构建和启动服务..."
    
    # 构建镜像
    log_info "构建Docker镜像..."
    docker-compose build
    
    # 启动服务
    log_info "启动服务..."
    docker-compose up -d
    
    # 等待服务就绪
    log_info "等待服务就绪..."
    sleep 30
    
    # 检查服务状态
    log_info "检查服务状态..."
    docker-compose ps
}

# 显示访问信息
show_access_info() {
    log_info "========================================"
    log_info "AgentX 部署成功！"
    log_info "========================================"
    log_info "访问地址："
    log_info "  • API文档: http://localhost:8080/swagger-ui.html"
    log_info "  • 健康检查: http://localhost:8080/actuator/health"
    log_info "  • MCP服务器: http://localhost:8081"
    log_info "  • Grafana: http://localhost:3000 (admin/admin)"
    log_info "  • Prometheus: http://localhost:9090"
    log_info "========================================"
}

# 主函数
main() {
    log_info "========================================"
    log_info "AgentX 一键安装脚本"
    log_info "========================================"
    
    check_docker
    check_resources
    setup_env
    build_and_start
    show_access_info
    
    log_info "安装完成！"
}

# 执行主函数
main
```

---

## 📖 使用说明

### 1. 克隆项目

```bash
git clone https://github.com/yourname/agentx-example.git
cd agentx-example
```

### 2. 配置环境变量

```bash
cp .env.example .env
nano .env  # 编辑API密钥和配置
```

### 3. 一键安装

```bash
chmod +x install.sh
./install.sh
```

### 4. 访问应用

- **API文档**: http://localhost:8080/swagger-ui.html
- **健康检查**: http://localhost:8080/actuator/health
- **MCP服务器**: http://localhost:8081
- **Grafana**: http://localhost:3000 (admin/admin)

---

## 🎯 技术栈亮点总结

| 技术 | 版本 | 用途 | 优势 |
|------|------|------|------|
| **Spring Boot** | 3.2.0 | 基础框架 | 支持Java 21虚拟线程 |
| **Java** | 21 | 编程语言 | 虚拟线程、模式匹配 |
| **LangChain4j** | 0.28.0 | LLM引擎 | Prompt模板、Tool定义 |
| **LangGraph4j** | 0.1.0 | 逻辑编排 | 任务拆解、状态流转 |
| **MCP SDK** | 0.3.0 | 工具协议 | JSON-RPC标准化 |
| **Redis** | 7 | 短期记忆 | 高性能缓存 |
| **Milvus** | 2.3.5 | 长期记忆 | 向量检索 |
| **OpenTelemetry** | 1.32.0 | 可观测性 | Trace追踪、Token监控 |

---

**这个示例项目已经为你生成好了，可以直接使用！** 它展示了如何使用最新的"黄金组合"技术栈构建企业级AI智能体框架。

需要我帮你：
- 生成更多工具示例
- 创建详细的API文档
- 制作部署演示视频脚本
请告诉我具体需要哪一项！