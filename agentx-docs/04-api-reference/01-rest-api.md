# REST API 参考

本章详细说明 AgentX 平台的所有 REST API 接口，包括请求/响应格式、参数说明、示例代码和使用限制。

## API 概览

### 基础信息
- **基础 URL**：`http://localhost:8080/api/v1`（开发环境）
- **生产 URL**：`https://api.agentx.ai/api/v1`
- **默认端口**：8080（可配置）
- **协议**：HTTP/HTTPS

### 接口分类

| 类别 | 前缀 | 说明 |
|------|------|------|
| 认证授权 | `/auth/*` | 用户认证、令牌管理、权限验证 |
| Agent 管理 | `/agents/*` | Agent 创建、配置、执行、监控 |
| 工具管理 | `/tools/*` | 工具注册、发现、调用、管理 |
| 工作流 | `/workflows/*` | 工作流定义、执行、监控、管理 |
| 会话管理 | `/sessions/*` | 会话创建、维护、历史、清理 |
| 向量存储 | `/vectors/*` | 向量存储、检索、管理 |
| 系统管理 | `/system/*` | 系统状态、配置、监控、日志 |
| 插件管理 | `/plugins/*` | 插件安装、配置、启用、卸载 |

### 版本信息
- **当前版本**：v1.0.0
- **API 版本**：v1
- **兼容性**：向后兼容，新增字段不影响现有功能
- **弃用策略**：提前 3 个月通知，提供迁移工具

## 认证授权接口

### 1. 用户登录

**接口**：`POST /auth/login`

**功能**：使用用户名密码获取访问令牌。

**请求头**：
```http
Content-Type: application/json
```

**请求体**：
```json
{
  "username": "string，必填，用户名",
  "password": "string，必填，密码",
  "clientId": "string，可选，客户端ID",
  "clientSecret": "string，可选，客户端密钥"
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshExpiresIn": 2592000,
    "user": {
      "id": "user-123",
      "username": "admin",
      "email": "admin@example.com",
      "roles": ["admin", "user"],
      "permissions": ["agents:*", "tools:read"]
    }
  },
  "message": "登录成功",
  "timestamp": "2026-04-13T09:01:00Z"
}
```

**错误码**：
- `AUTH_001`：用户名或密码错误
- `AUTH_002`：账户被锁定
- `AUTH_003`：客户端认证失败
- `VALID_001`：参数验证失败

**示例**：
```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "admin123"
  }'
```

### 2. 令牌刷新

**接口**：`POST /auth/refresh`

**功能**：使用刷新令牌获取新的访问令牌。

**请求头**：
```http
Content-Type: application/json
Authorization: Bearer <refresh_token>
```

**响应**：同登录接口，返回新的访问令牌和刷新令牌。

### 3. 令牌验证

**接口**：`GET /auth/verify`

**功能**：验证令牌有效性。

**请求头**：
```http
Authorization: Bearer <access_token>
```

**响应**：
```json
{
  "success": true,
  "data": {
    "valid": true,
    "expiresAt": "2026-04-13T10:01:00Z",
    "userId": "user-123",
    "scopes": ["agents:*", "tools:read"]
  }
}
```

### 4. 用户登出

**接口**：`POST /auth/logout`

**功能**：使令牌失效。

**请求头**：
```http
Authorization: Bearer <access_token>
```

**响应**：
```json
{
  "success": true,
  "message": "登出成功",
  "timestamp": "2026-04-13T09:01:00Z"
}
```

## Agent 管理接口

### 1. 获取 Agent 列表

**接口**：`GET /agents`

**功能**：获取所有可用 Agent 的列表。

**查询参数**：
- `page`：页码，默认 1
- `size`：每页数量，默认 20
- `type`：Agent 类型过滤
- `status`：状态过滤（active、inactive、error）
- `sort`：排序字段，如 `createdAt,desc`

