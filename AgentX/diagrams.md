# AgentX 图表文档

## 1. 系统架构图

基于分层架构，展示AgentX平台的组件和交互。

```mermaid
graph TB
    subgraph "客户端层"
        A[Web应用]
        B[移动应用]
        C[CLI工具]
        D[其他服务]
    end

    subgraph "API网关层"
        E[REST控制器]
        F[WebSocket处理器]
        G[认证授权]
    end

    subgraph "核心服务层"
        H[智能体服务]
        I[工作流编排器]
        J[工具注册表]
        K[内存管理]
    end

    subgraph "AI模型层"
        L[LangChain集成]
        M[LLM提供商]
        N[嵌入模型]
    end

    subgraph "工具层"
        O[内置工具]
        P[MCP服务器]
        Q[自定义工具]
    end

    subgraph "数据层"
        R[Redis<br/>会话内存]
        S[Milvus<br/>向量存储]
        T[PostgreSQL<br/>元数据]
    end

    A --> E
    B --> E
    C --> E
    D --> E
    
    E --> H
    E --> I
    E --> J
    
    H --> L
    I --> L
    J --> O
    
    L --> M
    L --> N
    
    O --> P
    Q --> P
    
    H --> R
    K --> R
    K --> S
    H --> T
    I --> T
    
    style A fill:#e1f5fe
    style B fill:#e1f5fe
    style C fill:#e1f5fe
    style D fill:#e1f5fe
    style E fill:#f3e5f5
    style F fill:#f3e5f5
    style G fill:#f3e5f5
    style H fill:#e8f5e8
    style I fill:#e8f5e8
    style J fill:#e8f5e8
    style K fill:#e8f5e8
    style L fill:#fff3e0
    style M fill:#fff3e0
    style N fill:#fff3e0
    style O fill:#fce4ec
    style P fill:#fce4ec
    style Q fill:#fce4ec
    style R fill:#e8eaf6
    style S fill:#e8eaf6
    style T fill:#e8eaf6
```

## 2. 类图（核心类）

展示AgentX平台的核心Java类及其关系。

```mermaid
classDiagram
    class AgentXApplication {
        +main(String[] args)
    }
    
    class AgentController {
        -AgentService agentService
        +processAgentRequest(AgentRequest request) AgentResponse
        +getAgentStatus(String agentId) Map
    }
    
    class AgentService {
        -ConversationMemory memory
        -ToolRegistry toolRegistry
        +process(AgentRequest request) AgentResponse
        +createAgentContext(String sessionId) AgentContext
    }
    
    class AgentContext {
        -String sessionId
        -List~Message~ messageHistory
        -Map~String, Object~ userPreferences
        +addMessage(Message message)
        +getContextSummary() String
    }
    
    class ToolRegistry {
        -Map~String, Tool~ tools
        +registerTool(Tool tool)
        +executeTool(String toolName, Map params) Object
        +listTools() List~Tool~
    }
    
    class McpToolServer {
        -Map~String, McpTool~ registeredTools
        +startServer()
        +stopServer()
        +registerTool(McpTool tool) boolean
        +executeTool(String toolName, Map params) McpResponse
    }
    
    class McpRequestHandler {
        -McpToolServer mcpToolServer
        +handleRequest(McpRequest request) McpResponse
        -validateRequest(McpRequest request) boolean
    }
    
    class ConversationMemory {
        -RedisTemplate redisTemplate
        +addMessage(String sessionId, Message message)
        +getMessages(String sessionId) List~Message~
        +clearSession(String sessionId)
    }
    
    class VectorStoreService {
        -MilvusClient milvusClient
        +storeVector(String collection, List~Float~ vector, String id)
        +searchSimilar(String collection, List~Float~ vector, int k) List~SearchResult~
    }
    
    class WorkflowExecutor {
        -LangGraphOrchestrator orchestrator
        +executeWorkflow(WorkflowRequest request) WorkflowResponse
        +getWorkflowStatus(String workflowId) WorkflowStatus
    }
    
    AgentController --> AgentService : 使用
    AgentService --> ConversationMemory : 使用
    AgentService --> ToolRegistry : 使用
    AgentService --> AgentContext : 创建
    ToolRegistry --> McpToolServer : 集成
    McpRequestHandler --> McpToolServer : 使用
    McpToolServer ..|> McpTool : 注册
    ConversationMemory --> RedisTemplate : 使用
    VectorStoreService --> MilvusClient : 使用
    WorkflowExecutor --> LangGraphOrchestrator : 使用
```

