# 扩展框架

本章详细说明如何扩展 AgentX 框架，包括自定义 Agent 实现、新模型集成、存储后端扩展和插件机制。

## 扩展架构概览

AgentX 采用模块化设计，支持多个维度的扩展：

```
┌─────────────────────────────────────────────────────┐
│                  扩展点概览                         │
├─────────────────────────────────────────────────────┤
│ 1. Agent 扩展      │ 自定义 Agent 类型和行为       │
│ 2. 模型扩展       │ 集成新的 AI 模型和提供商       │
│ 3. 工具扩展       │ 添加新的工具类型和实现         │
│ 4. 存储扩展       │ 支持新的数据存储后端           │
│ 5. 工作流扩展     │ 自定义工作流步骤和执行器       │
│ 6. 插件扩展       │ 通过插件机制扩展功能           │
└─────────────────────────────────────────────────────┘
```

## 1. 自定义 Agent 实现

### Agent 接口体系

#### 核心接口
```java
public interface Agent {
    
    /**
     * Agent 类型标识
     */
    String getType();
    
    /**
     * 处理请求
     */
    AgentResponse process(AgentRequest request);
    
    /**
     * 获取 Agent 能力描述
     */
    AgentCapabilities getCapabilities();
    
    /**
     * 初始化 Agent
     */
    default void initialize(AgentConfig config) { }
    
    /**
     * 销毁 Agent
     */
    default void destroy() { }
}

public interface StreamingAgent extends Agent {
    
    /**
     * 流式处理请求
     */
    Flux<AgentResponse> processStreaming(AgentRequest request);
}

public interface ToolCallingAgent extends Agent {
    
    /**
     * 获取支持的工具列表
     */
    List<AgentTool> getSupportedTools();
    
    /**
     * 执行工具调用
     */
    ToolResponse executeTool(ToolRequest request);
}
```

### 实现自定义 Agent

#### 基础 Agent 示例
```java
package com.agentx.agent.custom;

import com.agentx.core.agent.*;
import org.springframework.stereotype.Component;

@Component
public class CustomAgent implements Agent {
    
    private final String type = "custom-agent";
    private AgentConfig config;
    
    @Override
    public String getType() {
        return type;
    }
    
    @Override
    public AgentResponse process(AgentRequest request) {
        // 1. 验证请求
        validateRequest(request);
        
        // 2. 预处理上下文
        AgentContext context = preprocessContext(request);
        
        // 3. 执行核心逻辑
        AgentResponse response = executeCoreLogic(context);
        
        // 4. 后处理响应
        return postprocessResponse(response);
    }
    
    @Override
    public AgentCapabilities getCapabilities() {
        return AgentCapabilities.builder()
            .supportsStreaming(false)
            .supportsTools(true)
            .maxContextLength(4096)
            .supportedModels(List.of("custom-model"))
            .build();
    }
    
    @Override
    public void initialize(AgentConfig config) {
        this.config = config;
        // 初始化资源
        initializeResources();
    }
    
    private void validateRequest(AgentRequest request) {
        if (request.getMessage() == null || request.getMessage().isEmpty()) {
            throw new IllegalArgumentException("消息不能为空");
        }
        
        if (request.getSessionId() == null) {
            request.setSessionId(UUID.randomUUID().toString());
        }
    }
    
    private AgentContext preprocessContext(AgentRequest request) {
        AgentContext context = new AgentContext();
        context.setSessionId(request.getSessionId());
        context.setMessage(request.getMessage());
        context.setParameters(request.getParameters());
        
        // 添加上下文增强
        enhanceContext(context);
        
        return context;
    }
    
    private AgentResponse executeCoreLogic(AgentContext context) {
        // 自定义逻辑实现
        String processedMessage = processMessage(context.getMessage());
        
        AgentResponse response = new AgentResponse();
        response.setMessage(processedMessage);
        response.setSessionId(context.getSessionId());
        response.setTimestamp(LocalDateTime.now());
        
        return response;
    }
    
    private AgentResponse postprocessResponse(AgentResponse response) {
        // 添加元数据
        response.setMetadata(Map.of(
            "agentType", getType(),
            "processingTime", System.currentTimeMillis(),
            "version", "1.0.0"
        ));
        
        return response;
    }
    
    private void initializeResources() {
        // 初始化数据库连接、缓存等
    }
    
    private String processMessage(String message) {
        // 自定义消息处理逻辑
        return "处理后的消息: " + message.toUpperCase();
    }
}
```

