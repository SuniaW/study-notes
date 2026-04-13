# 快速体验

本章提供 AgentX 的快速上手指南，帮助你在 **5 分钟内** 完成环境搭建和基础功能体验。

## 环境要求

### 最低配置（体验环境）
- **操作系统**：Windows 10/11、macOS 10.15+、Linux（Ubuntu 20.04+）
- **内存**：4GB 以上（2GB 可用内存）
- **磁盘空间**：2GB 可用空间
- **网络**：可访问互联网（下载 Docker 镜像）

### 推荐配置（开发环境）
- **CPU**：2 核以上
- **内存**：8GB 以上
- **磁盘**：SSD，10GB 可用空间
- **Docker**：20.10+ 版本
- **Docker Compose**：2.0+ 版本

## 方法一：Docker Compose 一键启动（推荐）

### 步骤 1：安装 Docker 和 Docker Compose

#### Windows/macOS
1. 下载 [Docker Desktop](https://www.docker.com/products/docker-desktop/)
2. 安装后启动 Docker Desktop
3. 确保 Docker 服务正常运行

#### Linux（Ubuntu/Debian）
```bash
# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 安装 Docker Compose 插件
sudo apt-get update
sudo apt-get install docker-compose-plugin

# 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER
newgrp docker
```

### 步骤 2：下载 AgentX 项目
```bash
# 克隆项目（如果已有项目可跳过）
git clone https://github.com/your-org/agentx.git
cd agentx

# 或直接下载 docker-compose.yml
curl -o docker-compose.yml https://raw.githubusercontent.com/your-org/agentx/main/docker-compose.yml
```

### 步骤 3：配置环境变量
```bash
# 复制环境变量模板
cp .env.example .env

# 编辑 .env 文件，设置 OpenAI API 密钥
# 使用文本编辑器打开 .env，修改以下内容：
# OPENAI_API_KEY=your_actual_api_key_here
# 如果暂时没有 API 密钥，可使用模拟模式（功能受限）
# OPENAI_API_KEY=sk-mock-key-for-testing
```

> 💡 **获取 API 密钥**：
> - 访问 [OpenAI Platform](https://platform.openai.com/api-keys) 创建 API 密钥
> - 或使用 [DeepSeek](https://platform.deepseek.com/api_keys) 等替代服务
> - 测试阶段可使用模拟模式，但部分 AI 功能不可用

### 步骤 4：启动服务
```bash
# 启动所有服务（前台运行，查看日志）
docker-compose up

# 或后台运行
docker-compose up -d

# 查看服务状态
docker-compose ps
```

正常启动后，你应该看到类似以下输出：
```
Creating network "agentx_default" with the default driver
Creating agentx-milvus ... done
Creating agentx-redis  ... done
Creating agentx-app    ... done
```

### 步骤 5：验证服务
```bash
# 检查应用健康状态
curl http://localhost:8080/actuator/health

# 预期输出：
# {"status":"UP","components":{...}}

# 检查各组件健康状态
curl http://localhost:8080/actuator/health/redis
curl http://localhost:8080/actuator/health/milvus
```

## 方法二：从源码运行（开发者模式）

### 步骤 1：安装 Java 和 Maven

#### Windows
1. 下载 [Java 21](https://www.oracle.com/java/technologies/downloads/#java21)
2. 下载 [Maven](https://maven.apache.org/download.cgi)
3. 配置环境变量 `JAVA_HOME` 和 `MAVEN_HOME`

#### macOS
```bash
# 使用 Homebrew 安装
brew install openjdk@21
brew install maven

# 设置 Java 环境变量
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
```

#### Linux（Ubuntu）
```bash
# 安装 Java 21
sudo apt update
sudo apt install openjdk-21-jdk

# 安装 Maven
sudo apt install maven

# 验证安装
java -version
mvn -version
```

### 步骤 2：下载并编译项目
```bash
# 克隆项目
git clone https://github.com/your-org/agentx.git
cd agentx

# 编译项目
mvn clean package -DskipTests

# 编译成功后，在 target/ 目录生成 agentx-*.jar
```

### 步骤 3：启动依赖服务
```bash
# 使用 Docker Compose 启动 Redis 和 Milvus
docker-compose up -d redis milvus

# 或手动启动（如果已有 Redis/Milvus 实例）
# 确保 Redis 运行在 localhost:6379
# 确保 Milvus 运行在 localhost:19530
```

### 步骤 4：运行应用
```bash
# 方式1：直接运行 Jar 包
java -jar target/agentx-*.jar

# 方式2：使用 Maven Spring Boot 插件
mvn spring-boot:run

# 方式3：使用启动脚本（Linux/macOS）
chmod +x start.sh
./start.sh start
```

应用启动后，控制台会显示类似信息：
```
Started AgentXApplication in 15.234 seconds (process running for 16.456)
Tomcat started on port(s): 8080 (http)
```

## 基础功能体验

### 1. 测试 AI 对话
```bash
# 使用 curl 测试
curl -X POST http://localhost:8080/api/v1/agents/process \
  -H "Content-Type: application/json" \
  -d '{
    "message": "你好，介绍一下你自己",
    "agentType": "general",
    "sessionId": "test-session-001"
  }'

# 或使用浏览器访问 Swagger UI
# 打开 http://localhost:8080/swagger-ui.html
# 找到 AgentController -> processAgentRequest 接口
# 点击 "Try it out"，输入参数并执行
```

### 2. 测试工具调用
```bash
# 查询天气（使用内置 WeatherTool）
curl -X POST http://localhost:8080/api/v1/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "toolName": "get-weather",
    "parameters": {
      "city": "北京"
    }
  }'

# 查询风控规则（金融场景）
curl -X POST http://localhost:8080/api/v1/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "toolName": "risk-rule-query",
    "parameters": {
      "ruleId": "RULE_001"
    }
  }'
```

### 3. 测试工作流
```bash
# 执行简单工作流
curl -X POST http://localhost:8080/api/v1/workflows/execute \
  -H "Content-Type: application/json" \
  -d '{
    "workflowId": "simple-qa",
    "parameters": {
      "question": "北京今天天气如何？"
    }
  }'
```

### 4. 查看监控指标
```bash
# 查看应用指标
curl http://localhost:8080/actuator/metrics

# 查看特定指标（如 JVM 内存）
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Prometheus 格式指标
curl http://localhost:8080/actuator/prometheus
```

## Web 界面访问

### Swagger API 文档
- 地址：http://localhost:8080/swagger-ui.html
- 功能：所有 API 的交互式文档，可直接测试接口

### Spring Boot Actuator
- 健康检查：http://localhost:8080/actuator/health
- 应用信息：http://localhost:8080/actuator/info
- 环境配置：http://localhost:8080/actuator/env
- 日志级别：http://localhost:8080/actuator/loggers

### （可选）前端管理界面
> 注：部分版本提供独立的前端管理界面，需额外启动

```bash
# 启动前端服务
cd agentx-frontend
npm install
npm run dev

# 访问地址：http://localhost:3000
```

## 常见问题排查

### 1. Docker 启动失败
**问题**：`docker-compose up` 报错或容器异常退出

**解决步骤**：
```bash
# 查看详细日志
docker-compose logs

# 检查端口冲突（8080、6379、19530 是否被占用）
netstat -an | grep 8080

# 清理旧容器和镜像
docker-compose down -v
docker system prune -a

# 重新启动
docker-compose up
```

### 2. 应用启动失败
**问题**：Java 应用启动报错，如连接不上 Redis/Milvus

**解决步骤**：
```bash
# 检查依赖服务是否正常运行
docker-compose ps

# 检查服务连通性
curl http://localhost:6379  # Redis 应该返回错误（正常）
telnet localhost 19530     # Milvus 应该能连接

# 查看应用日志
docker-compose logs agentx-app
```

### 3. API 调用返回错误
**问题**：curl 测试返回 4xx/5xx 错误

**解决步骤**：
```bash
# 查看详细的错误信息（增加 -v 参数）
curl -v -X POST http://localhost:8080/api/v1/agents/process ...

# 检查请求格式是否正确
# 确保 Content-Type: application/json
# 确保 JSON 格式正确

# 查看应用日志中的错误堆栈
docker-compose logs agentx-app | grep -A 10 -B 5 "ERROR"
```

### 4. 内存不足
**问题**：容器因 OOM（内存溢出）被杀死

**解决步骤**：
```bash
# 调整 Docker 内存限制（在 docker-compose.yml 中）
services:
  agentx-app:
    deploy:
      resources:
        limits:
          memory: 2G  # 增加到 2GB

# 或调整 JVM 参数（在 application.yml 中）
spring:
  jvm:
    args: "-Xmx1g -Xms512m"
```

## 下一步学习建议

### 1. 业务决策者
- 查看 [适用场景](03-use-cases.md) 了解更多行业案例
- 联系我们的 [销售团队](mailto:sales@agentx.example.com) 获取商业评估
- 阅读 [客户案例](../appendix/04-resource-links.md#客户案例) 了解实施效果

### 2. 技术架构师
- 深入学习 [系统架构](../02-architecture/README.md)
- 了解 [部署方案](../05-config-deployment/README.md)
- 评估 [扩展性设计](../08-advanced-topics/03-large-scale-deployment.md)

### 3. 开发人员
- 学习 [开发指南](../03-development-guide/README.md)
- 实践 [添加新工具](../03-development-guide/03-adding-tools.md)
- 探索 [API 文档](../04-api-reference/README.md)

### 4. 运维人员
- 学习 [健康检查](../07-operations-monitoring/01-health-checks.md)
- 掌握 [性能监控](../07-operations-monitoring/02-performance-monitoring.md)
- 了解 [故障排查](../07-operations-monitoring/04-troubleshooting.md)

## 获取帮助

- **文档问题**：查看 [常见问题](../appendix/01-faq.md)
- **技术问题**：
  - GitHub Issues：https://github.com/your-org/agentx/issues
  - Discord 社区：https://discord.gg/agentx
- **商业咨询**：sales@agentx.example.com
- **紧急支持**：support@agentx.example.com（仅限企业客户）

> 💡 **提示**：快速体验环境仅用于功能验证，生产环境需要更多配置和优化。建议在测试环境充分验证后再部署到生产环境。