# MCP 协议接口

本章详细说明 AgentX 平台的 Model Context Protocol (MCP) 接口，包括协议规范、工具发现、资源管理和服务器配置。

## MCP 概述

### 什么是 MCP？

Model Context Protocol (MCP) 是一个开放协议，用于在 AI 模型和外部工具/资源之间建立标准化连接。AgentX 实现了完整的 MCP 服务器，允许：

1. **工具发现**：AI 模型动态发现可用工具
2. **工具调用**：安全地执行工具并获取结果
3. **资源管理**：访问和管理外部数据资源
4. **上下文提供**：为模型提供相关上下文信息

### 协议特点

- **双向通信**：支持服务器推送和客户端拉取
- **强类型**：基于 JSON Schema 的严格类型定义
- **可扩展**：支持自定义工具和资源类型
- **安全**：细粒度的权限控制和审计

### MCP 版本

AgentX 支持以下 MCP 版本：
- **MCP v1.0**：基础版本，支持工具调用
- **MCP v2.0**：当前稳定版本，支持资源和上下文
- **MCP v2.1**：最新版本，增强的流式支持

## MCP 服务器配置

### 启动配置

AgentX MCP 服务器可以通过以下方式启动：

#### 1. 命令行启动
```bash
# 启动 MCP 服务器
java -jar agentx-server.jar \
  --mcp.enabled=true \
  --mcp.port=8081 \
  --mcp.transport=stdio \
  --mcp.allowed-origins="*"
```

#### 2. Docker 启动
```bash
docker run -p 8081:8081 \
  -e MCP_ENABLED=true \
  -e MCP_PORT=8081 \
  agentx/agentx-server:latest
```

#### 3. 配置文件
```yaml
# application.yml
mcp:
  enabled: true
  server:
    port: 8081
    transport:
      - stdio
      - sse
      - websocket
    auth:
      enabled: true
      type: jwt
      required-scopes:
        - tools:read
        - tools:execute
  clients:
    claude-desktop:
      allowed: true
      scopes: ["tools:*"]
    cursor:
      allowed: true
      scopes: ["tools:read", "tools:execute"]
```

### 传输协议

AgentX 支持多种 MCP 传输协议：

#### 1. stdio（标准输入输出）
- **用途**：命令行工具集成
- **特点**：简单、无网络依赖
- **配置**：
  ```json
  {
    "command": "java",
    "args": ["-jar", "agentx-server.jar", "--mcp.transport=stdio"],
    "env": {
      "MCP_ENABLED": "true"
    }
  }
  ```

#### 2. SSE（Server-Sent Events）
- **用途**：浏览器和 HTTP 客户端
- **特点**：单向流式，客户端拉取
- **端点**：`http://localhost:8081/mcp/sse`

#### 3. WebSocket
- **用途**：实时双向通信
- **特点**：全双工，低延迟
- **端点**：`ws://localhost:8081/mcp/ws`

#### 4. HTTP
- **用途**：RESTful 风格的请求/响应
- **特点**：兼容性好，易于调试
- **端点**：`http://localhost:8081/mcp/http`

## MCP 协议消息

### 消息格式

所有 MCP 消息遵循以下 JSON 格式：

```json
{
  "jsonrpc": "2.0",
  "id": "unique-request-id",
  "method": "method-name",
  "params": {
    // 方法参数
  }
}
```

### 请求/响应流

```
客户端请求 → 服务器处理 → 服务器响应
      │           │           │
      ▼           ▼           ▼
  初始化 → 工具发现 → 工具调用 → 结果返回
```

### 消息类型

#### 1. 初始化消息
```json
{
  "jsonrpc": "2.0",
  "id": "init-1",
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {
        "listChanged": true
      },
      "resources": {
        "subscribe": true
      }
    },
    "clientInfo": {
      "name": "claude-desktop",
      "version": "1.0.0"
    }
  }
}
```

#### 2. 成功响应
```json
{
  "jsonrpc": "2.0",
  "id": "init-1",
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {
        "listChanged": true
      },
      "resources": {
        "subscribe": true
      }
    },
    "serverInfo": {
      "name": "agentx-mcp-server",
      "version": "1.0.0"
    }
  }
}
```

#### 3. 错误响应
```json
{
  "jsonrpc": "2.0",
  "id": "init-1",
  "error": {
    "code": -32601,
    "message": "Method not found",
    "data": {
      "method": "unknown-method"
    }
  }
}
```

