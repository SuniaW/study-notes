# 第四部分：API 参考

本部分提供 AgentX 平台所有 API 接口的详细说明，包括 REST API、MCP 协议接口和工具调用示例。

## 章节列表

### 1. [REST API](01-rest-api.md)
- Agent 相关接口
- 工具管理接口
- 工作流执行接口
- 系统管理接口

### 2. [MCP 协议接口](02-mcp-api.md)
- MCP 协议概述
- 工具发现接口
- 工具调用接口
- 服务器状态接口

### 3. [工具调用示例](03-tool-examples.md)
- 内置工具使用示例
- 自定义工具调用
- 错误处理示例
- 性能优化建议

## API 设计原则

### 1. RESTful 设计
- **资源导向**：所有端点围绕资源设计
- **HTTP 方法**：正确使用 GET、POST、PUT、DELETE
- **状态码**：规范的 HTTP 状态码返回
- **HATEOAS**：支持超媒体作为应用状态引擎

### 2. 一致性原则
- **命名一致**：统一的命名规范和风格
- **响应格式**：统一的成功/错误响应格式
- **版本管理**：清晰的 API 版本管理策略
- **文档同步**：代码和文档保持同步

### 3. 安全性
- **认证授权**：所有接口需要认证（除健康检查）
- **输入验证**：严格的输入参数验证
- **输出过滤**：敏感信息输出时自动过滤
- **速率限制**：防止滥用和攻击

### 4. 可观测性
- **请求追踪**：每个请求有唯一追踪 ID
- **性能指标**：接口性能指标自动收集
- **错误分类**：标准化的错误分类和代码
- **使用统计**：API 使用情况统计和分析

## 基础概念

### 1. 认证方式

#### JWT 认证（默认）
```http
Authorization: Bearer <jwt_token>
```

#### API Key 认证
```http
X-API-Key: <api_key>
```

#### Basic 认证（内部使用）
```http
Authorization: Basic <base64_credentials>
```

### 2. 请求格式

#### 请求头
```http
Content-Type: application/json
Accept: application/json
X-Request-ID: <uuid>  # 可选，用于请求追踪
```

#### 请求体（JSON）
```json
{
  "param1": "value1",
  "param2": "value2"
}
```

### 3. 响应格式

#### 成功响应
```json
{
  "success": true,
  "data": {
    // 具体数据
  },
  "message": "操作成功",
  "timestamp": "2026-04-13T09:01:00Z",
  "requestId": "<request_id>"
}
```

#### 错误响应
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述",
    "details": {
      // 错误详情
    }
  },
  "timestamp": "2026-04-13T09:01:00Z",
  "requestId": "<request_id>"
}
```

### 4. 分页和排序

#### 分页参数
```http
GET /api/v1/resources?page=1&size=20
```

#### 排序参数
```http
GET /api/v1/resources?sort=createdAt,desc&sort=name,asc
```

#### 分页响应
```json
{
  "success": true,
  "data": {
    "content": [...],
    "page": 1,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8,
    "first": true,
    "last": false
  }
}
```

## API 版本管理

### 版本策略
- **URI 版本化**：`/api/v1/...`、`/api/v2/...`
- **向后兼容**：新版本尽量保持向后兼容
- **弃用通知**：提前通知即将弃用的接口
- **迁移指南**：提供版本迁移指南和工具

### 当前版本
- **v1**：当前稳定版本，生产环境使用
- **v2**：开发中，预计 2026 Q3 发布
- **beta**：实验性功能，可能变化

## 快速开始

### 1. 获取访问令牌
```bash
# 使用用户名密码获取 JWT
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "password123"
  }'

# 响应示例
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600,
    "tokenType": "Bearer"
  }
}
```

### 2. 调用 Agent 接口
```bash
# 使用 JWT 调用 Agent
curl -X POST http://localhost:8080/api/v1/agents/process \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -d '{
    "message": "你好，AgentX",
    "agentType": "general"
  }'
```

### 3. 查看 API 文档
- **Swagger UI**：http://localhost:8080/swagger-ui.html
- **OpenAPI 规范**：http://localhost:8080/v3/api-docs
- **ReDoc**：http://localhost:8080/redoc

## 错误处理

### 常见错误码

| 错误码 | HTTP 状态 | 说明 |
|--------|-----------|------|
| `AUTH_001` | 401 | 认证失败 |
| `AUTH_002` | 403 | 权限不足 |
| `VALID_001` | 400 | 参数验证失败 |
| `NOT_FOUND` | 404 | 资源不存在 |
| `CONFLICT` | 409 | 资源冲突 |
| `RATE_LIMIT` | 429 | 请求过于频繁 |
| `SERVER_ERR` | 500 | 服务器内部错误 |

### 错误处理示例
```json
{
  "success": false,
  "error": {
    "code": "VALID_001",
    "message": "参数验证失败",
    "details": {
      "field": "username",
      "reason": "不能为空",
      "value": null
    }
  }
}
```

## 性能建议

### 1. 请求优化
- **批量操作**：支持批量操作的接口尽量批量调用
- **字段过滤**：使用 `fields` 参数只返回需要的字段
- **缓存利用**：合理使用客户端缓存
- **连接复用**：保持 HTTP 连接复用

### 2. 错误处理优化
- **指数退避**：失败重试时使用指数退避算法
- **熔断机制**：连续失败时暂时停止调用
- **降级策略**：核心功能不可用时提供降级方案

### 3. 监控和调优
- **性能监控**：监控接口响应时间和成功率
- **容量规划**：基于监控数据的容量规划
- **A/B 测试**：新功能的 A/B 测试和效果评估

## 限流策略

### 限流规则
- **默认限制**：每个用户 100 请求/分钟
- **认证请求**：登录接口 10 请求/分钟
- **工具调用**：高风险工具单独限流
- **工作流执行**：复杂工作流单独限流

### 限流响应
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1703275200
```

## 安全建议

### 1. 传输安全
- **始终使用 HTTPS**：生产环境必须使用 HTTPS
- **证书验证**：验证服务器证书有效性
- **TLS 版本**：使用 TLS 1.3 或更高版本

### 2. 密钥管理
- **环境变量**：API 密钥存储在环境变量中
- **定期轮换**：定期轮换 API 密钥和令牌
- **访问日志**：记录所有 API 密钥使用情况

### 3. 输入验证
- **类型检查**：验证参数类型和格式
- **长度限制**：限制字符串和数组长度
- **内容过滤**：过滤恶意内容和脚本

## 下一步

根据你的需求选择章节：

- **API 使用者**：从 [REST API](01-rest-api.md) 开始，了解具体接口
- **工具开发者**：查看 [MCP 协议接口](02-mcp-api.md)，了解工具集成标准
- **示例学习者**：直接看 [工具调用示例](03-tool-examples.md)，快速上手

如需完整 API 列表，可访问 Swagger UI 或下载 OpenAPI 规范文件。