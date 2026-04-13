# 项目结构

本章详细说明 AgentX 项目的代码组织结构、模块划分、配置文件和构建系统。

## 目录结构概览

```
agentx/
├── src/                          # 源代码目录
│   ├── main/                    # 主代码
│   │   ├── java/com/agentx/     # Java 源代码
│   │   │   ├── core/           # 核心模块
│   │   │   │   ├── agent/      # Agent 服务
│   │   │   │   ├── workflow/   # 工作流引擎
│   │   │   │   ├── tool/       # 工具管理
│   │   │   │   ├── memory/     # 记忆系统
│   │   │   │   ├── mcp/        # MCP 服务器
│   │   │   │   └── security/   # 安全模块
│   │   │   ├── tools/          # 内置工具
│   │   │   │   ├── weather/    # 天气工具
│   │   │   │   ├── document/   # 文档工具
│   │   │   │   ├── risk/       # 风控工具
│   │   │   │   └── finance/    # 金融工具
│   │   │   ├── store/          # 数据存储
│   │   │   │   ├── redis/      # Redis 存储
│   │   │   │   ├── milvus/     # Milvus 存储
│   │   │   │   └── postgres/   # PostgreSQL 存储
│   │   │   ├── config/         # 配置类
│   │   │   ├── controller/     # 控制器
│   │   │   ├── service/        # 服务层
│   │   │   ├── repository/     # 数据访问层
│   │   │   └── model/          # 数据模型
│   │   └── resources/          # 资源文件
│   │       ├── application.yml # 主配置文件
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── static/         # 静态资源
│   │       └── templates/      # 模板文件
│   └── test/                   # 测试代码
│       └── java/com/agentx/    # 测试类
├── docs/                       # 项目文档
├── scripts/                    # 脚本文件
├── docker/                     # Docker 相关
├── k8s/                        # Kubernetes 配置
├── .github/                    # GitHub 配置
├── .gitignore                  # Git 忽略文件
├── pom.xml                     # Maven 配置
├── README.md                   # 项目说明
└── LICENSE                     # 许可证
```

## 核心模块详解

### 1. 核心模块（core）

#### agent/ - Agent 服务
```
core/agent/
├── AgentController.java      # REST 控制器
├── AgentService.java         # 业务逻辑
├── AgentContext.java         # 会话上下文
├── AgentRequest.java         # 请求对象
├── AgentResponse.java        # 响应对象
├── AgentType.java            # Agent 类型枚举
└── AgentFactory.java         # Agent 工厂
```

**关键类说明**：
- `AgentService`：处理 AI Agent 请求的核心服务
- `AgentContext`：管理会话状态和上下文信息
- `AgentFactory`：根据类型创建不同的 Agent 实例

#### workflow/ - 工作流引擎
```
core/workflow/
├── WorkflowController.java   # 工作流控制器
├── WorkflowService.java      # 工作流服务
├── WorkflowDefinition.java   # 工作流定义
├── WorkflowStep.java         # 工作流步骤抽象
├── WorkflowExecutor.java     # 工作流执行器
├── StepResult.java           # 步骤执行结果
├── dsl/                      # DSL 解析器
│   ├── WorkflowDSLParser.java
│   └── WorkflowDSL.g4        # ANTLR 语法文件
└── langgraph/                # LangGraph 集成
    ├── LangGraphOrchestrator.java
    └── StateMachine.java
```

#### tool/ - 工具管理
```
core/tool/
├── ToolController.java       # 工具控制器
├── ToolRegistry.java         # 工具注册表
├── AgentTool.java            # 工具接口
├── ToolRequest.java          # 工具请求
├── ToolResponse.java         # 工具响应
├── ToolMetadata.java         # 工具元数据
└── ToolExecutionMetrics.java # 工具执行指标
```

#### memory/ - 记忆系统
```
core/memory/
├── ConversationMemory.java   # 会话记忆
├── VectorMemory.java         # 向量记忆
├── MemoryManager.java        # 记忆管理器
├── Memory.java               # 记忆实体
├── MemoryRetriever.java      # 记忆检索器
└── MemoryCompressor.java     # 记忆压缩器
```

#### mcp/ - MCP 服务器
```
core/mcp/
├── McpServer.java            # MCP 服务器实现
├── McpController.java        # MCP 控制器
├── McpRequest.java           # MCP 请求
├── McpResponse.java          # MCP 响应
├── McpToolAdapter.java       # MCP 工具适配器
└── McpProtocolHandler.java   # MCP 协议处理器
```

### 2. 工具模块（tools）