## 3. 请求处理流程图

展示用户请求在AgentX平台中的处理流程。

```mermaid
flowchart TD
    A[用户请求] --> B{请求类型}
    B -->|智能体请求| C[AgentController]
    B -->|工具执行| D[ToolController]
    B -->|工作流执行| E[WorkflowController]
    B -->|MCP请求| F[McpController]
    
    C --> G[AgentService.process]
    G --> H{会话ID存在?}
    H -->|是| I[从ConversationMemory获取历史]
    H -->|否| J[创建新会话]
    
    I --> K[构建AgentContext]
    J --> K
    
    K --> L[执行工具调用<br/>（如需要）]
    L --> M[调用LLM生成响应]
    M --> N[保存到ConversationMemory]
    N --> O[返回AgentResponse]
    
    D --> P[ToolRegistry.executeTool]
    P --> Q{工具类型}
    Q -->|内置工具| R[直接执行]
    Q -->|MCP工具| S[McpToolServer.executeTool]
    R --> T[返回ToolResponse]
    S --> T
    
    E --> U[WorkflowExecutor.executeWorkflow]
    U --> V[LangGraphOrchestrator执行]
    V --> W[返回WorkflowResponse]
    
    F --> X[McpRequestHandler.handleRequest]
    X --> Y{请求方法}
    Y -->|EXECUTE| Z[执行工具]
    Y -->|LIST_TOOLS| AA[列出工具]
    Y -->|GET_TOOL| AB[获取工具详情]
    Y -->|PING| AC[返回服务器状态]
    Z --> AD[返回McpResponse]
    AA --> AD
    AB --> AD
    AC --> AD
```

## 4. 工作流执行流程图

展示LangGraph工作流的执行过程。

```mermaid
flowchart TD
    A[工作流请求] --> B[WorkflowExecutor]
    B --> C[初始化工作流状态]
    C --> D[创建LangGraph工作流]
    
    D --> E{工作流节点类型}
    E -->|智能体节点| F[调用AgentService]
    E -->|工具节点| G[调用ToolRegistry]
    E -->|条件节点| H[评估条件]
    E -->|并行节点| I[并行执行]
    
    F --> J[更新工作流状态]
    G --> J
    H --> K{条件结果}
    K -->|真| L[执行真分支]
    K -->|假| M[执行假分支]
    L --> J
    M --> J
    
    I --> N[等待所有并行任务]
    N --> J
    
    J --> O{工作流完成?}
    O -->|是| P[生成最终结果]
    O -->|否| D
    
    P --> Q[保存工作流状态]
    Q --> R[返回WorkflowResponse]
```

## 5. 产品功能结构图

展示AgentX平台的产品功能模块。

```mermaid
mindmap
  root((AgentX平台))
    核心功能
      智能体对话
        : 多轮对话管理
        : 上下文保持
        : 会话历史
      工具执行
        : 内置工具库
        : MCP协议支持
        : 自定义工具
      工作流编排
        : LangGraph集成
        : 复杂流程控制
        : 状态管理
      记忆系统
        : 短期记忆(Redis)
        : 长期记忆(向量存储)
        : 记忆检索
    集成能力
      AI模型
        : OpenAI GPT
        : 本地模型
        : 多模型支持
      向量数据库
        : Milvus集成
        : 相似性搜索
        : 向量存储
      外部服务
        : REST API
        : 数据库连接
        : 消息队列
    可观测性
      监控指标
        : 性能监控
        : 业务指标
        : 资源使用
      日志系统
        : 结构化日志
        : 分布式追踪
        : 错误追踪
      告警系统
        : 阈值告警
        : 异常检测
        : 通知机制
    部署运维
      容器化
        : Docker镜像
        : Kubernetes部署
        : Helm Chart
      伸缩性
        : 水平扩展
        : 负载均衡
        : 自动伸缩
      安全性
        : 认证授权
        : 数据加密
        : 网络安全
```