#### 支持工具调用的 Agent
```java
@Component
public class ToolEnhancedAgent implements ToolCallingAgent {
    
    @Autowired
    private ToolRegistry toolRegistry;
    
    @Autowired
    private ModelProvider modelProvider;
    
    @Override
    public AgentResponse process(AgentRequest request) {
        // 1. 使用模型判断是否需要工具调用
        ToolCallDecision decision = decideToolCall(request);
        
        if (decision.needsToolCall()) {
            // 2. 执行工具调用
            ToolResponse toolResponse = executeToolCall(decision);
            
            // 3. 结合工具结果生成最终响应
            return generateFinalResponse(request, toolResponse);
        } else {
            // 4. 直接生成响应
            return generateDirectResponse(request);
        }
    }
    
    @Override
    public List<AgentTool> getSupportedTools() {
        // 返回 Agent 支持的工具列表
        return toolRegistry.listTools().stream()
            .filter(tool -> isToolSupported(tool.getName()))
            .map(this::convertToAgentTool)
            .collect(Collectors.toList());
    }
    
    @Override
    public ToolResponse executeTool(ToolRequest request) {
        // 委托给工具注册表执行
        return toolRegistry.execute(request.getToolName(), request);
    }
    
    private ToolCallDecision decideToolCall(AgentRequest request) {
        // 使用 AI 模型判断是否需要工具调用
        String prompt = buildToolDecisionPrompt(request);
        ModelResponse response = modelProvider.generate(prompt);
        
        return parseToolDecision(response);
    }
    
    private String buildToolDecisionPrompt(AgentRequest request) {
        return String.format("""
            用户消息: %s
            可用工具: %s
            请判断是否需要调用工具，如果需要，请指定工具名称和参数。
            """, request.getMessage(), getAvailableToolsDescription());
    }
}
```

### Agent 配置和注册

#### 配置类
```java
@Configuration
public class CustomAgentConfig {
    
    @Bean
    public CustomAgent customAgent() {
        CustomAgent agent = new CustomAgent();
        
        AgentConfig config = new AgentConfig();
        config.setProperty("model", "custom-model");
        config.setProperty("temperature", 0.7);
        config.setProperty("maxTokens", 1000);
        
        agent.initialize(config);
        return agent;
    }
    
    @Bean
    public AgentRegistry agentRegistry(List<Agent> agents) {
        AgentRegistry registry = new AgentRegistry();
        
        for (Agent agent : agents) {
            registry.register(agent);
        }
        
        return registry;
    }
}
```

#### 自动发现机制
```java
@Component
public class AgentAutoDiscovery {
    
    @Autowired
    private ApplicationContext applicationContext;
    
    @Autowired
    private AgentRegistry agentRegistry;
    
    @PostConstruct
    public void discoverAgents() {
        // 发现所有实现了 Agent 接口的 Bean
        Map<String, Agent> agentBeans = applicationContext.getBeansOfType(Agent.class);
        
        for (Map.Entry<String, Agent> entry : agentBeans.entrySet()) {
            String beanName = entry.getKey();
            Agent agent = entry.getValue();
            
            agentRegistry.register(agent);
            log.info("注册 Agent: {} -> {}", beanName, agent.getType());
        }
    }
}
```

## 2. 新模型集成

### 模型提供商接口

#### 核心接口
```java
public interface ModelProvider {
    
    /**
     * 提供商名称
     */
    String getName();
    
    /**
     * 支持的模型列表
     */
    List<String> getSupportedModels();
    
    /**
     * 生成文本
     */
    String generate(String prompt, ModelOptions options);
    
    /**
     * 流式生成
     */
    Flux<String> generateStreaming(String prompt, ModelOptions options);
    
    /**
     * 生成嵌入向量
     */
    float[] generateEmbedding(String text);
    
    /**
     * 批量生成嵌入向量
     */
    List<float[]> generateEmbeddings(List<String> texts);
    
    /**
     * 检查提供商状态
     */
    HealthCheckResult healthCheck();
}

public interface ModelOptions {
    
    String getModel();
    
    Double getTemperature();
    
    Integer getMaxTokens();
    
    Double getTopP();
    
    Integer getTopK();
    
    List<String> getStopSequences();
    
    // 其他模型特定选项
}
```

### 实现自定义模型提供商