## 工具发现接口

### 1. 列出所有工具

**方法**：`tools/list`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-list-1",
  "method": "tools/list"
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-list-1",
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name, e.g. 'San Francisco'"
            },
            "unit": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "Temperature unit",
              "default": "celsius"
            }
          },
          "required": ["location"]
        }
      },
      {
        "name": "calculate",
        "description": "Perform mathematical calculations",
        "inputSchema": {
          "type": "object",
          "properties": {
            "expression": {
              "type": "string",
              "description": "Mathematical expression, e.g. '2 + 3 * 4'"
            }
          },
          "required": ["expression"]
        }
      }
    ]
  }
}
```

### 2. 工具列表变更通知

当工具列表发生变化时，服务器会主动通知客户端：

**通知**：
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed",
  "params": {}
}
```

### 3. 获取工具详情

**方法**：`tools/describe`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-describe-1",
  "method": "tools/describe",
  "params": {
    "toolName": "get_weather"
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-describe-1",
  "result": {
    "name": "get_weather",
    "description": "Get current weather for a location",
    "inputSchema": {
      // 完整的 JSON Schema
    },
    "outputSchema": {
      "type": "object",
      "properties": {
        "location": {
          "type": "string"
        },
        "temperature": {
          "type": "number"
        },
        "unit": {
          "type": "string"
        },
        "condition": {
          "type": "string"
        }
      }
    },
    "examples": [
      {
        "input": {
          "location": "Beijing",
          "unit": "celsius"
        },
        "output": {
          "location": "Beijing",
          "temperature": 25.0,
          "unit": "celsius",
          "condition": "Sunny"
        }
      }
    ]
  }
}
```

## 工具调用接口

### 1. 调用工具

**方法**：`tools/call`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-call-1",
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "San Francisco",
      "unit": "celsius"
    }
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-call-1",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Current weather in San Francisco: 18°C, Partly Cloudy"
      }
    ],
    "isError": false,
    "metadata": {
      "toolName": "get_weather",
      "executionTime": 120,
      "timestamp": "2026-04-13T09:01:00Z"
    }
  }
}
```

### 2. 流式工具调用

对于长时间运行的工具，支持流式响应：

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-call-2",
  "method": "tools/call",
  "params": {
    "name": "analyze_document",
    "arguments": {
      "documentUrl": "https://example.com/report.pdf"
    },
    "stream": true
  }
}
```

**流式响应**：
```json
// 第一个 chunk
{
  "jsonrpc": "2.0",
  "id": "tools-call-2",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "正在分析文档..."
      }
    ],
    "isError": false,
    "stream": true,
    "index": 0
  }
}

// 第二个 chunk
{
  "jsonrpc": "2.0",
  "id": "tools-call-2",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "文档分析完成，共 10 页"
      }
    ],
    "isError": false,
    "stream": true,
    "index": 1
  }
}

// 最后一个 chunk
{
  "jsonrpc": "2.0",
  "id": "tools-call-2",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "分析结果：..."
      }
    ],
    "isError": false,
    "stream": false,
    "index": 2
  }
}
```

### 3. 批量工具调用

**方法**：`tools/call_batch`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-call-batch-1",
  "method": "tools/call_batch",
  "params": {
    "calls": [
      {
        "name": "get_weather",
        "arguments": {
          "location": "Beijing"
        }
      },
      {
        "name": "calculate",
        "arguments": {
          "expression": "2 + 3 * 4"
        }
      }
    ]
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "tools-call-batch-1",
  "result": {
    "results": [
      {
        "content": [
          {
            "type": "text",
            "text": "Beijing weather: 25°C, Sunny"
          }
        ],
        "isError": false
      },
      {
        "content": [
          {
            "type": "text",
            "text": "Result: 14"
          }
        ],
        "isError": false
      }
    ]
  }
}
```

## 资源管理接口

### 1. 列出资源

