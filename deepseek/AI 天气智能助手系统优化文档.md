这是一份关于 **AI 天气智能助手系统优化** 的完整技术文档，整理了从基础修复到高级流式打字机效果实现的全部过程。

---

# AI 天气智能助手系统优化文档

## 1. 项目背景与目标
本项目基于 **Spring AI (后端)** 与 **Vue 3 (前端)** 开发，旨在构建一个能够通过 AI 调用天气工具（Function Calling）并实时反馈给用户的天气查询助手。
**核心目标**：
*   实现低延迟的 **流式（Streaming）响应** 与 **打字机效果**。
*   优化 UI 体验，增加 **思考计时器**、**Markdown 渲染** 与 **现代化交互界面**。
*   解决 `SelfRefineEvaluationAdvisor` 与流式输出的逻辑冲突问题。

---

## 2. 后端架构优化 (Spring AI)

### 2.1 解决 Advisor 冲突与流式配置
**核心问题**：`SelfRefineEvaluationAdvisor`（自我修正顾问）需要获取完整文本进行评估，不支持流式输出，会导致 500 错误。
**优化方案**：配置两个 `ChatClient` 实例，分别处理高质量生成与实时流式输出。

#### `AiConfig.java` 配置
```java
@Configuration
public class AiConfig {
    // 1. 标准客户端：包含自我修正，用于非流式 call()
    @Bean
    public ChatClient weatherChatClient(ChatClient.Builder builder, ChatModel chatModel) {
        return builder
            .defaultSystem("你是一个专业的气象助手。")
            .defaultTools(new WeatherTool()) 
            .defaultAdvisors(
                SelfRefineEvaluationAdvisor.builder()
                    .chatClientBuilder(ChatClient.builder(chatModel))
                    .maxRepeatAttempts(5)
                    .build(),
                new MyLoggingAdvisor(2)
            ).build();
    }

    // 2. 流式客户端：去掉自我修正，用于追求速度的 .stream()
    @Bean
    public ChatClient weatherStreamingChatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("你是一个专业的气象助手。")
            .defaultTools(new WeatherTool()) 
            .defaultAdvisors(new MyLoggingAdvisor(2))
            .build();
    }
}
```

### 2.2 逻辑层与接口实现
在 Service 层根据业务需求分发请求，Controller 开启 SSE 流支持。

#### `WeatherService.java`
```java
public Flux<String> doWorkStream(String city) {
    return this.streamingClient.prompt()
            .user(u -> u.text("你好！请查询 {city} 的天气，并友好回复。").param("city", city))
            .stream().content();
}
```

#### `Controller.java`
```java
@GetMapping(value = "/weather/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> weatherStream(@RequestParam String city) {
    // 校验逻辑：仅允许中英文及空格，防止 Prompt 注入
    if (!city.matches("^[a-zA-Z\\u4e00-\\u9fa5\\s·]+$")) return Flux.just("无效的城市名");
    return weatherService.doWorkStream(city);
}
```

---

## 3. 前端界面优化 (Vue 3)

### 3.1 核心逻辑提升
*   **流式接收**：摒弃 Axios，改用原生 `fetch` 处理 `ReadableStream`。
*   **Markdown 渲染**：引入 `markdown-it` 解析 AI 返回的加粗、列表等语法。
*   **思考计时**：通过 `setInterval` 实现实时秒数统计。

### 3.2 打字机效果实现逻辑
```typescript
const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  const chunk = decoder.decode(value, { stream: true });
  
  // 处理 SSE 格式数据 (data: ...)
  const lines = chunk.split('\n');
  lines.forEach(line => {
    if (line.startsWith('data:')) {
      answer.value += line.replace('data:', '').trim();
    } else if (line.trim() !== '' && !line.startsWith('event:')) {
      answer.value += line;
    }
  });
}
```

---

## 4. UI/UX 设计方案 (简约现代风)

### 4.1 视觉改进
1.  **卡片化布局**：使用 `box-shadow` 和 `border-radius: 20px` 营造浮动感。
2.  **拟物化交互**：搜索框增加 `focus` 状态下的蓝色呼吸灯效果。
3.  **状态动效**：
    *   **思考中**：增加 `pulse` 动画小圆点。
    *   **传输中**：文字末尾增加 `blink` 闪烁光标。
    *   **内容载入**：使用 `<transition>` 实现内容向上滑入动画。

### 4.2 关键 CSS 样式
```css
/* 现代渐变背景 */
.page-container {
  background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
}

/* 模拟 AI 打字光标 */
.cursor {
  display: inline-block;
  width: 2px;
  height: 1.2em;
  background-color: #3498db;
  animation: blink 1s infinite;
}

@keyframes blink { 50% { opacity: 0; } }
```

---

## 5. 关键优化总结 (QA)

| 问题现象 | 根本原因 | 最终解决方案 |
| :--- | :--- | :--- |
| **文字垂直排列（一字一行）** | 前端 `v-for` 错误遍历了字符串。 | 删除循环，使用 `v-html` 渲染完整 Markdown。 |
| **500 内部服务器错误** | `SelfRefineAdvisor` 与流模式冲突。 | **双 Client 架构**：流式请求跳过该 Advisor。 |
| **响应速度慢** | 重复请求 AI（先问好再查天气）。 | **Prompt 合并**：一次调用完成人格设定与查询。 |
| **界面死板** | 缺乏状态反馈。 | 增加 **思考计时器** 和 **呼吸灯加载动画**。 |
| **安全性风险** | 潜在的 Prompt 注入攻击。 | 前后端增加 **正则校验**，拦截非法特殊字符。 |

---

## 6. 后续扩展建议
1.  **历史记录保存**：引入 Pinia 或 LocalStorage 保存用户对话记录。
2.  **语音播报**：集成浏览器自带的 `SpeechSynthesis` API 实现天气语音播报。
3.  **自动补全**：前端对接地图 API（如高德/百度），提供城市名自动补全功能。

---
**整理人**：AI 助手
**日期**：2026-03-09