**响应**：
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "agent-001",
        "type": "general",
        "name": "通用助手",
        "description": "适用于通用问答场景",
        "status": "active",
        "capabilities": {
          "supportsStreaming": true,
          "supportsTools": true,
          "maxContextLength": 4096
        },
        "createdAt": "2026-04-13T08:00:00Z",
        "updatedAt": "2026-04-13T08:00:00Z"
      }
    ],
    "page": 1,
    "size": 20,
    "totalElements": 5,
    "totalPages": 1,
    "first": true,
    "last": true
  }
}
```

### 2. 获取 Agent 详情

**接口**：`GET /agents/{agentId}`

**功能**：获取指定 Agent 的详细信息。

**路径参数**：
- `agentId`：Agent ID

**响应**：
```json
{
  "success": true,
  "data": {
    "id": "agent-001",
    "type": "general",
    "name": "通用助手",
    "description": "适用于通用问答场景",
    "config": {
      "model": "gpt-4",
      "temperature": 0.7,
      "maxTokens": 1000,
      "systemPrompt": "你是一个有帮助的助手"
    },
    "status": "active",
    "metrics": {
      "totalRequests": 1500,
      "successRate": 99.8,
      "avgResponseTime": 450,
      "lastUsed": "2026-04-13T08:59:00Z"
    },
    "createdAt": "2026-04-13T08:00:00Z",
    "updatedAt": "2026-04-13T08:00:00Z"
  }
}
```

### 3. 执行 Agent

**接口**：`POST /agents/process`

**功能**：向 Agent 发送消息并获取响应。

**请求体**：
```json
{
  "agentType": "string，必填，Agent类型",
  "message": "string，必填，消息内容",
  "sessionId": "string，可选，会话ID（用于维护上下文）",
  "parameters": {
    // 额外参数
  },
  "options": {
    "streaming": false,
    "temperature": 0.7,
    "maxTokens": 1000,
    "tools": ["get-weather", "calculate"],
    "toolChoice": "auto"
  }
}
```

**响应（非流式）**：
```json
{
  "success": true,
  "data": {
    "message": "Agent的响应内容",
    "sessionId": "session-123",
    "toolCalls": [
      {
        "toolName": "get-weather",
        "parameters": {
          "city": "北京"
        },
        "response": {
          "city": "北京",
          "temperature": 25.0,
          "condition": "晴朗"
        }
      }
    ],
    "usage": {
      "promptTokens": 50,
      "completionTokens": 100,
      "totalTokens": 150,
      "cost": 0.002
    },
    "latency": 450
  }
}
```

**流式响应**：
对于流式请求（`streaming: true`），使用 Server-Sent Events (SSE)：

```http
Content-Type: text/event-stream
Transfer-Encoding: chunked
```

```text
event: message
data: {"type": "chunk", "content": "这是", "chunkId": 1}

event: message
data: {"type": "chunk", "content": "流式", "chunkId": 2}

event: message
data: {"type": "chunk", "content": "响应", "chunkId": 3}

event: complete
data: {"type": "complete", "totalChunks": 3, "usage": {...}}
```

**示例**：
```bash
# 非流式请求
curl -X POST http://localhost:8080/api/v1/agents/process \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "agentType": "general",
    "message": "北京天气怎么样？"
  }'

# 流式请求
curl -X POST http://localhost:8080/api/v1/agents/process \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -H "Accept: text/event-stream" \
  -d '{
    "agentType": "general",
    "message": "写一篇关于人工智能的文章",
    "options": {
      "streaming": true,
      "maxTokens": 500
    }
  }'