#### 内置工具分类
```
tools/
├── weather/                  # 天气工具
│   ├── WeatherTool.java
│   ├── WeatherService.java
│   └── WeatherData.java
├── document/                 # 文档工具
│   ├── DocumentTool.java
│   ├── DocumentParser.java
│   └── DocumentExtractor.java
├── risk/                     # 风控工具
│   ├── RiskRuleTool.java
│   ├── RiskRuleService.java
│   └── RiskRule.java
├── finance/                  # 金融工具
│   ├── FinanceCalculatorTool.java
│   ├── FinanceService.java
│   └── CalculationResult.java
└── system/                   # 系统工具
    ├── SystemInfoTool.java
    ├── FileOperationsTool.java
    └── DatabaseQueryTool.java
```

### 3. 存储模块（store）

#### 多存储支持
```
store/
├── redis/                    # Redis 存储
│   ├── RedisConfig.java
│   ├── RedisTemplateConfig.java
│   └── RedisRepository.java
├── milvus/                   # Milvus 存储
│   ├── MilvusConfig.java
│   ├── MilvusClientWrapper.java
│   └── VectorStore.java
├── postgres/                 # PostgreSQL 存储
│   ├── PostgresConfig.java
│   ├── JpaConfig.java
│   └── BaseRepository.java
└── cache/                    # 缓存抽象
    ├── CacheManager.java
    ├── MultiLevelCache.java
    └── CacheEvictionPolicy.java
```

## 配置文件说明

### 1. 主配置文件（application.yml）

```yaml
# 应用基础配置
spring:
  application:
    name: agentx
  
  profiles:
    active: dev  # 默认使用开发环境
  
  # 服务器配置
  server:
    port: 8080
    servlet:
      context-path: /
    tomcat:
      threads:
        max: 200
        min-spare: 20

# AI 模型配置
ai:
  openai:
    api-key: ${OPENAI_API_KEY}
    chat:
      options:
        model: gpt-3.5-turbo
        temperature: 0.7
        max-tokens: 1000
    embedding:
      options:
        model: text-embedding-ada-002
  
  local:
    enabled: false
    model-path: /models/ggml-model.bin
    model-type: qwen-7b

# 工具配置
tools:
  timeout: 10000  # 工具超时时间（毫秒）
  retry:
    max-attempts: 3
    backoff-delay: 1000
  cache:
    enabled: true
    ttl: 300  # 缓存时间（秒）

# 工作流配置
workflow:
  max-concurrent: 10
  timeout: 300000  # 工作流超时时间（毫秒）
  persistence:
    enabled: true
    checkpoint-interval: 60000  # 检查点间隔（毫秒）

# 监控配置
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
  metrics:
    export:
      prometheus:
        enabled: true
```

### 2. 开发环境配置（application-dev.yml）

```yaml
# 开发环境配置
spring:
  # 数据源配置（使用 H2 内存数据库）
  datasource:
    url: jdbc:h2:mem:agentx_dev
    driver-class-name: org.h2.Driver
    username: sa
    password: 
  
  # JPA 配置
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  
  # Redis 配置（本地 Docker）
  data:
    redis:
      host: localhost
      port: 6379
      password: 
      timeout: 2000

# Milvus 配置
milvus:
  host: localhost
  port: 19530

# 日志配置
logging:
  level:
    com.agentx: DEBUG
    org.springframework: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

### 3. 生产环境配置（application-prod.yml）

```yaml
# 生产环境配置
spring:
  # 数据源配置（生产数据库）
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:agentx}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  
  # JPA 配置
  jpa:
    hibernate:
      ddl-auto: validate  # 生产环境使用 validate
    show-sql: false
  
  # Redis 配置（集群）
  data:
    redis:
      cluster:
        nodes: ${REDIS_NODES:localhost:6379}
      password: ${REDIS_PASSWORD}
      timeout: 5000
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5

# Milvus 配置（集群）
milvus:
  host: ${MILVUS_HOST}
  port: ${MILVUS_PORT}
  username: ${MILVUS_USERNAME}
  password: ${MILVUS_PASSWORD}

# 安全配置
security:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400  # 24小时

# 监控配置
management:
  endpoints:
    web:
      exposure:
        include: health,info
  metrics:
    tags:
      application: ${spring.application.name}
      environment: prod
```

## 构建系统（Maven）

### pom.xml 结构

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
        <version>3.4.4</version>
        <relativePath/>
    </parent>

    <groupId>com.agentx</groupId>
    <artifactId>agentx</artifactId>
    <version>1.0.0</version>
    <name>AgentX</name>
    <description>Enterprise AI Agent Platform</description>

    <properties>
        <java.version>21</java.version>
        <spring-ai.version>1.0.0</spring-ai.version>
        <milvus.version>2.3.3</milvus.version>
        <lombok.version>1.18.30</lombok.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Spring AI -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-milvus-store-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>

        <!-- 数据库 -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Milvus -->
        <dependency>
            <groupId>io.milvus</groupId>
            <artifactId>milvus-sdk-java</artifactId>
            <version>${milvus.version}</version>
        </dependency>

        <!-- 工具库 -->
        <dependency>
            <groupId>org.apache.tika</groupId>
            <artifactId>tika-core</artifactId>
            <version>2.9.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents.client5</groupId>
            <artifactId>httpclient5</artifactId>
        </dependency>

        <!-- 开发工具 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>

        <!-- 测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>dev</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <spring.profiles.active>dev</spring.profiles.active>
            </properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <spring.profiles.active>prod</spring.profiles.active>
            </properties>
        </profile>
    </profiles>
</project>
```