## 6. 数据流图

展示数据在AgentX平台中的流动。

```mermaid
flowchart LR
    subgraph "数据输入"
        A[用户输入]
        B[API请求]
        C[文件上传]
    end
    
    subgraph "处理层"
        D[请求解析]
        E[数据验证]
        F[上下文构建]
    end
    
    subgraph "存储层"
        G[Redis<br/>会话数据]
        H[Milvus<br/>向量数据]
        I[PostgreSQL<br/>元数据]
    end
    
    subgraph "输出层"
        J[API响应]
        K[实时流]
        L[文件下载]
    end
    
    A --> D
    B --> D
    C --> D
    
    D --> E
    E --> F
    
    F --> G
    F --> H
    F --> I
    
    G --> J
    H --> J
    I --> J
    
    J --> K
    J --> L
    
    style A fill:#bbdefb
    style B fill:#bbdefb
    style C fill:#bbdefb
    style D fill:#c8e6c9
    style E fill:#c8e6c9
    style F fill:#c8e6c9
    style G fill:#fff9c4
    style H fill:#fff9c4
    style I fill:#fff9c4
    style J fill:#ffccbc
    style K fill:#ffccbc
    style L fill:#ffccbc
```

## 7. 部署架构图

展示AgentX在生产环境中的部署架构。

```mermaid
graph TB
    subgraph "客户端"
        A[Web浏览器]
        B[移动应用]
        C[API客户端]
    end
    
    subgraph "负载均衡层"
        D[负载均衡器]
        E[CDN]
    end
    
    subgraph "应用层"
        F[AgentX实例1]
        G[AgentX实例2]
        H[AgentX实例N]
    end
    
    subgraph "服务层"
        I[Redis集群]
        J[Milvus集群]
        K[PostgreSQL集群]
    end
    
    subgraph "监控层"
        L[Prometheus]
        M[Grafana]
        N[ELK Stack]
    end
    
    A --> D
    B --> D
    C --> D
    
    D --> F
    D --> G
    D --> H
    
    F --> I
    F --> J
    F --> K
    
    G --> I
    G --> J
    G --> K
    
    H --> I
    H --> J
    H --> K
    
    F --> L
    G --> L
    H --> L
    
    L --> M
    L --> N
    
    style A fill:#e1f5fe
    style B fill:#e1f5fe
    style C fill:#e1f5fe
    style D fill:#f3e5f5
    style E fill:#f3e5f5
    style F fill:#e8f5e8
    style G fill:#e8f5e8
    style H fill:#e8f5e8
    style I fill:#fff3e0
    style J fill:#fff3e0
    style K fill:#fff3e0
    style L fill:#fce4ec
    style M fill:#fce4ec
    style N fill:#fce4ec
```

## 使用说明

1. 这些图表使用Mermaid语法编写，可以在支持Mermaid的Markdown查看器中直接渲染
2. 支持Mermaid的常见平台：
   - GitHub/GitLab Markdown
   - VS Code（安装Mermaid插件）
   - Mermaid Live Editor（在线编辑）
3. 如需修改图表，可直接编辑对应的Mermaid代码块
4. 图表定期更新，确保与代码架构同步

## 图表版本

- 生成时间：2026年4月12日
- 对应代码版本：AgentX 1.0.0
- 最后更新：2026年4月12日