```

### 4. 创建自定义 Agent

**接口**：`POST /agents`

**功能**：创建新的自定义 Agent。

**请求体**：
```json
{
  "type": "string，必填，Agent类型（需唯一）",
  "name": "string，必填，显示名称",
  "description": "string，可选，描述",
  "config": {
    "model": "string，可选，模型名称",
    "temperature": 0.7,
    "maxTokens": 1000,
    "systemPrompt": "string，可选，系统提示词",
    "tools": ["array", "可选，默认工具列表"],
    "memoryConfig": {
      "maxHistory": 10,
      "strategy": "sliding"
    }
  }
}
```

**响应**：返回创建的 Agent 详情。

### 5. 更新 Agent

**接口**：`PUT /agents/{agentId}`

**功能**：更新 Agent 配置。

**请求体**：同创建接口，部分字段可选。

### 6. 删除 Agent

**接口**：`DELETE /agents/{agentId}`

**功能**：删除指定 Agent。

**响应**：
```json
{
  "success": true,
  "message": "Agent删除成功"
}
```

### 7. Agent 统计信息

**接口**：`GET /agents/{agentId}/metrics`

**功能**：获取 Agent 的性能指标和使用统计。

**查询参数**：
- `timeRange`：时间范围，如 `24h`、`7d`、`30d`
- `granularity`：粒度，如 `hour`、`day`、`week`

**响应**：
```json
{
  "success": true,
  "data": {
    "summary": {
      "totalRequests": 1500,
      "successRate": 99.8,
      "avgResponseTime": 450,
      "totalTokens": 150000,
      "totalCost": 15.50
    },
    "timeSeries": [
      {
        "timestamp": "2026-04-13T08:00:00Z",
        "requests": 120,
        "successful": 119,
        "avgResponseTime": 430,
        "tokens": 12000
      }
    ],
    "toolUsage": [
      {
        "toolName": "get-weather",
        "count": 450,
        "successRate": 99.5,
        "avgLatency": 320
      }
    ]
  }
}
```

## 工具管理接口

### 1. 获取工具列表

**接口**：`GET /tools`

**功能**：获取所有注册的工具。

**查询参数**：
- `category`：工具分类过滤
- `requiresAuth`：是否需要认证
- `enabled`：是否启用

**响应**：
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "name": "get-weather",
        "description": "查询指定城市的天气信息",
        "category": "weather",
        "version": "1.0.0",
        "requiresAuth": false,
        "enabled": true,
        "parameters": {
          "city": "城市名称，字符串，必填",
          "unit": "温度单位，celsius/fahrenheit，默认celsius"
        },
        "lastUpdated": "2026-04-13T08:00:00Z"
      }
    ],
    "total": 15
  }
}
```

### 2. 获取工具详情

**接口**：`GET /tools/{toolName}`

**功能**：获取工具的详细信息和模式定义。

### 3. 执行工具

**接口**：`POST /tools/{toolName}/execute`

**功能**：直接执行工具（绕过 Agent）。

**请求体**：
```json
{
  "parameters": {
    // 工具参数
  },
  "metadata": {
    "sessionId": "session-123",
    "userId": "user-123",
    "requestId": "req-456"
  }
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    // 工具执行结果
  },
  "latency": 120
}
```

### 4. 注册新工具

**接口**：`POST /tools`

**功能**：动态注册新工具。

**请求体**：
```json
{
  "name": "string，必填，工具名称",
  "description": "string，必填，工具描述",
  "className": "string，必填，工具类名",
  "config": {
    // 工具配置
  },
  "parameters": {
    "param1": "参数1描述",
    "param2": "参数2描述"
  }
}
```

### 5. 更新工具

**接口**：`PUT /tools/{toolName}`

**功能**：更新工具配置。

### 6. 禁用/启用工具

**接口**：`PUT /tools/{toolName}/status`

**功能**：更新工具启用状态。

**请求体**：
```json
{
  "enabled": false
}
```

## 工作流接口

### 1. 获取工作流列表

**接口**：`GET /workflows`

**功能**：获取所有定义的工作流。

**查询参数**：
- `status`：状态过滤（draft、active、archived）
- `tags`：标签过滤
- `createdBy`：创建者过滤

### 2. 创建工作流

**接口**：`POST /workflows`

**功能**：创建新的工作流定义。

**请求体**：
```json
{
  "name": "string，必填，工作流名称",
  "description": "string，可选，描述",
  "version": "1.0.0",
  "definition": {
    // 工作流定义（YAML或JSON）
  },
  "tags": ["tag1", "tag2"],
  "timeout": 300000,
  "retryPolicy": {
    "maxAttempts": 3,
    "backoffMs": 1000
  }
}
```