#### OpenAI 兼容提供商
```java
@Component
@ConditionalOnProperty(name = "ai.provider", havingValue = "openai")
public class OpenAIModelProvider implements ModelProvider {
    
    @Value("${ai.openai.api-key}")
    private String apiKey;
    
    @Value("${ai.openai.base-url:https://api.openai.com/v1}")
    private String baseUrl;
    
    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;
    
    public OpenAIModelProvider(RestTemplate restTemplate, 
                              ObjectMapper objectMapper) {
        this.restTemplate = restTemplate;
        this.objectMapper = objectMapper;
    }
    
    @Override
    public String getName() {
        return "openai";
    }
    
    @Override
    public List<String> getSupportedModels() {
        return List.of(
            "gpt-4", "gpt-4-turbo", "gpt-3.5-turbo",
            "text-embedding-ada-002", "text-embedding-3-small"
        );
    }
    
    @Override
    public String generate(String prompt, ModelOptions options) {
        // 构建请求
        OpenAIChatRequest request = buildChatRequest(prompt, options);
        
        // 调用 API
        OpenAIChatResponse response = callOpenAIChatAPI(request);
        
        // 提取响应
        return extractResponseText(response);
    }
    
    @Override
    public Flux<String> generateStreaming(String prompt, ModelOptions options) {
        return Flux.create(sink -> {
            try {
                // 构建流式请求
                OpenAIChatRequest request = buildChatRequest(prompt, options);
                request.setStream(true);
                
                // 调用流式 API
                callOpenAIStreamingAPI(request, chunk -> {
                    sink.next(chunk.getContent());
                });
                
                sink.complete();
            } catch (Exception e) {
                sink.error(e);
            }
        });
    }
    
    @Override
    public float[] generateEmbedding(String text) {
        OpenAIEmbeddingRequest request = new OpenAIEmbeddingRequest();
        request.setInput(text);
        request.setModel("text-embedding-ada-002");
        
        OpenAIEmbeddingResponse response = callOpenAIEmbeddingAPI(request);
        
        return response.getData().get(0).getEmbedding();
    }
    
    @Override
    public HealthCheckResult healthCheck() {
        try {
            // 发送简单的测试请求
            String testPrompt = "Hello";
            String response = generate(testPrompt, ModelOptions.defaults());
            
            return HealthCheckResult.healthy("OpenAI 服务正常");
        } catch (Exception e) {
            return HealthCheckResult.unhealthy("OpenAI 服务异常: " + e.getMessage());
        }
    }
    
    private OpenAIChatRequest buildChatRequest(String prompt, ModelOptions options) {
        OpenAIChatRequest request = new OpenAIChatRequest();
        request.setModel(options.getModel());
        request.setMessages(List.of(
            new ChatMessage("user", prompt)
        ));
        request.setTemperature(options.getTemperature());
        request.setMaxTokens(options.getMaxTokens());
        request.setTopP(options.getTopP());
        
        return request;
    }
    
    private OpenAIChatResponse callOpenAIChatAPI(OpenAIChatRequest request) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + apiKey);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        HttpEntity<OpenAIChatRequest> entity = new HttpEntity<>(request, headers);
        
        ResponseEntity<OpenAIChatResponse> response = restTemplate.exchange(
            baseUrl + "/chat/completions",
            HttpMethod.POST,
            entity,
            OpenAIChatResponse.class
        );
        
        return response.getBody();
    }
}
```

