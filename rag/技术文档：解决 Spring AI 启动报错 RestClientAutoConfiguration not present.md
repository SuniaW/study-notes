# 技术文档：解决 Spring AI 启动报错 "RestClientAutoConfiguration not present"

## 1. 问题描述
在集成 Spring AI（特别是 OpenAI 模块）时，项目启动失败并抛出以下异常：

```text
org.springframework.beans.factory.BeanDefinitionStoreException: 
Failed to process import candidates for configuration class [org.springframework.ai.autoconfigure.openai.OpenAiAutoConfiguration]: 
Type org.springframework.boot.autoconfigure.web.client.RestClientAutoConfiguration not present
```

## 2. 根本原因分析
该问题是由 **版本不兼容** 导致的：

*   **核心依赖缺失**：`OpenAiAutoConfiguration` 类在较新版本的 Spring AI 中依赖于 `RestClient`。
*   **版本门槛**：`RestClient` 是在 Spring Framework 6.1 中引入的，而 Spring Boot 对其的自动配置类 `RestClientAutoConfiguration` 是从 **Spring Boot 3.2.0** 版本才开始提供的。
*   **结论**：如果你在 **Spring Boot 3.0 或 3.1** 的项目中使用 **Spring AI 1.0.0-M1** 及以上版本，由于 classpath 中缺少该自动配置类，会导致初始化失败。

## 3. 解决方案

### 方案 A：升级 Spring Boot 版本（推荐）
这是最标准且长期的解决办法。将 Spring Boot 升级至 3.2.x 或更高版本。

#### Maven 配置 (`pom.xml`)
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version> <!-- 确保版本号 >= 3.2.0 -->
    <relativePath/>
</parent>
```

#### Gradle 配置 (`build.gradle`)
```gradle
plugins {
    id 'org.springframework.boot' version '3.2.5'
    id 'io.spring.dependency-management' version '1.1.4'
}
```

---

### 方案 B：规范 Spring AI 依赖管理
确保使用了 Spring AI 的 BOM（物料清单）来管理版本，以防止子模块版本冲突。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0-M1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

### 方案 C：降级 Spring AI 版本（不推荐）
如果你由于环境限制无法升级 Spring Boot 3.2，你必须将 Spring AI 降级到与 Spring Boot 3.1 兼容的旧版本（例如 `0.8.1`）。

> **注意**：旧版本可能不支持最新的 OpenAI 模型特性（如 GPT-4o 等）。

---

## 4. 版本兼容性参照表表

| Spring AI 版本 | 推荐 Spring Boot 版本 | 核心底层客户端 |
| :--- | :--- | :--- |
| **1.0.0-M1 及以上** | **3.2.x, 3.3.x+** | **RestClient** (新特性) |
| **0.8.x 及以下** | 3.0.x, 3.1.x | WebClient / RestTemplate |

## 5. 补充验证
升级后，请确保项目中包含 `spring-boot-starter-web` 依赖，因为它包含了 `RestClient` 运行所需的 HTTP 基础库：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

---
**文档状态**：已解决  
**关键词**：Spring AI, OpenAI, RestClient, Spring Boot 3.2, 自动配置失败