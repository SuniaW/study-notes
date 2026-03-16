
---
组件文档：SelfRefineEvaluationAdvisor
#  (自我改进评估顾问)

## 1. 概述
`SelfRefineEvaluationAdvisor` 是一个基于 Spring AI 框架定制的顾问组件（Advisor）。它实现了 **"LLM-as-a-Judge"（大模型即裁判）** 模式，通过引入一个自动化的“评价-反馈-改进”循环，提升大模型输出内容的质量和准确性。

## 2. 核心设计模式
*   **自我修正 (Self-Refine)**：模型生成初步回答后，由另一个（或同一个）模型实例进行评估。
*   **递归重试 (Recursive Retry)**：如果评估分数未达标，系统会自动将评估建议反馈给主模型，要求其重新生成，直到满足预设标准或达到尝试上限。
*   **结构化评估 (Structured Evaluation)**：评估结果以结构化 JSON 格式返回，包含评分、评估理由和具体的改进意见。

## 3. 工作流程
该顾问通过拦截 `ChatClient` 的请求执行以下逻辑：

1.  **初始生成**：调用原始模型生成初步回答。
2.  **跳过判定**：检查响应是否包含工具调用（Tool Calls）。如果包含，通常视为中间步骤，直接跳过评估。
3.  **模型评估**：
    *   将原始问题与生成的回答发送给“裁判模型”。
    *   裁判根据 1-4 分的量表进行评分。
4.  **阈值检查**：
    *   若 **评分 ≥ 成功阈值**：直接将结果返回给最终用户。
    *   若 **评分 < 成功阈值**：进入重试流程。
5.  **提示词增强 (Augmentation)**：
    *   将裁判提供的 `feedback`（改进建议）追加到原始用户问题中。
    *   重新发起模型调用。
6.  **终止条件**：达到成功评分或达到 `maxRepeatAttempts`（最大重试次数）。

## 4. 关键参数配置

| 参数名 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `successRating` | `int` | `3` | 视为成功的最低评分（范围 1-4）。 |
| `maxRepeatAttempts` | `int` | `3` | 最大重试次数（不含初始调用）。 |
| `chatClientBuilder` | `Builder` | 必填 | 用于创建执行评估任务的内部 `ChatClient`。 |
| `promptTemplate` | `Template` | 预定义模板 | 定义评估标准、评分量表和返回格式。 |
| `advisorOrder` | `int` | 低优先级 | 顾问在执行链中的顺序。 |

## 5. 评估响应结构 (EvaluationResponse)
评估模型必须返回如下 JSON 结构的数据：
*   `rating` (int): 1-4 的评分。
*   `evaluation` (String): 对当前回答质量的详细理由。
*   `feedback` (String): 具体的、建设性的改进意见（将作为重试时的输入）。

## 6. 使用示例

```java
ChatClient chatClient = ChatClient.builder(mainModel)
    .defaultAdvisors(
        SelfRefineEvaluationAdvisor.builder()
            .chatClientBuilder(ChatClient.builder(judgeModel)) // 使用专门的评估模型
            .successRating(4)                                 // 要求必须达到优秀级别
            .maxRepeatAttempts(2)                             // 最多重试2次
            .build()
    )
    .build();

String response = chatClient.prompt("请帮我写一段关于量子计算的科普介绍").call().content();
```

## 7. 优缺点分析

### 优点
*   **质量保障**：有效减少模型幻觉和偏离主题的情况。
*   **反馈闭环**：通过具体的反馈引导模型不断逼近理想答案，比盲目重新生成效果更好。
*   **解耦**：评估逻辑封装在 Advisor 中，不干扰业务代码。

### 限制
*   **延迟增加**：每次评估和重试都会增加总体的响应时间。
*   **成本消耗**：多次调用模型会消耗更多的 Token 费用。
*   **不支持流式**：由于需要获取完整内容后才能打分，该组件目前仅支持同步调用（`adviseCall`），不支持流式输出（`adviseStream`）。

## 8. 适用场景
*   **高质量内容创作**：如公文写作、营销文案生成。
*   **复杂逻辑推理**：需要多步检查准确性的任务。
*   **指令遵循校验**：确保模型严格按照特定的风格或格式要求输出。

---
**文档版本**：1.0
**作者**：AI 研发团队
**日期**：2023-2025