#### 本地模型提供商（Ollama）
```java
@Component
@ConditionalOnProperty(name = "ai.provider", havingValue = "ollama")
public class OllamaModelProvider implements ModelProvider {
    
    @Value("${ai.ollama.base-url:http://localhost:11434}")
    private String baseUrl;
    
    @Value("${ai.ollama.default-model:llama2}")
    private String defaultModel;
    
    private final RestTemplate restTemplate;
    
    public OllamaModelProvider(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    @Override
    public String getName() {
        return "ollama";
    }
    
    @Override
    public List<String> getSupportedModels() {
        try {
            // 动态获取 Ollama 支持的模型列表
            ResponseEntity<OllamaListResponse> response = restTemplate.getForEntity(
                baseUrl + "/api/tags",
                OllamaListResponse.class
            );
            
            return response.getBody().getModels().stream()
                .map(OllamaModel::getName)
                .collect(Collectors.toList());
        } catch (Exception e) {
            log.warn("无法获取 Ollama 模型列表，使用默认列表", e);
            return List.of("llama2", "mistral", "qwen");
        }
    }
    
    @Override
    public String generate(String prompt, ModelOptions options) {
        OllamaGenerateRequest request = new OllamaGenerateRequest();
        request.setModel(options.getModel() != null ? options.getModel() : defaultModel);
        request.setPrompt(prompt);
        request.setOptions(buildOllamaOptions(options));
        
        ResponseEntity<OllamaGenerateResponse> response = restTemplate.postForEntity(
            baseUrl + "/api/generate",
            request,
            OllamaGenerateResponse.class
        );
        
        return response.getBody().getResponse();
    }
    
    @Override
    public Flux<String> generateStreaming(String prompt, ModelOptions options) {
        return Flux.create(sink -> {
            OllamaGenerateRequest request = new OllamaGenerateRequest();
            request.setModel(options.getModel() != null ? options.getModel() : defaultModel);
            request.setPrompt(prompt);
            request.setStream(true);
            request.setOptions(buildOllamaOptions(options));
            
            // 使用 WebClient 进行流式响应
            WebClient webClient = WebClient.create(baseUrl);
            
            webClient.post()
                .uri("/api/generate")
                .bodyValue(request)
                .accept(MediaType.TEXT_EVENT_STREAM)
                .retrieve()
                .bodyToFlux(String.class)
                .subscribe(
                    chunk -> {
                        try {
                            OllamaStreamResponse response = objectMapper.readValue(
                                chunk, OllamaStreamResponse.class);
                            sink.next(response.getResponse());
                        } catch (JsonProcessingException e) {
                            sink.error(e);
                        }
                    },
                    sink::error,
                    sink::complete
                );
        });
    }
    
    private OllamaOptions buildOllamaOptions(ModelOptions options) {
        OllamaOptions ollamaOptions = new OllamaOptions();
        if (options.getTemperature() != null) {
            ollamaOptions.setTemperature(options.getTemperature());
        }
        if (options.getTopP() != null) {
            ollamaOptions.setTopP(options.getTopP());
        }
        if (options.getTopK() != null) {
            ollamaOptions.setTopK(options.getTopK());
        }
        return ollamaOptions;
    }
}
```

### 模型工厂和路由

#### 模型工厂
```java
@Component
public class ModelProviderFactory {
    
    private final Map<String, ModelProvider> providers = new ConcurrentHashMap<>();
    
    @Autowired
    public ModelProviderFactory(List<ModelProvider> providerList) {
        for (ModelProvider provider : providerList) {
            providers.put(provider.getName(), provider);
        }
    }
    
    public ModelProvider getProvider(String providerName) {
        ModelProvider provider = providers.get(providerName);
        if (provider == null) {
            throw new IllegalArgumentException("未知的模型提供商: " + providerName);
        }
        return provider;
    }
    
    public ModelProvider getDefaultProvider() {
        return providers.values().stream()
            .filter(p -> p.getName().equals("openai"))
            .findFirst()
            .orElse(providers.values().iterator().next());
    }
    
    public List<String> getAvailableProviders() {
        return new ArrayList<>(providers.keySet());
    }
}
```

#### 模型路由
```java
@Component
public class ModelRouter {
    
    @Autowired
    private ModelProviderFactory factory;
    
    @Value("${ai.routing.strategy:round-robin}")
    private String routingStrategy;
    
    private final AtomicInteger counter = new AtomicInteger(0);
    
    public ModelProvider route(ModelRequest request) {
        String providerName = determineProvider(request);
        return factory.getProvider(providerName);
    }
    
    private String determineProvider(ModelRequest request) {
        // 根据策略选择提供商
        switch (routingStrategy) {
            case "specific":
                return request.getPreferredProvider();
                
            case "model-based":
                return routeByModel(request.getModel());
                
            case "load-based":
                return routeByLoad();
                
            case "round-robin":
                return routeRoundRobin();
                
            default:
                return "openai";
        }
    }
    
    private String routeByModel(String model) {
        // 根据模型名称选择提供商
        if (model.contains("gpt")) {
            return "openai";
        } else if (model.contains("claude")) {
            return "anthropic";
        } else if (model.contains("llama") || model.contains("qwen")) {
            return "ollama";
        } else {
            return "openai";
        }
    }
    
    private String routeRoundRobin() {
        List<String> providers = factory.getAvailableProviders();
        int index = counter.getAndIncrement() % providers.size();
        return providers.get(index);
    }
}
```

## 3. 存储后端扩展

### 存储抽象层

