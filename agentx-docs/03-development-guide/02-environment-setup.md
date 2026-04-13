# 环境搭建

本章详细说明如何搭建 AgentX 的开发环境，包括系统要求、软件安装、配置步骤和常见问题解决。

## 系统要求

### 最低配置
- **操作系统**：Windows 10/11、macOS 10.15+、Linux（Ubuntu 20.04+/CentOS 8+）
- **CPU**：2 核以上（建议 4 核）
- **内存**：8GB 以上（建议 16GB）
- **磁盘空间**：10GB 可用空间（建议 SSD）
- **网络**：可访问互联网（下载依赖包）

### 推荐配置（最佳开发体验）
- **CPU**：4 核以上
- **内存**：16GB 以上
- **磁盘**：NVMe SSD，50GB 可用空间
- **操作系统**：Linux（开发环境更稳定）
- **网络**：稳定的网络连接

## 开发环境安装

### 1. Java 开发工具包（JDK）

#### Windows
1. 下载 [JDK 21](https://www.oracle.com/java/technologies/downloads/#java21)
2. 运行安装程序，按默认选项安装
3. 配置环境变量：
   ```
   JAVA_HOME = C:\Program Files\Java\jdk-21
   PATH = %JAVA_HOME%\bin;%PATH%
   ```

#### macOS
```bash
# 使用 Homebrew 安装
brew install openjdk@21

# 设置环境变量
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 21)' >> ~/.zshrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.zshrc
source ~/.zshrc
```

#### Linux（Ubuntu/Debian）
```bash
# 添加 OpenJDK 仓库
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:openjdk-r/ppa
sudo apt update

# 安装 OpenJDK 21
sudo apt install -y openjdk-21-jdk

# 验证安装
java -version
```

#### Linux（CentOS/RHEL）
```bash
# 安装 OpenJDK 21
sudo yum install -y java-21-openjdk-devel

# 设置默认 Java 版本
sudo alternatives --config java

# 验证安装
java -version
```

### 2. Apache Maven

#### Windows
1. 下载 [Maven](https://maven.apache.org/download.cgi)
2. 解压到 `C:\Program Files\apache-maven-3.9.6`
3. 配置环境变量：
   ```
   MAVEN_HOME = C:\Program Files\apache-maven-3.9.6
   PATH = %MAVEN_HOME%\bin;%PATH%
   ```

#### macOS
```bash
# 使用 Homebrew 安装
brew install maven

# 验证安装
mvn -version
```

#### Linux
```bash
# Ubuntu/Debian
sudo apt install -y maven

# CentOS/RHEL
sudo yum install -y maven

# 验证安装
mvn -version
```

### 3. Git 版本控制

#### Windows
1. 下载 [Git for Windows](https://git-scm.com/download/win)
2. 运行安装程序，按默认选项安装
3. 验证安装：
   ```bash
   git --version
   ```

#### macOS
```bash
# 使用 Homebrew 安装
brew install git

# 验证安装
git --version
```

#### Linux
```bash
# Ubuntu/Debian
sudo apt install -y git

# CentOS/RHEL
sudo yum install -y git

# 验证安装
git --version
```

### 4. Docker 和 Docker Compose

#### Windows/macOS
1. 下载 [Docker Desktop](https://www.docker.com/products/docker-desktop/)
2. 安装后启动 Docker Desktop
3. 确保 Docker 服务正常运行

#### Linux（Ubuntu/Debian）
```bash
# 卸载旧版本
sudo apt-get remove docker docker-engine docker.io containerd runc

# 安装依赖
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 添加 Docker 官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 添加 Docker 仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker 引擎
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 将当前用户加入 docker 组
sudo usermod -aG docker $USER
newgrp docker

# 验证安装
docker --version
docker compose version
```

#### Linux（CentOS/RHEL）
```bash
# 卸载旧版本
sudo yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine

# 安装依赖
sudo yum install -y yum-utils

# 添加 Docker 仓库
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装 Docker 引擎
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 启动 Docker 服务
sudo systemctl start docker
sudo systemctl enable docker

# 将当前用户加入 docker 组
sudo usermod -aG docker $USER
newgrp docker

# 验证安装
docker --version
docker compose version
```

## IDE 配置

### 1. IntelliJ IDEA（推荐）

#### 安装和配置
1. 下载 [IntelliJ IDEA Community 或 Ultimate](https://www.jetbrains.com/idea/download/)
2. 安装后启动，选择以下配置：
   - **主题**：根据喜好选择
   - **插件**：安装以下插件：
     - Lombok
     - MapStruct Support
     - Spring Boot
     - Maven
     - Git

#### 项目导入
```bash
# 克隆项目
git clone https://github.com/your-org/agentx.git
cd agentx
```

1. 在 IntelliJ IDEA 中：
   - File → New → Project from Existing Sources
   - 选择 `pom.xml` 文件
   - 选择 "Import project from external model" → "Maven"
   - 点击 "Next" 完成导入

2. 配置运行/调试：
   - 编辑配置 → 添加新的 Spring Boot 配置
   - 主类：`com.agentx.AgentXApplication`
   - VM 选项：`-Dspring.profiles.active=dev`
   - 工作目录：项目根目录

### 2. VS Code

#### 安装和配置
1. 下载 [VS Code](https://code.visualstudio.com/)
2. 安装以下扩展：
   - Java Extension Pack
   - Spring Boot Extension Pack
   - Maven for Java
   - Lombok Annotations Support
   - Docker
   - GitLens

#### 项目配置
```json
// .vscode/settings.json
{
  "java.configuration.updateBuildConfiguration": "automatic",
  "java.compile.nullAnalysis.mode": "automatic",
  "maven.executable.path": "mvn",
  "java.home": "/path/to/jdk-21",
  "java.import.gradle.enabled": false,
  "java.import.maven.enabled": true,
  "java.server.launchMode": "Standard",
  "spring-boot.ls.java.home": "/path/to/jdk-21"
}
```

### 3. Eclipse

#### 安装和配置
1. 下载 [Eclipse IDE for Enterprise Java Developers](https://www.eclipse.org/downloads/)
2. 安装后配置：
   - Window → Preferences → Java → Installed JREs：添加 JDK 21
   - Window → Preferences → Maven：启用自动导入

#### 项目导入
1. File → Import → Maven → Existing Maven Projects
2. 选择项目根目录
3. 点击 Finish

## 依赖服务安装

### 1. Redis

#### Docker 方式（推荐）
```bash
# 启动 Redis
docker run -d \
  --name redis-dev \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7.4-alpine \
  redis-server --appendonly yes --maxmemory 512mb

# 验证运行
docker ps | grep redis
```

#### 本地安装（Ubuntu）
```bash
# 安装 Redis
sudo apt update
sudo apt install -y redis-server

# 配置 Redis
sudo nano /etc/redis/redis.conf
# 修改以下配置：
# maxmemory 512mb
# maxmemory-policy allkeys-lru
# appendonly yes

# 重启 Redis
sudo systemctl restart redis
sudo systemctl enable redis

# 验证运行
redis-cli ping
```

### 2. Milvus

#### Docker Compose 方式（推荐）
```bash
# 下载 Milvus Docker Compose 文件
wget https://github.com/milvus-io/milvus/releases/download/v2.3.3/milvus-standalone-docker-compose.yml -O docker-compose-milvus.yml

# 启动 Milvus
docker-compose -f docker-compose-milvus.yml up -d

# 验证运行
docker-compose -f docker-compose-milvus.yml ps

# 健康检查
curl http://localhost:19530/health
```

#### 本地安装（仅开发测试）
```bash
# 使用 Docker 单机模式
docker run -d \
  --name milvus-dev \
  -p 19530:19530 \
  -v milvus-data:/var/lib/milvus \
  milvusdb/milvus:v2.3.3 \
  milvus run standalone

# 验证运行
docker logs milvus-dev
```

### 3. PostgreSQL（可选）

#### Docker 方式
```bash
# 启动 PostgreSQL
docker run -d \
  --name postgres-dev \
  -p 5432:5432 \
  -e POSTGRES_USER=agentx \
  -e POSTGRES_PASSWORD=agentx123 \
  -e POSTGRES_DB=agentx_dev \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15-alpine

# 验证运行
docker exec -it postgres-dev psql -U agentx -d agentx_dev -c "SELECT version();"
```

## 项目初始化

### 1. 克隆项目
```bash
# 克隆项目
git clone https://github.com/your-org/agentx.git
cd agentx

# 查看项目结构
tree -L 2
```

### 2. 配置环境变量
```bash
# 复制环境变量模板
cp .env.example .env

# 编辑 .env 文件
nano .env  # 或使用其他编辑器

# 重要配置项：
# OPENAI_API_KEY=your_api_key_here
# REDIS_HOST=localhost
# MILVUS_HOST=localhost
# DB_HOST=localhost
```

### 3. 获取 API 密钥

#### OpenAI API 密钥
1. 访问 [OpenAI Platform](https://platform.openai.com/api-keys)
2. 登录或注册账号
3. 点击 "Create new secret key"
4. 复制生成的密钥到 `.env` 文件

#### 替代方案（如无 OpenAI 密钥）
```bash
# 使用模拟模式（功能受限）
OPENAI_API_KEY=sk-mock-key-for-testing

# 或使用本地模型
AI_PROVIDER=local
LOCAL_MODEL_PATH=/path/to/model.gguf
```

### 4. 构建项目
```bash
# 首次构建（下载依赖）
mvn clean compile

# 如果下载依赖缓慢，可以配置 Maven 镜像
# 编辑 ~/.m2/settings.xml
```

### 5. 启动依赖服务
```bash
# 使用 Docker Compose 启动所有依赖
docker-compose up -d redis milvus postgres

# 或使用项目提供的脚本
./scripts/start-dependencies.sh
```

### 6. 运行应用
```bash
# 方式1：使用 Maven Spring Boot 插件
mvn spring-boot:run

# 方式2：直接运行 Jar 包
mvn clean package -DskipTests
java -jar target/agentx-*.jar

# 方式3：使用 Docker
docker build -t agentx:dev .
docker run -p 8080:8080 --env-file .env agentx:dev
```

### 7. 验证安装
```bash
# 检查应用健康
curl http://localhost:8080/actuator/health

# 检查 API 文档
curl http://localhost:8080/v3/api-docs

# 测试简单请求
curl -X POST http://localhost:8080/api/v1/agents/process \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello", "agentType": "general"}'
```

## 开发工具配置

### 1. 代码格式化

#### Spotless 配置
```xml
<!-- pom.xml 中添加 -->
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>2.40.0</version>
    <configuration>
        <java>
            <googleJavaFormat>
                <version>1.17.0</version>
                <style>GOOGLE</style>
            </googleJavaFormat>
            <removeUnusedImports/>
            <trimTrailingWhitespace/>
            <endWithNewline/>
        </java>
    </configuration>
</plugin>
```

使用方式：
```bash
# 格式化代码
mvn spotless:apply

# 检查代码格式
mvn spotless:check
```

### 2. 代码质量检查

#### Checkstyle 配置
```xml
<!-- pom.xml 中添加 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.2.2</version>
    <configuration>
        <configLocation>checkstyle.xml</configLocation>
        <encoding>UTF-8</encoding>
        <consoleOutput>true</consoleOutput>
        <failsOnError>true</failsOnError>
    </configuration>
</plugin>
```

使用方式：
```bash
# 检查代码规范
mvn checkstyle:check
```

### 3. 测试配置

#### 单元测试
```bash
# 运行所有测试
mvn test

# 运行特定测试类
mvn test -Dtest=AgentServiceTest

# 运行特定测试方法
mvn test -Dtest=AgentServiceTest#testProcessRequest
```

#### 集成测试
```bash
# 使用 Testcontainers 运行集成测试
mvn verify -P integration-test

# 生成测试报告
mvn surefire-report:report
```

## 调试配置

### 1. 远程调试

#### 应用启动参数
```bash
# 添加 JVM 调试参数
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 \
  -jar target/agentx-*.jar
```

#### IDE 配置
1. **IntelliJ IDEA**：
   - Run → Edit Configurations → Add New → Remote JVM Debug
   - Host: localhost, Port: 5005

2. **VS Code**：
   ```json
   // .vscode/launch.json
   {
     "version": "0.2.0",
     "configurations": [
       {
         "type": "java",
         "name": "Debug (Attach)",
         "request": "attach",
         "hostName": "localhost",
         "port": 5005
       }
     ]
   }
   ```

### 2. 日志调试

#### 日志级别配置
```yaml
# application-dev.yml
logging:
  level:
    com.agentx: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

#### 日志文件配置
```yaml
logging:
  file:
    name: logs/agentx-dev.log
  logback:
    rollingpolicy:
      max-file-size: 10MB
      max-history: 30
  pattern:
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

### 3. 性能分析

#### JProfiler 配置
```bash
# 启动应用时添加代理
java -agentpath:/path/to/jprofiler/bin/linux-x64/libjprofilerti.so=port=8849 \
  -jar target/agentx-*.jar
```

#### VisualVM 监控
```bash
# 添加 JMX 参数
java -Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9010 \
  -Dcom.sun.management.jmxremote.ssl=false \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -jar target/agentx-*.jar
```

## 常见问题解决

### 1. 端口冲突

**问题**：端口 8080、6379、19530 被占用

**解决**：
```bash
# 查看端口占用
netstat -tulpn | grep :8080  # Linux
lsof -i :8080               # macOS
netstat -ano | findstr :8080 # Windows

# 停止占用进程或修改配置
# 修改 application.yml 中的 server.port
```

### 2. 内存不足

**问题**：Java 应用内存不足或 Docker 容器内存不足

**解决**：
```bash
# 调整 JVM 内存
export JAVA_OPTS="-Xmx2g -Xms1g"

# 调整 Docker 内存限制
docker run --memory=4g --memory-swap=4g ...

# 查看内存使用
docker stats
```

### 3. 依赖下载失败

**问题**：Maven 依赖下载缓慢或失败

**解决**：
```xml
<!-- 配置 Maven 镜像 -->
<mirror>
  <id>aliyunmaven</id>
  <mirrorOf>*</mirrorOf>
  <name>阿里云公共仓库</name>
  <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

### 4. Docker 连接问题

**问题**：Docker 容器间无法通信

**解决**：
```bash
# 使用 Docker Compose 网络
docker-compose up -d

# 或创建自定义网络
docker network create agentx-network
docker run --network agentx-network ...
```

### 5. 数据库连接失败

**问题**：应用无法连接 Redis、Milvus 或 PostgreSQL

**解决**：
```bash
# 检查服务状态
docker ps

# 检查服务日志
docker logs redis-dev

# 测试连接
redis-cli -h localhost -p 6379 ping
```

## 环境验证清单

### 基本环境
- [ ] Java 21 安装正确：`java -version`
- [ ] Maven 安装正确：`mvn -version`
- [ ] Git 安装正确：`git --version`
- [ ] Docker 安装正确：`docker --version`

### 依赖服务
- [ ] Redis 运行正常：`redis-cli ping` 返回 PONG
- [ ] Milvus 运行正常：`curl http://localhost:19530/health` 返回正常
- [ ] PostgreSQL 运行正常（可选）

### 项目配置
- [ ] 项目克隆成功
- [ ] 环境变量配置正确
- [ ] API 密钥配置正确
- [ ] 依赖下载完成：`mvn dependency:resolve`

### 应用运行
- [ ] 应用编译成功：`mvn clean compile`
- [ ] 应用启动成功：无错误日志
- [ ] 健康检查通过：`curl http://localhost:8080/actuator/health`
- [ ] API 接口可用：简单请求测试通过

## 下一步

环境搭建完成后，建议：

1. **运行示例**：运行项目中的示例代码
2. **查看文档**：阅读 API 文档和代码注释
3. **修改测试**：尝试修改代码并运行测试
4. **开发工具**：按照下一章指导开发新工具

如果遇到问题，请查看 [常见问题](02-environment-setup.md#常见问题解决) 或提交 Issue。