### 3. 执行工作流

**接口**：`POST /workflows/{workflowId}/execute`

**功能**：执行指定的工作流。

**请求体**：
```json
{
  "parameters": {
    // 工作流输入参数
  },
  "options": {
    "async": false,
    "timeout": 300000,
    "priority": "normal",
    "callbackUrl": "https://example.com/callback"
  }
}
```

**同步响应**：
```json
{
  "success": true,
  "data": {
    "executionId": "exec-123",
    "status": "completed",
    "result": {
      // 工作流执行结果
    },
    "steps": [
      {
        "stepId": "step-1",
        "status": "completed",
        "result": {...},
        "duration": 1200
      }
    ],
    "startTime": "2026-04-13T09:00:00Z",
    "endTime": "2026-04-13T09:00:05Z",
    "duration": 5000
  }
}
```

**异步响应**：
当 `async: true` 时，立即返回执行 ID：
```json
{
  "success": true,
  "data": {
    "executionId": "exec-123",
    "statusUrl": "/api/v1/workflows/executions/exec-123"
  }
}
```

### 4. 获取执行状态

**接口**：`GET /workflows/executions/{executionId}`

**功能**：获取工作流执行状态和结果。

### 5. 取消执行

**接口**：`POST /workflows/executions/{executionId}/cancel`

**功能**：取消正在执行的工作流。

## 会话管理接口

### 1. 创建会话

**接口**：`POST /sessions`

**功能**：创建新的对话会话。

**请求体**：
```json
{
  "agentType": "string，可选，默认Agent类型",
  "context": {
    // 初始上下文信息
  },
  "metadata": {
    "userId": "user-123",
    "channel": "web",
    "tags": ["customer-service"]
  }
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    "sessionId": "session-123",
    "createdAt": "2026-04-13T09:00:00Z",
    "agentType": "general",
    "context": {...},
    "metadata": {...}
  }
}
```

### 2. 获取会话详情

**接口**：`GET /sessions/{sessionId}`

**功能**：获取会话详情和消息历史。

**查询参数**：
- `includeMessages`：是否包含消息历史，默认 true
- `messageLimit`：消息数量限制，默认 50

### 3. 发送消息

**接口**：`POST /sessions/{sessionId}/messages`

**功能**：向会话发送消息。

**请求体**：
```json
{
  "message": "string，必填，消息内容",
  "role": "user", // user 或 assistant
  "metadata": {
    // 消息元数据
  }
}
```

### 4. 获取会话历史

**接口**：`GET /sessions/{sessionId}/history`

**功能**：获取会话的历史消息。

### 5. 清除会话

**接口**：`DELETE /sessions/{sessionId}`

**功能**：删除会话及其所有历史。

## 向量存储接口

### 1. 创建集合

**接口**：`POST /vectors/collections`

**功能**：创建新的向量集合。

**请求体**：
```json
{
  "name": "string，必填，集合名称",
  "description": "string，可选，描述",
  "dimension": 1536,
  "metric": "cosine",
  "indexConfig": {
    "type": "hnsw",
    "m": 16,
    "efConstruction": 200
  }
}
```

### 2. 插入向量

**接口**：`POST /vectors/collections/{collectionName}/insert`

**功能**：向集合中插入向量。

**请求体**：
```json
{
  "vectors": [
    {
      "id": "vec-001",
      "vector": [0.1, 0.2, ...],
      "metadata": {
        "text": "文档内容",
        "source": "file.pdf",
        "page": 1
      }
    }
  ]
}
```

### 3. 搜索向量

**接口**：`POST /vectors/collections/{collectionName}/search`

**功能**：在集合中搜索相似向量。

**请求体**：
```json
{
  "vector": [0.1, 0.2, ...],
  "topK": 10,
  "filter": {
    "source": "file.pdf"
  },
  "includeMetadata": true
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    "results": [
      {
        "id": "vec-001",
        "score": 0.95,
        "metadata": {
          "text": "文档内容",
          "source": "file.pdf"
        }
      }
    ],
    "searchTime": 45
  }
}
```

