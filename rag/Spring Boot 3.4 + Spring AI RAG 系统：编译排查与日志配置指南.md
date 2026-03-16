

# Spring Boot 3.4 + Spring AI RAG 系统：编译排查与日志配置指南

**项目背景：** 基于 Spring Boot 3.4 + Spring AI + Milvus 构建的政务大模型 RAG（检索增强生成）系统。
**文档目的：** 记录 JDK/Lombok 版本不兼容导致的编译错误解决方案，并提供业务所需的 Debug 日志及按天滚动日志的最佳实践配置。

---

## 🛠️ 第一部分：编译异常排查与解决

### 1. 错误现象
在 IntelliJ IDEA 中编译或启动项目时，出现以下致命错误：
```text
java: java.lang.ExceptionInInitializerError
com.sun.tools.javac.code.TypeTag :: UNKNOWN
```

### 2. 问题根源
这是 **Lombok 与 Java 25 早期访问版（EA）不兼容** 导致的。
Lombok 的底层原理是利用 Java 编译器的内部私有 API（`com.sun.tools.javac.*`）在编译期动态生成 `getter/setter` 等方法。由于 Java 25 的编译器内部 API 发生了变化，而当前的 Lombok 版本尚未适配 Java 25，导致 Lombok 在解析抽象语法树（AST）时崩溃，抛出 `TypeTag :: UNKNOWN` 异常。

### 3. 解决方案：降级至 Java 21 LTS（推荐）
对于企业级政务系统，稳定性是首要考虑因素。建议将项目和开发环境统一降级至当前的长期支持版本 **Java 21**。

#### 步骤一：修改 `pom.xml`
1. 将 `<java.version>` 修改为 21。
2. 移除手动指定的 Lombok 版本号（让 Spring Boot 3.4 自动管理最兼容的版本）。
3. 为 `maven-compiler-plugin` 显式添加 Lombok 的注解处理器路径。

修改后的核心 `pom.xml` 如下：
```xml
<properties>
    <java.version>21</java.version>
</properties>

<!-- 依赖部分只保留 lombok，去掉版本号 -->
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>${java.version}</source>
                <target>${java.version}</target>
                <!-- 必须配置注解处理器路径，否则 Java 21 无法正常触发 Lombok -->
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### 步骤二：修改 IDEA 配置
确保 IDEA 的所有环境都指向 Java 21：
1. **项目结构：** `File` -> `Project Structure` -> `Project`，将 SDK 和 Language level 设为 21。
2. **编译器：** `Settings` -> `Build, Execution, Deployment` -> `Compiler` -> `Java Compiler`，将 Bytecode version 设为 21。
3. **Maven 运行环境：** `Settings` -> `Build, Execution, Deployment` -> `Build Tools` -> `Maven` -> `Runner`，将 JRE 设为 21。
4. 刷新 Maven 并重新编译。

---

## 📝 第二部分：日志系统配置（Debug 与 按天归档）

在 RAG 系统开发中，开启 Debug 日志至关重要，它可以帮助我们查看**大模型的真实 Prompt 组装过程**以及 **Milvus 向量检索的具体参数**。同时，业务要求日志必须按天生成，避免单文件过大。

以下提供两种配置方案，请根据实际场景选择。

### 方案一：通过 `application.yml` 快速配置（适合开发/测试环境）
这是最简单的方式，直接利用 Spring Boot 对 Logback 的内置支持。只需在 `application.yml` 中添加以下配置即可实现 Debug 日志输出和按天滚动：

```yaml
logging:
  level:
    # 1. 开启你自己项目包的 Debug 日志
    com.ailearn.governmentaffairsrag: DEBUG
    # 2. 开启 Spring AI 日志 (非常重要！可查看真实 Prompt 和模型请求)
    org.springframework.ai: DEBUG
    # 3. 开启 Milvus 向量数据库交互日志
    io.milvus: DEBUG
  
  file:
    # 当前正在写入的活动日志文件名
    name: logs/rag-system.log
  
  logback:
    rollingpolicy:
      # 【核心】历史日志归档的文件名模式，%d{yyyy-MM-dd} 表示按天，%i 表示当天的第几个切分文件
      file-name-pattern: logs/rag-system-%d{yyyy-MM-dd}.%i.log
      # 单个日志文件的最大大小，超过则生成 .1.log, .2.log
      max-file-size: 100MB
      # 历史日志保留的天数
      max-history: 30
      # 所有日志文件占用的总最大空间，超过会自动清理旧日志
      total-size-cap: 3GB
```

---

### 方案二：通过 `logback-spring.xml` 高级配置（适合生产环境）
生产环境下，运维通常要求**将常规日志和 ERROR 报错日志分开存放**，以便快速定位系统故障。
请在项目的 `src/main/resources/` 目录下新建 `logback-spring.xml` 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 引入 Spring Boot 默认的控制台颜色和格式设置 -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <!-- 定义日志存储路径和文件前缀 -->
    <property name="LOG_PATH" value="logs" />
    <property name="APP_NAME" value="rag-system" />

    <!-- 1. 控制台输出配置 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 2. 全量日志（按天滚动输出） -->
    <appender name="FILE_DAILY" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.log</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!-- 3. ERROR 级别错误日志（按天独立存放） -->
    <appender name="FILE_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-error.log</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 拦截器：只接受 ERROR 级别的日志 -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <!-- 错误日志建议保留时间更长 -->
            <maxHistory>60</maxHistory> 
        </rollingPolicy>
    </appender>

    <!-- 4. 动态调整指定包的日志级别 (开启 Debug) -->
    <logger name="com.ailearn.governmentaffairsrag" level="DEBUG" />
    <logger name="org.springframework.ai" level="DEBUG" />
    <logger name="io.milvus" level="DEBUG" />

    <!-- 5. 根节点配置 -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE_DAILY" />
        <appender-ref ref="FILE_ERROR" />
    </root>
</configuration>
```
*(注：如果使用了 `logback-spring.xml`，请将 `application.yml` 中关于 `logging.file` 和 `logging.logback` 的配置删除，以避免冲突。)*