**方法**：`resources/list`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "resources-list-1",
  "method": "resources/list"
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "resources-list-1",
  "result": {
    "resources": [
      {
        "uri": "file:///docs/api-reference.md",
        "name": "API Reference",
        "description": "API 参考文档",
        "mimeType": "text/markdown"
      },
      {
        "uri": "file:///data/customers.csv",
        "name": "Customer Data",
        "description": "客户数据表",
        "mimeType": "text/csv"
      },
      {
        "uri": "agentx://knowledge-base/faq",
        "name": "FAQ",
        "description": "常见问题解答",
        "mimeType": "application/json"
      }
    ]
  }
}
```

### 2. 读取资源

**方法**：`resources/read`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "resources-read-1",
  "method": "resources/read",
  "params": {
    "uri": "file:///docs/api-reference.md"
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "resources-read-1",
  "result": {
    "contents": [
      {
        "uri": "file:///docs/api-reference.md",
        "mimeType": "text/markdown",
        "text": "# API 参考\n\n本文档详细说明 AgentX 平台的所有 API 接口..."
      }
    ]
  }
}
```

### 3. 订阅资源变更

**方法**：`resources/subscribe`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "resources-subscribe-1",
  "method": "resources/subscribe",
  "params": {
    "uris": [
      "file:///docs/api-reference.md",
      "agentx://knowledge-base/faq"
    ]
  }
}
```

**资源变更通知**：
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///docs/api-reference.md",
    "changeType": "modified"
  }
}
```

### 4. 搜索资源

**方法**：`resources/search`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "resources-search-1",
  "method": "resources/search",
  "params": {
    "query": "API authentication",
    "limit": 10
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "resources-search-1",
  "result": {
    "results": [
      {
        "uri": "file:///docs/api-reference.md",
        "name": "API Reference",
        "snippet": "... API authentication methods include JWT and API key ...",
        "score": 0.95
      },
      {
        "uri": "file:///docs/security.md",
        "name": "Security Guide",
        "snippet": "... Authentication and authorization best practices ...",
        "score": 0.82
      }
    ]
  }
}
```

## 提示词模板接口

### 1. 列出提示词模板

**方法**：`prompts/list`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "prompts-list-1",
  "method": "prompts/list"
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "prompts-list-1",
  "result": {
    "prompts": [
      {
        "name": "code_review",
        "description": "代码审查助手",
        "arguments": [
          {
            "name": "code",
            "description": "要审查的代码",
            "required": true
          },
          {
            "name": "language",
            "description": "编程语言",
            "required": false
          }
        ]
      },
      {
        "name": "document_summary",
        "description": "文档总结助手",
        "arguments": [
          {
            "name": "document",
            "description": "文档内容",
            "required": true
          },
          {
            "name": "max_length",
            "description": "总结最大长度",
            "required": false
          }
        ]
      }
    ]
  }
}
```

### 2. 获取提示词

**方法**：`prompts/get`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "prompts-get-1",
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "code": "function add(a, b) { return a + b; }",
      "language": "javascript"
    }
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "prompts-get-1",
  "result": {
    "messages": [
      {
        "role": "system",
        "content": "你是一个专业的代码审查助手。请审查以下代码，指出潜在问题并提供改进建议。"
      },
      {
        "role": "user",
        "content": "请审查以下 JavaScript 代码：\n\n```javascript\nfunction add(a, b) { return a + b; }\n```\n\n请提供审查意见。"
      }
    ],
    "description": "代码审查助手"
  }
}
```

## 日志和追踪接口

### 1. 设置日志级别

**方法**：`logging/set_level`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "logging-set-level-1",
  "method": "logging/set_level",
  "params": {
    "level": "debug",
    "scope": "tools"
  }
}
```

### 2. 获取日志

**方法**：`logging/get`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "logging-get-1",
  "method": "logging/get",
  "params": {
    "limit": 100,
    "level": "info"
  }
}
```

### 3. 追踪工具调用

**方法**：`tracing/enable`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "tracing-enable-1",
  "method": "tracing/enable",
  "params": {
    "enabled": true,
    "samplingRate": 0.1
  }
}
```

## 服务器管理接口

### 1. 获取服务器信息

**方法**：`server/info`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "server-info-1",
  "method": "server/info"
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "server-info-1",
  "result": {
    "name": "agentx-mcp-server",
    "version": "1.0.0",
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {
        "listChanged": true,
        "call": true,
        "callStreaming": true,
        "callBatch": true
      },
      "resources": {
        "list": true,
        "read": true,
        "subscribe": true,
        "search": true
      },
      "prompts": {
        "list": true,
        "get": true
      }
    },
    "uptime": 86400,
    "connectedClients": 3,
    "toolCount": 25,
    "resourceCount": 100
  }
}
```

### 2. 健康检查

**方法**：`server/health`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "server-health-1",
  "method": "server/health"
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "server-health-1",
  "result": {
    "status": "healthy",
    "components": {
      "database": {
        "status": "healthy",
        "latency": 15
      },
      "cache": {
        "status": "healthy",
        "latency": 2
      },
      "tools": {
        "status": "healthy",
        "available": 25,
        "enabled": 20
      }
    },
    "timestamp": "2026-04-13T09:01:00Z"
  }
}
```