### 4. 删除向量

**接口**：`DELETE /vectors/collections/{collectionName}/vectors`

**功能**：删除指定向量。

**请求体**：
```json
{
  "ids": ["vec-001", "vec-002"]
}
```

## 系统管理接口

### 1. 系统状态

**接口**：`GET /system/health`

**功能**：获取系统健康状态。

**响应**：
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "components": {
      "database": {
        "status": "healthy",
        "latency": 12
      },
      "cache": {
        "status": "healthy",
        "latency": 2
      },
      "vectorStore": {
        "status": "healthy",
        "latency": 45
      }
    },
    "uptime": "7d 3h 25m",
    "version": "1.0.0",
    "timestamp": "2026-04-13T09:00:00Z"
  }
}
```

### 2. 系统指标

**接口**：`GET /system/metrics`

**功能**：获取系统性能指标。

**响应**：
```json
{
  "success": true,
  "data": {
    "cpu": {
      "usage": 35.5,
      "cores": 8
    },
    "memory": {
      "used": 2048,
      "total": 8192,
      "usage": 25.0
    },
    "disk": {
      "used": 10240,
      "total": 51200,
      "usage": 20.0
    },
    "requests": {
      "total": 15000,
      "rate": 5.2,
      "errorRate": 0.1
    },
    "agents": {
      "active": 5,
      "total": 15
    },
    "tools": {
      "registered": 25,
      "enabled": 20
    }
  }
}
```

### 3. 系统日志

**接口**：`GET /system/logs`

**功能**：获取系统日志。

**查询参数**：
- `level`：日志级别（error、warn、info、debug）
- `component`：组件名称
- `from`：开始时间（ISO格式）
- `to`：结束时间（ISO格式）
- `limit`：数量限制，默认 100

### 4. 系统配置

**接口**：`GET /system/config`

**功能**：获取系统配置（非敏感信息）。

**接口**：`PUT /system/config`

**功能**：更新系统配置。

## 插件管理接口

### 1. 获取插件列表

**接口**：`GET /plugins`

**功能**：获取所有已安装的插件。

### 2. 安装插件

**接口**：`POST /plugins`

**功能**：安装新插件。

**请求体**：
```json
{
  "name": "string，必填，插件名称",
  "version": "string，必填，版本",
  "jarUrl": "string，可选，JAR文件URL",
  "jarFile": "string，可选，base64编码的JAR文件",
  "config": {
    // 插件配置
  }
}
```

### 3. 启用/禁用插件

**接口**：`PUT /plugins/{pluginId}/status`

**功能**：更新插件启用状态。

### 4. 卸载插件

**接口**：`DELETE /plugins/{pluginId}`

**功能**：卸载指定插件。

## 错误处理

### 标准错误响应格式

所有 API 接口使用统一的错误响应格式：

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "人类可读的错误描述",
    "details": {
      // 错误详情，开发者使用
    },
    "traceId": "trace-123", // 用于分布式追踪
    "timestamp": "2026-04-13T09:01:00Z"
  }
}
```

### 常见错误码

| 错误码 | HTTP 状态 | 说明 |
|--------|-----------|------|
| `BAD_REQUEST` | 400 | 请求格式错误或参数无效 |
| `UNAUTHORIZED` | 401 | 认证失败或令牌无效 |
| `FORBIDDEN` | 403 | 权限不足 |
| `NOT_FOUND` | 404 | 资源不存在 |
| `CONFLICT` | 409 | 资源冲突（如重复创建） |
| `TOO_MANY_REQUESTS` | 429 | 请求过于频繁 |
| `INTERNAL_ERROR` | 500 | 服务器内部错误 |
| `SERVICE_UNAVAILABLE` | 503 | 服务暂时不可用 |
| `GATEWAY_TIMEOUT` | 504 | 网关超时 |

### 详细错误码表

#### 认证相关 (AUTH_*)
- `AUTH_001`：无效的用户名或密码
- `AUTH_002`：令牌已过期
- `AUTH_003`：令牌无效
- `AUTH_004`：缺少认证信息
- `AUTH_005`：权限不足

