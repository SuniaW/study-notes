
# Spring AI 函数调用（Function Calling）配置与实现指南

## 1. 问题诊断：为什么你的 Function 无法使用？

通过分析你提供的 `pom.xml`，存在以下三个核心问题：

### 1.1 非法的 Spring Boot 版本 (4.0.0)
*   **现象**：你设置了 `<version>4.0.0</version>`。
*   **原因**：目前 Spring Boot 官方尚未发布 4.0.0 版本（当前稳定版为 3.4.x）。
*   **后果**：Maven 无法找到父工程，导致所有依赖解析失败，Spring 容器无法正常启动或无法加载 AI 相关组件。

### 1.2 缺少具体的模型 Starter
*   **现象**：你只配置了 `spring-ai-bom`，且注释掉了 `spring-ai-starter`。
*   **原因**：Spring AI 的函数调用是**依赖具体模型实现**的。
*   **后果**：如果你不引入 `spring-ai-openai-starter` 或 `spring-ai-ollama-starter` 等具体依赖，系统中就没有 `ChatModel` 的实现类，Function 也就无处挂载。

### 1.3 版本不匹配 (Spring AI 2.0.0-M2)
*   **原因**：`2.0.0-M2` 是针对未来更高版本 Spring Boot 的预览版。对于目前的开发，建议使用与 Spring Boot 3.x 兼容性最好的 `1.0.0-M5` 版本。

---

## 2. 修正后的 Maven 配置 (pom.xml)

请将你的 `pom.xml` 修改为以下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.4.2</version> <!-- 修正：使用稳定的 3.x 版本 -->
		<relativePath/>
	</parent>

	<groupId>com.wx.ai.learn</groupId>
	<artifactId>ai-learn</artifactId>
	<version>0.0.1-SNAPSHOT</version>

	<properties>
		<java.version>21</java.version>
		<spring-ai.version>1.0.0-M5</spring-ai.version> <!-- 修正：匹配当前主流版本 -->
	</properties>

	<dependencies>
		<!-- 基础 Web 支持 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<!-- 必须：引入具体模型的 Starter（以 OpenAI 为例） -->
		<dependency>
			<groupId>org.springframework.ai</groupId>
			<artifactId>spring-ai-openai-starter</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.ai</groupId>
				<artifactId>spring-ai-bom</artifactId>
				<version>${spring-ai.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<repositories>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>
</project>
```

---

## 3. 实现 Function Calling 的关键步骤

配置好环境后，要让 Function 真正跑起来，需要遵循以下三个步骤：

### 第一步：定义函数 Bean
AI 并不直接调用你的代码，它通过 Spring 容器找到对应的 `Function`。必须使用 `@Description` 告诉 AI 这个函数是干什么的。

```java
@Configuration
public class AiConfig {

    @Bean
    @Description("获取指定城市的当前天气情况") // 必须：AI 根据此描述决定是否调用
    public Function<WeatherRequest, WeatherResponse> weatherFunction() {
        return request -> {
            // 这里写你的业务逻辑（调天气 API）
            return new WeatherResponse(request.city() + "，25度，晴");
        };
    }
}

// 定义入参和出参记录
public record WeatherRequest(String city) {}
public record WeatherResponse(String status) {}
```

### 第二步：在 Prompt 中开启函数调用
在调用模型时，你需要显式地指定要使用哪个函数。

```java
@RestController
public class ChatController {

    private final ChatModel chatModel;

    public ChatController(ChatModel chatModel) {
        this.chatModel = chatModel;
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        var options = OpenAiChatOptions.builder()
            .withFunction("weatherFunction") // 对应 Bean 的名称
            .build();

        ChatResponse response = chatModel.call(new Prompt(message, options));
        return response.getResult().getOutput().getContent();
    }
}
```

---

## 4. 常见排查点 (Checklist)

1.  **Bean 名称匹配**：`withFunction("xxx")` 里的名字必须和 `@Bean` 的方法名完全一致。
2.  **`@Description` 缺失**：如果没有这个注解，AI 无法理解函数的作用，永远不会触发调用。
3.  **API Key 权限**：确保你的 OpenAI 或其他模型 Key 具有调用相关功能的权限（如 GPT-4o, GPT-3.5-turbo）。
4.  **网络代理**：如果你在国内环境，确保配置了 `spring.ai.openai.base-url` 或系统代理。

---

**总结**：先修正 `pom.xml` 中的 Spring Boot 版本并引入具体的 AI 实现库，然后确保函数以 `@Bean` 形式存在并带有详细的 `@Description`。