### 3. 服务器统计

**方法**：`server/stats`

**请求**：
```json
{
  "jsonrpc": "2.0",
  "id": "server-stats-1",
  "method": "server/stats",
  "params": {
    "timeRange": "24h",
    "granularity": "hour"
  }
}
```

**响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "server-stats-1",
  "result": {
    "toolCalls": {
      "total": 1500,
      "successful": 1490,
      "failed": 10,
      "successRate": 99.33,
      "avgLatency": 120
    },
    "resourceAccess": {
      "reads": 450,
      "searches": 120,
      "subscriptions": 25
    },
    "clientConnections": {
      "total": 3,
      "active": 3,
      "disconnected": 2
    },
    "timeSeries": [
      {
        "timestamp": "2026-04-13T08:00:00Z",
        "toolCalls": 120,
        "avgLatency": 115
      }
    ]
  }
}
```

## 错误处理

### MCP 错误码

| 错误码 | 含义 | 说明 |
|--------|------|------|
| -32600 | 无效请求 | JSON-RPC 请求格式错误 |
| -32601 | 方法不存在 | 请求的方法未实现 |
| -32602 | 无效参数 | 参数验证失败 |
| -32603 | 内部错误 | 服务器内部错误 |
| -32700 | 解析错误 | JSON 解析失败 |
| -32800 | 服务器错误 | 服务器处理错误 |
| -32900 | 工具错误 | 工具执行错误 |
| -32901 | 工具不存在 | 请求的工具未找到 |
| -32902 | 工具参数错误 | 工具参数验证失败 |
| -32903 | 工具执行超时 | 工具执行超时 |
| -33000 | 资源错误 | 资源访问错误 |
| -33001 | 资源不存在 | 请求的资源未找到 |
| -33002 | 资源访问拒绝 | 没有资源访问权限 |
| -33100 | 认证错误 | 认证失败 |
| -33101 | 权限不足 | 没有执行操作的权限 |

### 错误响应示例

```json
{
  "jsonrpc": "2.0",
  "id": "tools-call-1",
  "error": {
    "code": -32902,
    "message": "Invalid tool arguments",
    "data": {
      "toolName": "get_weather",
      "errors": [
        {
          "field": "location",
          "message": "Location is required",
          "code": "REQUIRED"
        }
      ]
    }
  }
}
```

## 客户端集成示例

### 1. Claude Desktop 集成

#### 配置文件
```json
{
  "mcpServers": {
    "agentx": {
      "command": "java",
      "args": [
        "-jar",
        "/path/to/agentx-server.jar",
        "--mcp.transport=stdio"
      ],
      "env": {
        "MCP_ENABLED": "true",
        "MCP_AUTH_TYPE": "jwt",
        "MCP_AUTH_TOKEN": "${AGENTX_MCP_TOKEN}"
      }
    }
  }
}
```

#### 环境变量
```bash
export AGENTX_MCP_TOKEN="eyJhbGciOiJIUzI1NiIs..."
```

### 2. Cursor IDE 集成

#### 配置文件
```json
{
  "mcpServers": {
    "agentx-tools": {
      "command": "npx",
      "args": [
        "-y",
        "@agentx/mcp-client",
        "--server",
        "http://localhost:8081/mcp/sse",
        "--token",
        "${AGENTX_API_KEY}"
      ]
    }
  }
}
```

### 3. 自定义客户端实现

#### Python 客户端
```python
import json
import subprocess
from typing import Dict, Any, Generator

class MCPClient:
    def __init__(self, server_command: list):
        self.process = subprocess.Popen(
            server_command,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            bufsize=1
        )
    
    def send_request(self, method: str, params: Dict[str, Any]) -> Dict[str, Any]:
        request = {
            "jsonrpc": "2.0",
            "id": self._generate_id(),
            "method": method,
            "params": params
        }
        
        # 发送请求
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()
        
        # 读取响应
        response_line = self.process.stdout.readline()
        return json.loads(response_line)
    
    def list_tools(self) -> Dict[str, Any]:
        return self.send_request("tools/list", {})
    
    def call_tool(self, tool_name: str, arguments: Dict[str, Any]) -> Dict[str, Any]:
        return self.send_request("tools/call", {
            "name": tool_name,
            "arguments": arguments
        })
    
    def _generate_id(self) -> str:
        import uuid
        return str(uuid.uuid4())
    
    def close(self):
        self.process.terminate()
        self.process.wait()