#### 统一存储接口
```java
public interface VectorStore {
    
    /**
     * 存储名称
     */
    String getName();
    
    /**
     * 创建集合
     */
    void createCollection(String collectionName, CollectionConfig config);
    
    /**
     * 删除集合
     */
    void deleteCollection(String collectionName);
    
    /**
     * 插入向量
     */
    void insert(String collectionName, List<VectorRecord> records);
    
    /**
     * 搜索相似向量
     */
    List<SearchResult> search(String collectionName, float[] vector, SearchOptions options);
    
    /**
     * 删除向量
     */
    void delete(String collectionName, List<String> ids);
    
    /**
     * 获取集合统计信息
     */
    CollectionStats getStats(String collectionName);
}

public interface CacheStore {
    
    /**
     * 获取缓存值
     */
    <T> T get(String key, Class<T> type);
    
    /**
     * 设置缓存值
     */
    void set(String key, Object value, Duration ttl);
    
    /**
     * 删除缓存值
     */
    void delete(String key);
    
    /**
     * 检查键是否存在
     */
    boolean exists(String key);
    
    /**
     * 获取多个键的值
     */
    <T> Map<String, T> multiGet(List<String> keys, Class<T> type);
    
    /**
     * 设置多个键的值
     */
    void multiSet(Map<String, Object> values, Duration ttl);
}
```

### 实现自定义存储

#### Pinecone 向量存储
```java
@Component
@ConditionalOnProperty(name = "vector.store", havingValue = "pinecone")
public class PineconeVectorStore implements VectorStore {
    
    @Value("${pinecone.api-key}")
    private String apiKey;
    
    @Value("${pinecone.environment}")
    private String environment;
    
    private PineconeClient client;
    
    @PostConstruct
    public void init() {
        this.client = new PineconeClient.Builder()
            .withApiKey(apiKey)
            .withEnvironment(environment)
            .build();
    }
    
    @Override
    public String getName() {
        return "pinecone";
    }
    
    @Override
    public void createCollection(String collectionName, CollectionConfig config) {
        CreateCollectionRequest request = CreateCollectionRequest.builder()
            .name(collectionName)
            .dimension(config.getDimension())
            .metric(config.getMetric())
            .build();
        
        client.createCollection(request);
    }
    
    @Override
    public void insert(String collectionName, List<VectorRecord> records) {
        List<Vector> vectors = records.stream()
            .map(record -> Vector.builder()
                .id(record.getId())
                .values(record.getVector())
                .metadata(record.getMetadata())
                .build())
            .collect(Collectors.toList());
        
        UpsertRequest request = UpsertRequest.builder()
            .vectors(vectors)
            .build();
        
        client.upsert(collectionName, request);
    }
    
    @Override
    public List<SearchResult> search(String collectionName, float[] vector, SearchOptions options) {
        QueryRequest request = QueryRequest.builder()
            .vector(vector)
            .topK(options.getTopK())
            .includeMetadata(true)
            .build();
        
        QueryResponse response = client.query(collectionName, request);
        
        return response.getMatches().stream()
            .map(match -> SearchResult.builder()
                .id(match.getId())
                .score(match.getScore())
                .metadata(match.getMetadata())
                .build())
            .collect(Collectors.toList());
    }
}
```

#### Redis 缓存存储
```java
@Component
@ConditionalOnProperty(name = "cache.store", havingValue = "redis")
public class RedisCacheStore implements CacheStore {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    public <T> T get(String key, Class<T> type) {
        Object value = redisTemplate.opsForValue().get(key);
        return type.cast(value);
    }
    
    @Override
    public void set(String key, Object value, Duration ttl) {
        if (ttl != null && !ttl.isZero()) {
            redisTemplate.opsForValue().set(key, value, ttl);
        } else {
            redisTemplate.opsForValue().set(key, value);
        }
    }
    
    @Override
    public <T> Map<String, T> multiGet(List<String> keys, Class<T> type) {
        List<Object> values = redisTemplate.opsForValue().multiGet(keys);
        
        Map<String, T> result = new HashMap<>();
        for (int i = 0; i < keys.size(); i++) {
            if (values.get(i) != null) {
                result.put(keys.get(i), type.cast(values.get(i)));
            }
        }
        
        return result;
    }
    
    @Override
    public void multiSet(Map<String, Object> values, Duration ttl) {
        if (ttl != null && !ttl.isZero()) {
            values.forEach((key, value) -> {
                redisTemplate.opsForValue().set(key, value, ttl);
            });
        } else {
            redisTemplate.opsForValue().multiSet(values);
        }
    }
}
```

### 存储工厂和适配器