### 模块依赖关系

```
┌─────────────────────────────────────────────────────┐
│                 agentx (root module)                │
├─────────────────────────────────────────────────────┤
│ 依赖关系：                                          │
│  • spring-boot-starter-web                          │
│  • spring-ai-openai                                 │
│  • spring-boot-starter-data-redis                   │
│  • milvus-sdk-java                                  │
│  • postgresql                                       │
└─────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   core module   │ │  tools module   │ │  store module   │
│                 │ │                 │ │                 │
│ • Agent 服务    │ │ • 天气工具      │ │ • Redis 存储    │
│ • 工作流引擎    │ │ • 文档工具      │ │ • Milvus 存储   │
│ • 工具管理      │ │ • 风控工具      │ │ • PostgreSQL    │
│ • 记忆系统      │ │ • 金融工具      │ │                 │
│ • MCP 服务器    │ │                 │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

## 代码生成说明

### 自动生成代码部分

AgentX 项目包含以下自动生成的代码：

1. **实体类**：基于 JPA 注解的实体类
2. **Repository 接口**：Spring Data JPA 自动实现的 Repository
3. **MapStruct 映射器**：基于 MapStruct 注解生成的 DTO 映射代码
4. **OpenAPI 文档**：基于 SpringDoc OpenAPI 自动生成的 API 文档
5. **配置类**：Spring Boot 自动配置类

### 手动开发部分

以下代码需要手动开发：

1. **业务逻辑**：Service 层的业务逻辑实现
2. **工具实现**：自定义 AgentTool 接口实现
3. **工作流定义**：业务工作流的 DSL 定义
4. **控制器**：REST API 端点定义
5. **配置定制**：应用特定的配置定制

### 代码生成工具

```bash
# 生成 MapStruct 映射器
mvn compile

# 生成 OpenAPI 文档
mvn spring-boot:run
# 然后访问 http://localhost:8080/swagger-ui.html

# 生成 JPA 实体元数据
mvn hibernate:update
```

## 开发工作流

### 1. 环境准备
```bash
# 克隆项目
git clone https://github.com/your-org/agentx.git
cd agentx

# 安装依赖
mvn clean install

# 启动开发环境
docker-compose up -d
```

### 2. 开发新功能
```bash
# 创建新分支
git checkout -b feature/new-tool

# 开发代码
# 1. 在 tools/ 目录创建新工具
# 2. 实现 AgentTool 接口
# 3. 添加单元测试
# 4. 更新配置文件

# 运行测试
mvn test

# 提交代码
git add .
git commit -m "feat: add new tool"
```

### 3. 代码审查和合并
```bash
# 推送分支
git push origin feature/new-tool

# 创建 Pull Request
# 等待代码审查

# 合并到主分支
git checkout main
git merge feature/new-tool
git push origin main
```

### 4. 构建和部署
```bash
# 构建项目
mvn clean package -DskipTests

# 构建 Docker 镜像
docker build -t agentx:latest .

# 推送镜像
docker push registry.example.com/agentx:latest

# 部署到 Kubernetes
kubectl set image deployment/agentx-deployment agentx=registry.example.com/agentx:latest
```

## 项目约定

### 命名约定
- **包名**：`com.agentx.模块名.子模块`
- **实体类**：大驼峰，后缀省略（如 `User` 而非 `UserEntity`）
- **DTO 类**：大驼峰，后缀 `DTO`（如 `UserDTO`）
- **Service 类**：大驼峰，后缀 `Service`（如 `UserService`）
- **Repository 接口**：大驼峰，后缀 `Repository`（如 `UserRepository`）

### 代码组织
- **按功能分包**：相同功能的类放在同一个包中
- **按层分包**：controller、service、repository 分层
- **按模块分包**：core、tools、store 等模块分离

### 配置管理
- **环境分离**：dev、test、prod 环境配置分离
- **敏感信息**：敏感信息通过环境变量注入
- **配置优先级**：环境变量 > 配置文件 > 默认值

## 下一步

了解项目结构后，建议：

- **浏览代码**：实际查看项目代码，理解结构
- **运行示例**：运行项目示例，验证环境
- **修改配置**：尝试修改配置文件，观察效果
- **添加工具**：尝试添加一个简单的工具

如果需要更深入的了解，请继续阅读 [环境搭建](02-environment-setup.md) 章节。