# 使用示例
client = MCPClient([
    "java", "-jar", "agentx-server.jar",
    "--mcp.transport=stdio"
])

try:
    # 列出所有工具
    tools_response = client.list_tools()
    print("可用工具:", tools_response["result"]["tools"])
    
    # 调用工具
    weather_response = client.call_tool("get_weather", {
        "location": "Beijing",
        "unit": "celsius"
    })
    print("天气信息:", weather_response["result"]["content"])
finally:
    client.close()
```

#### JavaScript 客户端
```javascript
import { EventSource } from 'eventsource';

class MCPClient {
  constructor(serverUrl, apiKey) {
    this.serverUrl = serverUrl;
    this.apiKey = apiKey;
    this.requestId = 0;
    this.callbacks = new Map();
  }
  
  async connect() {
    // 使用 SSE 连接
    this.eventSource = new EventSource(`${this.serverUrl}/mcp/sse`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });
    
    this.eventSource.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this.handleMessage(message);
    };
    
    // 等待连接建立
    return new Promise((resolve) => {
      this.eventSource.onopen = () => {
        resolve();
      };
    });
  }
  
  async sendRequest(method, params) {
    const requestId = ++this.requestId;
    const request = {
      jsonrpc: '2.0',
      id: requestId.toString(),
      method,
      params
    };
    
    // 发送请求（通过单独的 HTTP 请求）
    const response = await fetch(`${this.serverUrl}/mcp/http`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`
      },
      body: JSON.stringify(request)
    });
    
    return response.json();
  }
  
  async listTools() {
    return this.sendRequest('tools/list', {});
  }
  
  async callTool(toolName, arguments) {
    return this.sendRequest('tools/call', {
      name: toolName,
      arguments
    });
  }
  
  handleMessage(message) {
    // 处理服务器推送的消息
    if (message.method === 'notifications/tools/list_changed') {
      console.log('工具列表已更新');
      this.onToolsChanged?.();
    }
  }
  
  disconnect() {
    this.eventSource?.close();
  }
}

// 使用示例
const client = new MCPClient('http://localhost:8081', 'your-api-key');

async function main() {
  await client.connect();
  
  // 列出工具
  const toolsResponse = await client.listTools();
  console.log('可用工具:', toolsResponse.result.tools);
  
  // 调用工具
  const weatherResponse = await client.callTool('get_weather', {
    location: 'Beijing',
    unit: 'celsius'
  });
  console.log('天气信息:', weatherResponse.result.content);
  
  client.disconnect();
}

main().catch(console.error);
```

## 安全配置

### 1. 认证和授权

#### JWT 认证
```yaml
mcp:
  auth:
    enabled: true
    type: jwt
    jwkSetUrl: https://auth.agentx.ai/.well-known/jwks.json
    requiredScopes:
      - mcp:tools:read
      - mcp:tools:execute
    audience: agentx-mcp-server
    issuer: https://auth.agentx.ai
```

#### API Key 认证
```yaml
mcp:
  auth:
    enabled: true
    type: api-key
    headerName: X-API-Key
    validKeys:
      - name: claude-desktop
        key: sk-xxx
        scopes: [mcp:tools:*]
      - name: cursor-ide
        key: sk-yyy
        scopes: [mcp:tools:read, mcp:tools:execute]
```

### 2. 传输安全

#### TLS/SSL 配置
```yaml
mcp:
  server:
    ssl:
      enabled: true
      keyStore: /path/to/keystore.p12
      keyStorePassword: ${SSL_KEYSTORE_PASSWORD}
      keyAlias: agentx-mcp
      protocols: TLSv1.3
```

#### CORS 配置
```yaml
mcp:
  server:
    cors:
      allowedOrigins:
        - https://claude.ai
        - https://cursor.sh
        - http://localhost:3000
      allowedMethods: [GET, POST, OPTIONS]
      allowedHeaders: [Authorization, Content-Type]
      allowCredentials: true
```

### 3. 权限控制

#### 工具级别权限
```yaml
mcp:
  tools:
    permissions:
      get_weather:
        scopes: [mcp:tools:weather]
        rateLimit: 10/分钟
      calculate:
        scopes: [mcp:tools:calculate]
        rateLimit: 50/分钟
      analyze_document:
        scopes: [mcp:tools:document]
        rateLimit: 5/分钟
        requireAuth: true
```

#### 资源级别权限
```yaml
mcp:
  resources:
    permissions:
      file:///docs/*:
        scopes: [mcp:resources:docs:read]
      file:///data/*:
        scopes: [mcp:resources:data:read]
        requireAuth: true
      agentx://knowledge-base/*:
        scopes: [mcp:resources:kb:*]
```

## 性能优化

### 1. 连接池配置
```yaml
mcp:
  server:
    connection:
      maxConnections: 100
      maxConcurrentRequests: 50
      idleTimeout: 300000
      keepAlive: true
```

### 2. 缓存配置
```yaml
mcp:
  cache:
    tools:
      enabled: true
      ttl: 300000  # 5分钟
    resources:
      enabled: true
      ttl: 60000   # 1分钟
    prompts:
      enabled: true
      ttl: 1800000 # 30分钟
```

### 3. 监控和指标
```yaml
mcp:
  metrics:
    enabled: true
    export:
      prometheus:
        enabled: true
        path: /mcp/metrics
      statsd:
        enabled: false
        host: localhost
        port: 8125
    tracing:
      enabled: true
      sampler: 0.1
      exporter: jaeger
```

## 故障排除

### 常见问题

#### 1. 连接失败
- **症状**：客户端无法连接到 MCP 服务器
- **检查**：
  1. 确认服务器正在运行：`ps aux | grep agentx-server`
  2. 确认端口监听：`netstat -tlnp | grep 8081`
  3. 检查防火墙规则
  4. 验证认证配置

#### 2. 工具调用超时
- **症状**：工具调用长时间无响应
- **检查**：
  1. 确认工具实现没有阻塞
  2. 增加超时配置：`mcp.tools.timeout: 30000`
  3. 检查外部服务依赖
  4. 查看工具执行日志

#### 3. 内存泄漏
- **症状**：服务器内存使用持续增长
- **检查**：
  1. 检查工具实现是否正确释放资源
  2. 监控连接池使用情况
  3. 启用 GC 日志分析
  4. 定期重启服务

#### 4. 性能下降
- **症状**：响应时间变慢
- **检查**：
  1. 监控系统资源（CPU、内存、磁盘）
  2. 检查数据库连接池
  3. 分析慢查询日志
  4. 优化工具实现

### 调试工具

#### 1. MCP 调试客户端
```bash
# 安装调试工具
npm install -g @modelcontextprotocol/inspector

# 启动调试器
mcp-inspector --server "java -jar agentx-server.jar --mcp.transport=stdio"
```

#### 2. 日志级别调整
```bash
# 启动时设置日志级别
java -jar agentx-server.jar \
  --logging.level.com.agentx.mcp=DEBUG \
  --logging.level.com.agentx.tools=INFO
```

#### 3. 网络抓包
```bash
# 使用 tcpdump 捕获 MCP 流量
tcpdump -i lo port 8081 -w mcp-traffic.pcap

# 使用 Wireshark 分析
wireshark mcp-traffic.pcap
```

## 最佳实践

### 1. 工具设计
- 保持工具接口简单明确
- 提供完整的参数验证
- 返回结构化的结果
- 实现适当的错误处理

### 2. 资源管理
- 使用合适的 URI 方案
- 实现资源缓存
- 提供资源变更通知
- 控制资源访问权限

### 3. 客户端实现
- 实现连接重试机制
- 处理服务器推送通知
- 优化请求批处理
- 监控连接状态

### 4. 安全考虑
- 始终使用认证
- 实施最小权限原则
- 记录所有工具调用
- 定期审计权限配置

### 5. 性能优化
- 启用连接池
- 配置适当的缓存
- 监控关键指标
- 定期性能测试

## 下一步

根据你的需求选择：
- **工具开发者**：查看工具实现和集成指南
- **客户端开发者**：参考客户端集成示例
- **系统管理员**：查看安全配置和性能优化

如需更多帮助，请查看：
- [MCP 官方文档](https://spec.modelcontextprotocol.io)
- [AgentX MCP 示例](https://github.com/agentx/mcp-examples)
- [社区支持](https://discord.gg/agentx)