#### 存储工厂
```java
@Component
public class StorageFactory {
    
    private final Map<String, VectorStore> vectorStores = new ConcurrentHashMap<>();
    private final Map<String, CacheStore> cacheStores = new ConcurrentHashMap<>();
    
    @Autowired
    public StorageFactory(List<VectorStore> vectorStoreList,
                         List<CacheStore> cacheStoreList) {
        
        for (VectorStore store : vectorStoreList) {
            vectorStores.put(store.getName(), store);
        }
        
        for (CacheStore store : cacheStoreList) {
            cacheStores.put(store.getClass().getSimpleName(), store);
        }
    }
    
    public VectorStore getVectorStore(String name) {
        VectorStore store = vectorStores.get(name);
        if (store == null) {
            throw new IllegalArgumentException("未知的向量存储: " + name);
        }
        return store;
    }
    
    public CacheStore getCacheStore() {
        // 返回默认的缓存存储
        return cacheStores.values().iterator().next();
    }
    
    public VectorStore getDefaultVectorStore() {
        return vectorStores.get("milvus");
    }
}
```

#### 存储适配器
```java
@Component
public class StorageAdapter {
    
    @Autowired
    private StorageFactory factory;
    
    @Value("${storage.vector.default:milvus}")
    private String defaultVectorStore;
    
    @Value("${storage.cache.default:redis}")
    private String defaultCacheStore;
    
    public VectorStore getVectorStore() {
        return factory.getVectorStore(defaultVectorStore);
    }
    
    public CacheStore getCacheStore() {
        return factory.getCacheStore();
    }
    
    public void switchVectorStore(String storeName) {
        VectorStore store = factory.getVectorStore(storeName);
        // 执行数据迁移（如果需要）
        migrateData(defaultVectorStore, storeName);
        defaultVectorStore = storeName;
    }
    
    private void migrateData(String fromStore, String toStore) {
        // 数据迁移逻辑
        log.info("从 {} 迁移数据到 {}", fromStore, toStore);
    }
}
```

## 4. 插件机制

### 插件架构

#### 插件接口
```java
public interface AgentXPlugin {
    
    /**
     * 插件名称
     */
    String getName();
    
    /**
     * 插件版本
     */
    String getVersion();
    
    /**
     * 插件描述
     */
    String getDescription();
    
    /**
     * 初始化插件
     */
    void initialize(PluginContext context);
    
    /**
     * 销毁插件
     */
    void destroy();
    
    /**
     * 获取插件配置
     */
    PluginConfig getConfig();
    
    /**
     * 获取插件提供的服务
     */
    default List<Object> getServices() {
        return Collections.emptyList();
    }
    
    /**
     * 获取插件扩展点
     */
    default Map<String, Class<?>> getExtensionPoints() {
        return Collections.emptyMap();
    }
}

public interface PluginContext {
    
    ApplicationContext getApplicationContext();
    
    PluginConfig getPluginConfig();
    
    PluginManager getPluginManager();
    
    void registerService(String name, Object service);
    
    void unregisterService(String name);
}
```

### 插件管理器

#### 插件管理器实现
```java
@Component
public class PluginManager {
    
    private final Map<String, AgentXPlugin> plugins = new ConcurrentHashMap<>();
    private final Map<String, Object> services = new ConcurrentHashMap<>();
    private final Map<String, List<Object>> extensions = new ConcurrentHashMap<>();
    
    @Autowired
    private ApplicationContext applicationContext;
    
    /**
     * 加载插件
     */
    public void loadPlugin(String pluginPath) {
        try {
            // 动态加载插件 JAR
            URLClassLoader classLoader = new URLClassLoader(
                new URL[] { new File(pluginPath).toURI().toURL() },
                getClass().getClassLoader()
            );
            
            // 读取插件描述文件
            Properties pluginProps = new Properties();
            try (InputStream is = classLoader.getResourceAsStream("plugin.properties")) {
                pluginProps.load(is);
            }
            
            String mainClass = pluginProps.getProperty("plugin.main-class");
            Class<?> pluginClass = classLoader.loadClass(mainClass);
            AgentXPlugin plugin = (AgentXPlugin) pluginClass.newInstance();
            
            // 初始化插件
            PluginContext context = createPluginContext(pluginProps);
            plugin.initialize(context);
            
            // 注册插件
            plugins.put(plugin.getName(), plugin);
            
            // 注册服务
            List<Object> pluginServices = plugin.getServices();
            for (Object service : pluginServices) {
                registerService(service.getClass().getName(), service);
            }
            
            log.info("插件加载成功: {} v{}", plugin.getName(), plugin.getVersion());
            
        } catch (Exception e