#### 验证相关 (VALID_*)
- `VALID_001`：必填字段缺失
- `VALID_002`：字段格式错误
- `VALID_003`：字段值超出范围
- `VALID_004`：数据验证失败

#### 资源相关 (RESOURCE_*)
- `RESOURCE_001`：资源不存在
- `RESOURCE_002`：资源已存在
- `RESOURCE_003`：资源状态不允许此操作
- `RESOURCE_004`：资源访问被拒绝

#### 业务逻辑 (BUSINESS_*)
- `BUSINESS_001`：业务规则冲突
- `BUSINESS_002`：工作流执行失败
- `BUSINESS_003`：工具执行失败
- `BUSINESS_004`：Agent 执行失败

#### 系统错误 (SYSTEM_*)
- `SYSTEM_001`：数据库错误
- `SYSTEM_002`：缓存错误
- `SYSTEM_003`：外部服务错误
- `SYSTEM_004`：配置错误

## 限流策略

### 默认限流规则

| 接口类别 | 限制 | 说明 |
|----------|------|------|
| 认证接口 | 10 请求/分钟/IP | 防止暴力破解 |
| Agent 接口 | 100 请求/分钟/用户 | 普通用户限制 |
| 工具接口 | 50 请求/分钟/用户 | 工具调用限制 |
| 管理接口 | 30 请求/分钟/用户 | 管理操作限制 |
| 工作流接口 | 20 请求/分钟/用户 | 工作流执行限制 |

### 限流响应头

当触发限流时，响应包含以下头部：

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1703275200
Retry-After: 60
```

### 自定义限流

对于特定需求，可以通过配置自定义限流规则：

```yaml
rateLimits:
  custom:
    - pattern: "/api/v1/agents/process"
      limit: 200
      period: "1m"
      perUser: true
    - pattern: "/api/v1/tools/*/execute"
      limit: 100
      period: "1m"
      perUser: true
```

## 最佳实践

### 1. 认证和授权
- 始终使用 HTTPS 传输敏感信息
- 定期轮换访问令牌和刷新令牌
- 使用最小权限原则分配角色和权限
- 记录所有敏感操作日志

### 2. 请求优化
- 使用批量接口减少请求次数
- 合理设置超时时间，避免长时间阻塞
- 使用流式接口处理大文本生成
- 启用压缩减少网络传输量

### 3. 错误处理
- 实现指数退避重试机制
- 添加熔断器防止级联故障
- 使用降级策略保证核心功能
- 监控错误率并设置告警

### 4. 性能监控
- 监控接口响应时间和成功率
- 设置关键指标的 SLO/SLA
- 定期进行负载测试和性能优化
- 使用 APM 工具进行分布式追踪

### 5. 安全考虑
- 对所有输入进行验证和清理
- 实施输出编码防止 XSS
- 限制敏感接口的访问频率
- 定期进行安全审计和渗透测试

## 客户端实现示例

### Python 客户端

```python
import requests
from typing import Optional, Dict, Any

class AgentXClient:
    def __init__(self, base_url: str, api_key: Optional[str] = None):
        self.base_url = base_url.rstrip('/')
        self.api_key = api_key
        self.session = requests.Session()
        
        if api_key:
            self.session.headers.update({
                'Authorization': f'Bearer {api_key}'
            })
    
    def login(self, username: str, password: str) -> Dict[str, Any]:
        """用户登录"""
        response = self.session.post(
            f'{self.base_url}/api/v1/auth/login',
            json={'username': username, 'password': password}
        )
        response.raise_for_status()
        data = response.json()
        
        if data['success']:
            token = data['data']['token']
            self.session.headers.update({
                'Authorization': f'Bearer {token}'
            })
        
        return data['data']
    
    def process_message(self, agent_type: str, message: str, **kwargs) -> Dict[str, Any]:
        """发送消息给 Agent"""
        payload = {
            'agentType': agent_type,
            'message': message,
            **kwargs
        }
        
        response = self.session.post(
            f'{self.base_url}/api/v1/agents/process',
            json=payload
        )
        response.raise_for_status()
        return response.json()['data']
    
    def process_message_streaming(self, agent_type: str, message: str, **kwargs):
        """流式处理消息"""
        payload = {
            'agentType': agent_type,
            'message': message,
            'options': {
                'streaming': True,
                **kwargs.get('options', {})
            }
        }
        
        response = self.session.post(
            f'{self.base_url}/api/v1/agents/process',
            json=payload,
            stream=True,
            headers={'Accept': 'text/event-stream'}
        )
        
        for line in response.iter_lines():
            if line:
                # 解析 Server-Sent Events
                yield self._parse_sse_line(line)
    
    def _parse_sse_line(self, line: bytes) -> Dict[str, Any]:
        """解析 SSE 行"""
        # SSE 格式解析逻辑
        pass
```

### JavaScript/TypeScript 客户端

```typescript
interface AgentXConfig {
  baseUrl: string;
  apiKey?: string;
}

interface AgentRequest {
  agentType: string;
  message: string;
  sessionId?: string;
  parameters?: Record<string, any>;
  options?: {
    streaming?: boolean;
    temperature?: number;
    maxTokens?: number;
  };
}

class AgentXClient {
  private baseUrl: string;
  private apiKey?: string;
  
  constructor(config: AgentXConfig) {
    this.baseUrl = config.baseUrl.replace(/\/$/, '');
    this.apiKey = config.apiKey;
  }
  
  async login(username: string, password: string): Promise<any> {
    const response = await fetch(`${this.baseUrl}/api/v1/auth/login`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ username, password })
    });
    
    const data = await response.json();
    
    if (data.success) {
      this.apiKey = data.data.token;
    }
    
    return data.data;
  }
  
  async processMessage(request: AgentRequest): Promise<any> {
    const response = await fetch(`${this.baseUrl}/api/v1/agents/process`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`
      },
      body: JSON.stringify(request)
    });
    
    const data = await response.json();
    return data.data;
  }
  
  async *processMessageStreaming(request: AgentRequest): AsyncGenerator<any> {
    const response = await fetch(`${this.baseUrl}/api/v1/agents/process`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`,
        'Accept': 'text/event-stream'
      },
      body: JSON.stringify({
        ...request,
        options: {
          ...request.options,
          streaming: true
        }
      })
    });
    
    const reader = response.body?.getReader();
    const decoder = new TextDecoder();
    
    if (!reader) {
      throw new Error('Failed to get response reader');
    }
    
    while (true) {
      const { done, value } = await reader.read();
      
      if (done) {
        break;
      }
      
      const chunk = decoder.decode(value);
      const events = this.parseSSE(chunk);
      
      for (const event of events) {
        yield event;
      }
    }
  }
  
  private parseSSE(chunk: string): any[] {
    // SSE 解析逻辑
    return [];
  }
}
```

## 迁移指南

### 从 v0.x 迁移到 v1.0

#### 主要变化
1. **API 路径标准化**：所有路径前缀从 `/api` 改为 `/api/v1`
2. **响应格式统一**：统一使用 `success` 字段标识成功/失败
3. **错误码规范化**：使用标准化的错误码体系
4. **认证方式增强**：支持多种认证方式

#### 迁移步骤
1. **更新 API 端点**：将所有请求路径添加 `/v1` 前缀
2. **更新响应处理**：使用 `success` 字段判断成功
3. **更新错误处理**：使用新的错误码体系
4. **测试验证**：使用新的 API 测试所有功能

#### 兼容性支持
v1.0 提供 6 个月的向后兼容支持：
- 旧版 API 路径继续可用
- 自动重定向到新版路径
- 提供迁移工具和文档

## 下一步

根据你的需求选择：
- **API 使用者**：查看具体接口的详细说明和使用示例
- **集成开发者**：参考客户端实现示例，集成到你的应用中
- **系统管理员**：查看系统管理接口，进行系统维护

如需完整 API 规范，可下载 OpenAPI 规范文件或访问 Swagger UI。