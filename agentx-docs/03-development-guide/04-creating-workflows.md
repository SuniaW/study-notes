# 创建工作流

本章详细说明如何在 AgentX 平台中创建工作流，包括工作流定义、DSL 语法、执行引擎和监控调试。

## 工作流系统概述

### 工作流概念
工作流是一系列有序或并行执行的步骤，用于自动化复杂的业务逻辑。AgentX 的工作流系统支持：
- **顺序执行**：步骤按顺序执行
- **条件分支**：基于条件选择不同路径
- **并行执行**：多个步骤同时执行
- **循环执行**：重复执行某些步骤
- **错误处理**：失败时的重试和补偿

### 工作流架构
```
┌─────────────────────────────────────────┐
│           工作流定义 (DSL/YAML)          │
└─────────────────────┬───────────────────┘
                      │ 解析
┌─────────────────────▼───────────────────┐
│           工作流执行引擎                │
│  ┌─────────────┐  ┌─────────────┐      │
│  │  步骤执行器 │  │  状态管理   │      │
│  └─────────────┘  └─────────────┘      │
│  ┌─────────────┐  ┌─────────────┐      │
│  │  条件评估   │  │  错误处理   │      │
│  └─────────────┘  └─────────────┘      │
└─────────────────────┬───────────────────┘
                      │ 执行
┌─────────────────────▼───────────────────┐
│             步骤实现                    │
│  • Agent 步骤    • 工具步骤             │
│  • 条件步骤      • 并行步骤             │
│  • 循环步骤      • 自定义步骤           │
└─────────────────────────────────────────┘
```

## 工作流定义

### 1. YAML DSL（推荐）

#### 基本结构
```yaml
name: "客户服务工单处理"
version: "1.0"
description: "自动化处理客户服务工单的工作流"

# 输入变量定义
variables:
  - name: "ticketId"
    type: "string"
    required: true
    description: "工单ID"
  - name: "customerId"
    type: "string"
    required: true
    description: "客户ID"

# 输出变量定义
outputs:
  - name: "resolution"
    type: "string"
    description: "解决方案"
  - name: "satisfactionScore"
    type: "number"
    description: "满意度评分"

# 工作流步骤
steps:
  # 步骤1：获取工单信息
  - id: "get-ticket-info"
    type: "tool"
    name: "获取工单信息"
    tool: "ticket-query"
    parameters:
      ticketId: "{{ticketId}}"
    next: "classify-ticket"
    
  # 步骤2：分类工单
  - id: "classify-ticket"
    type: "agent"
    name: "工单分类"
    agent: "ticket-classifier"
    parameters:
      ticketDescription: "{{ticket.description}}"
      customerHistory: "{{customer.history}}"
    next: "route-ticket"
    
  # 步骤3：路由工单（条件分支）
  - id: "route-ticket"
    type: "condition"
    name: "工单路由"
    condition: "{{ticket.category}} == 'technical'"
    trueNext: "handle-technical"
    falseNext: "handle-billing"
    
  # 步骤4：处理技术问题
  - id: "handle-technical"
    type: "parallel"
    name: "并行处理技术问题"
    branches:
      - steps:
        - id: "analyze-logs"
          type: "tool"
          name: "分析日志"
          tool: "log-analyzer"
          parameters:
            ticketId: "{{ticketId}}"
        - id: "check-knowledge-base"
          type: "tool"
          name: "查询知识库"
          tool: "knowledge-base-search"
          parameters:
            query: "{{ticket.description}}"
    next: "generate-solution"
    
  # 步骤5：处理账单问题
  - id: "handle-billing"
    type: "agent"
    name: "处理账单问题"
    agent: "billing-agent"
    parameters:
      customerId: "{{customerId}}"
      ticketDetails: "{{ticket.details}}"
    next: "generate-solution"
    
  # 步骤6：生成解决方案
  - id: "generate-solution"
    type: "agent"
    name: "生成解决方案"
    agent: "solution-generator"
    parameters:
      analysisResults: "{{previousResults}}"
      customerPreferences: "{{customer.preferences}}"
    outputs:
      resolution: "{{agentResponse}}"
    next: "evaluate-satisfaction"
    
  # 步骤7：评估满意度
  - id: "evaluate-satisfaction"
    type: "tool"
    name: "评估满意度"
    tool: "satisfaction-predictor"
    parameters:
      resolution: "{{resolution}}"
      customerHistory: "{{customer.history}}"
    outputs:
      satisfactionScore: "{{predictionScore}}"
    next: "end"
```

### 2. JSON DSL

#### 基本结构
```json
{
  "name": "风控审批流程",
  "version": "1.0",
  "description": "自动化风控审批工作流",
  "variables": [
    {
      "name": "applicantId",
      "type": "string",
      "required": true,
      "description": "申请人ID"
    },
    {
      "name": "loanAmount",
      "type": "number",
      "required": true,
      "description": "贷款金额"
    }
  ],
  "steps": [
    {
      "id": "identity-verification",
      "type": "tool",
      "name": "身份验证",
      "tool": "identity-verifier",
      "parameters": {
        "applicantId": "{{applicantId}}"
      },
      "next": "credit-check"
    },
    {
      "id": "credit-check",
      "type": "parallel",
      "name": "并行信用检查",
      "branches": [
        {
          "steps": [
            {
              "id": "check-credit-score",
              "type": "tool",
              "name": "查询信用分",
              "tool": "credit-score-query",
              "parameters": {
                "applicantId": "{{applicantId}}"
              }
            }
          ]
        },
        {
          "steps": [
            {
              "id": "check-debt-ratio",
              "type": "tool",
              "name": "计算负债比",
              "tool": "debt-ratio-calculator",
              "parameters": {
                "applicantId": "{{applicantId}}",
                "loanAmount": "{{loanAmount}}"
              }
            }
          ]
        }
      ],
      "next": "risk-assessment"
    }
  ]
}
```

### 3. 代码定义（Java API）

#### 编程式定义
```java
@Component
public class RiskApprovalWorkflow implements WorkflowDefinition {
    
    @Override
    public String getName() {
        return "risk-approval-workflow";
    }
    
    @Override
    public List<WorkflowVariable> getVariables() {
        return List.of(
            new WorkflowVariable("applicantId", String.class, true),
            new WorkflowVariable("loanAmount", BigDecimal.class, true)
        );
    }
    
    @Override
    public List<WorkflowStep> getSteps() {
        return List.of(
            // 步骤1：身份验证
            ToolStep.builder()
                .id("identity-verification")
                .name("身份验证")
                .toolName("identity-verifier")
                .parameters(Map.of("applicantId", "{{applicantId}}"))
                .nextStepId("credit-check")
                .build(),
            
            // 步骤2：信用检查（并行）
            ParallelStep.builder()
                .id("credit-check")
                .name("并行信用检查")
                .branches(List.of(
                    Branch.builder()
                        .steps(List.of(
                            ToolStep.builder()
                                .id("check-credit-score")
                                .name("查询信用分")
                                .toolName("credit-score-query")
                                .parameters(Map.of("applicantId", "{{applicantId}}"))
                                .build()
                        ))
                        .build(),
                    Branch.builder()
                        .steps(List.of(
                            ToolStep.builder()
                                .id("check-debt-ratio")
                                .name("计算负债比")
                                .toolName("debt-ratio-calculator")
                                .parameters(Map.of(
                                    "applicantId", "{{applicantId}}",
                                    "loanAmount", "{{loanAmount}}"
                                ))
                                .build()
                        ))
                        .build()
                ))
                .nextStepId("risk-assessment")
                .build(),
            
            // 步骤3：风险评估（条件分支）
            ConditionStep.builder()
                .id("risk-assessment")
                .name("风险评估")
                .condition("{{creditScore}} >= 650 && {{debtRatio}} <= 0.4")
                .trueNextStepId("auto-approval")
                .falseNextStepId("manual-review")
                .build()
        );
    }
}
```

## 步骤类型详解

### 1. Agent 步骤

#### 定义
```yaml
- id: "analyze-sentiment"
  type: "agent"
  name: "情感分析"
  agent: "sentiment-analyzer"
  parameters:
    text: "{{customerFeedback}}"
    language: "{{language}}"
  outputs:
    sentiment: "{{agentResponse.sentiment}}"
    confidence: "{{agentResponse.confidence}}"
  next: "generate-response"
```

#### 对应 Java 类
```java
public class AgentStep extends WorkflowStep {
    
    private String agentName;
    private Map<String, Object> parameters;
    private Map<String, String> outputs;
    
    @Override
    public StepResult execute(WorkflowContext context) {
        // 1. 准备参数
        Map<String, Object> resolvedParams = resolveParameters(parameters, context);
        
        // 2. 调用 Agent
        AgentRequest request = new AgentRequest();
        request.setAgentType(agentName);
        request.setParameters(resolvedParams);
        
        AgentResponse response = agentService.process(request);
        
        // 3. 提取输出
        if (outputs != null) {
            outputs.forEach((outputName, expression) -> {
                Object value = extractValue(response, expression);
                context.setVariable(outputName, value);
            });
        }
        
        return StepResult.success(response);
    }
}
```

### 2. 工具步骤

#### 定义
```yaml
- id: "calculate-tax"
  type: "tool"
  name: "计算税费"
  tool: "tax-calculator"
  parameters:
    amount: "{{invoiceAmount}}"
    province: "{{customerProvince}}"
    productType: "{{productType}}"
  outputs:
    taxAmount: "{{toolResult.tax}}"
    totalAmount: "{{toolResult.total}}"
  retry:
    maxAttempts: 3
    backoffMs: 1000
  timeout: 5000
  next: "generate-receipt"
```

#### 对应 Java 类
```java
public class ToolStep extends WorkflowStep {
    
    private String toolName;
    private Map<String, Object> parameters;
    private RetryConfig retryConfig;
    private long timeoutMs = 10000;
    
    @Override
    public StepResult execute(WorkflowContext context) {
        // 1. 准备参数
        Map<String, Object> resolvedParams = resolveParameters(parameters, context);
        
        // 2. 构建工具请求
        ToolRequest request = new ToolRequest();
        request.setToolName(toolName);
        request.setParameters(resolvedParams);
        
        // 3. 执行工具（支持重试）
        ToolResponse response = executeWithRetry(() -> {
            return toolRegistry.execute(toolName, request);
        }, retryConfig);
        
        // 4. 处理结果
        if (response.isSuccess()) {
            return StepResult.success(response.getData());
        } else {
            return StepResult.failure(response.getError());
        }
    }
    
    private <T> T executeWithRetry(Callable<T> task, RetryConfig config) {
        int attempts = 0;
        long delay = config.getInitialDelayMs();
        
        while (attempts <= config.getMaxAttempts()) {
            try {
                return task.call();
            } catch (Exception e) {
                attempts++;
                if (attempts > config.getMaxAttempts()) {
                    throw new RuntimeException("Max retries exceeded", e);
                }
                
                // 等待后重试
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Retry interrupted", ie);
                }
                
                // 指数退避
                delay = Math.min(delay * 2, config.getMaxDelayMs());
            }
        }
        
        throw new RuntimeException("Should not reach here");
    }
}
```

### 3. 条件步骤

#### 定义
```yaml
- id: "risk-decision"
  type: "condition"
  name: "风险决策"
  condition: >
    {{creditScore}} >= 700 && 
    {{income}} >= 50000 && 
    {{employmentYears}} >= 2
  trueNext: "approve-loan"
  falseNext: "request-more-info"
  description: "基于信用分、收入和工龄的自动决策"
```

#### 对应 Java 类
```java
public class ConditionStep extends WorkflowStep {
    
    private String condition;
    private String trueNextStepId;
    private String falseNextStepId;
    
    @Override
    public StepResult execute(WorkflowContext context) {
        // 1. 解析条件表达式
        Expression expression = parseExpression(condition);
        
        // 2. 评估条件
        boolean result = evaluateCondition(expression, context);
        
        // 3. 返回下一步
        String nextStepId = result ? trueNextStepId : falseNextStepId;
        
        StepResult stepResult = StepResult.success(result);
        stepResult.setNextStepId(nextStepId);
        
        return stepResult;
    }
    
    private boolean evaluateCondition(Expression expression, WorkflowContext context) {
        // 使用 Spring Expression Language (SpEL) 评估条件
        StandardEvaluationContext evalContext = new StandardEvaluationContext();
        evalContext.setVariables(context.getVariables());
        
        return expression.getValue(evalContext, Boolean.class);
    }
}
```

### 4. 并行步骤

#### 定义
```yaml
- id: "parallel-checks"
  type: "parallel"
  name: "并行检查"
  branches:
    - steps:
      - id: "check-blacklist"
        type: "tool"
        name: "检查黑名单"
        tool: "blacklist-checker"
        parameters:
          applicantId: "{{applicantId}}"
      - id: "check-fraud"
        type: "tool"
        name: "检查欺诈"
        tool: "fraud-detector"
        parameters:
          applicantId: "{{applicantId}}"
          transactionAmount: "{{loanAmount}}"
    - steps:
      - id: "verify-documents"
        type: "agent"
        name: "验证文档"
        agent: "document-verifier"
        parameters:
          documents: "{{applicantDocuments}}"
  next: "consolidate-results"
  timeout: 30000
  failFast: true
```

#### 对应 Java 类
```java
public class ParallelStep extends WorkflowStep {
    
    private List<Branch> branches;
    private long timeoutMs = 30000;
    private boolean failFast = true;
    
    @Override
    public StepResult execute(WorkflowContext context) {
        // 1. 创建执行器服务
        ExecutorService executor = Executors.newFixedThreadPool(branches.size());
        
        // 2. 提交分支任务
        List<Future<BranchResult>> futures = new ArrayList<>();
        for (Branch branch : branches) {
            Callable<BranchResult> task = () -> executeBranch(branch, context);
            futures.add(executor.submit(task));
        }
        
        // 3. 等待所有任务完成
        List<BranchResult> results = new ArrayList<>();
        try {
            for (Future<BranchResult> future : futures) {
                try {
                    BranchResult result = future.get(timeoutMs, TimeUnit.MILLISECONDS);
                    results.add(result);
                    
                    // fail-fast 模式：有失败立即返回
                    if (failFast && !result.isSuccess()) {
                        cancelOtherFutures(futures);
                        return StepResult.failure("分支执行失败: " + result.getError());
                    }
                } catch (TimeoutException e) {
                    cancelOtherFutures(futures);
                    return StepResult.failure("并行执行超时");
                }
            }
        } catch (Exception e) {
            cancelOtherFutures(futures);
            return StepResult.failure("并行执行异常: " + e.getMessage());
        } finally {
            executor.shutdownNow();
        }
        
        // 4. 合并结果
        Map<String, Object> consolidated = consolidateResults(results);
        return StepResult.success(consolidated);
    }
    
    private BranchResult executeBranch(Branch branch, WorkflowContext context) {
        // 复制上下文（避免并发修改）
        WorkflowContext branchContext = context.copy();
        
        // 顺序执行分支中的步骤
        for (WorkflowStep step : branch.getSteps()) {
            StepResult result = step.execute(branchContext);
            if (!result.isSuccess()) {
                return BranchResult.failure(result.getError());
            }
        }
        
        return BranchResult.success(branchContext.getVariables());
    }
}
```

### 5. 循环步骤

#### 定义
```yaml
- id: "process-items"
  type: "loop"
  name: "处理项目列表"
  items: "{{itemList}}"
  itemVariable: "currentItem"
  indexVariable: "currentIndex"
  steps:
    - id: "validate-item"
      type: "tool"
      name: "验证项目"
      tool: "item-validator"
      parameters:
        item: "{{currentItem}}"
        index: "{{currentIndex}}"
    - id: "process-item"
      type: "agent"
      name: "处理项目"
      agent: "item-processor"
      parameters:
        item: "{{currentItem}}"
  next: "summarize-results"
  maxIterations: 100
  breakCondition: "{{validationFailed}} == true"
```

#### 对应 Java 类
```java
public class LoopStep extends WorkflowStep {
    
    private String itemsExpression;
    private String itemVariable;
    private String indexVariable;
    private List<WorkflowStep> steps;
    private int maxIterations = 100;
    private String breakCondition;
    
    @Override
    public StepResult execute(WorkflowContext context) {
        // 1. 获取迭代集合
        Object itemsObj = evaluateExpression(itemsExpression, context);
        if (!(itemsObj instanceof Iterable)) {
            return StepResult.failure("items 表达式必须返回可迭代对象");
        }
        
        Iterable<?> items = (Iterable<?>) itemsObj;
        List<Object> results = new ArrayList<>();
        int iteration = 0;
        
        // 2. 遍历集合
        for (Object item : items) {
            iteration++;
            
            // 检查最大迭代次数
            if (iteration > maxIterations) {
                return StepResult.failure("超过最大迭代次数: " + maxIterations);
            }
            
            // 设置当前项和索引
            context.setVariable(itemVariable, item);
            context.setVariable(indexVariable, iteration - 1);
            
            // 执行循环体步骤
            for (WorkflowStep step : steps) {
                StepResult result = step.execute(context);
                if (!result.isSuccess()) {
                    return StepResult.failure("第 " + iteration + " 次迭代失败: " + result.getError());
                }
            }
            
            // 收集结果
            results.add(context.getVariable("result"));
            
            // 检查中断条件
            if (breakCondition != null) {
                Boolean shouldBreak = evaluateExpression(breakCondition, context, Boolean.class);
                if (Boolean.TRUE.equals(shouldBreak)) {
                    break;
                }
            }
        }
        
        // 3. 清理临时变量
        context.removeVariable(itemVariable);
        context.removeVariable(indexVariable);
        
        return StepResult.success(results);
    }
}
```

## 工作流执行

### 1. 启动工作流

#### REST API
```http
POST /api/v1/workflows/execute
Content-Type: application/json

{
  "workflowId": "customer-service-workflow",
  "parameters": {
    "ticketId": "TICKET-123456",
    "customerId": "CUST-789012"
  },
  "options": {
    "timeout": 300000,
    "priority": "normal",
    "callbackUrl": "https://example.com/callback"
  }
}
```

#### Java API
```java
@Service
public class WorkflowService {
    
    @Autowired
    private WorkflowExecutor workflowExecutor;
    
    public WorkflowExecution startWorkflow(String workflowId, 
                                          Map<String, Object> parameters) {
        
        // 创建工作流执行请求
        WorkflowExecutionRequest request = new WorkflowExecutionRequest();
        request.setWorkflowId(workflowId);
        request.setParameters(parameters);
        request.setExecutionId(UUID.randomUUID().toString());
        request.setStartedAt(LocalDateTime.now());
        
        // 异步执行工作流
        CompletableFuture<WorkflowResult> future = 
            workflowExecutor.executeAsync(request);
        
        // 返回执行句柄
        WorkflowExecution execution = new WorkflowExecution();
        execution.setExecutionId(request.getExecutionId());
        execution.setStatus(WorkflowStatus.RUNNING);
        execution.setFuture(future);
        
        // 注册回调
        future.thenAccept(result -> {
            handleWorkflowCompletion(request.getExecutionId(), result);
        });
        
        return execution;
    }
}
```

### 2. 工作流上下文

#### 上下文结构
```java
public class WorkflowContext {
    
    private String executionId;
    private String workflowId;
    private Map<String, Object> variables;
    private List<WorkflowStep> executionHistory;
    private LocalDateTime startTime;
    private LocalDateTime lastUpdated;
    
    // 变量管理方法
    public void setVariable(String name, Object value) { ... }
    public Object getVariable(String name) { ... }
    public boolean hasVariable(String name) { ... }
    
    // 历史记录方法
    public void addStepExecution(StepExecution step) { ... }
    public List<StepExecution> getStepHistory() { ... }
    
    // 上下文复制（用于并行执行）
    public WorkflowContext copy() { ... }
}
```

### 3. 状态管理

#### 状态持久化
```java
@Component
public class WorkflowStateStore {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void saveState(String executionId, WorkflowContext context) {
        // 序列化上下文
        String key = "workflow:state:" + executionId;
        redisTemplate.opsForValue().set(key, context, 24, TimeUnit.HOURS);
        
        // 保存检查点
        saveCheckpoint(executionId, context);
    }
    
    public WorkflowContext loadState(String executionId) {
        String key = "workflow:state:" + executionId;
        return (WorkflowContext) redisTemplate.opsForValue().get(key);
    }
    
    public void saveCheckpoint(String executionId, WorkflowContext context) {
        String key = "workflow:checkpoint:" + executionId + ":" + 
                    context.getCurrentStepId();
        redisTemplate.opsForValue().set(key, context, 7, TimeUnit.DAYS);
    }
}
```

## 错误处理和重试

### 1. 错误处理策略

#### 步骤级错误处理
```yaml
- id: "call-external-api"
  type: "tool"
  name: "调用外部API"
  tool: "external-api-caller"
  parameters:
    endpoint: "https://api.example.com/data"
  errorHandling:
    retry:
      maxAttempts: 3
      backoffMs: 1000
      backoffMultiplier: 2
    fallback:
      type: "tool"
      tool: "local-data-provider"
      parameters:
        query: "{{originalQuery}}"
    timeout: 5000
  next: "process-response"
```

#### 工作流级错误处理
```yaml
errorHandling:
  global:
    maxRetries: 2
    timeout: 300000
  steps:
    - stepId: "critical-step"
      actions:
        - type: "notify"
          channel: "slack"
          message: "关键步骤失败: {{error}}"
        - type: "rollback"
          steps: ["step-1", "step-2"]
        - type: "compensate"
          compensationWorkflow: "compensation-workflow"
```

### 2. 补偿事务

#### 补偿工作流定义
```yaml
name: "order-cancellation-compensation"
description: "订单取消的补偿工作流"

steps:
  - id: "release-inventory"
    type: "tool"
    name: "释放库存"
    tool: "inventory-manager"
    parameters:
      orderId: "{{orderId}}"
      action: "release"
    
  - id: "refund-payment"
    type: "tool"
    name: "退款处理"
    tool: "payment-processor"
    parameters:
      orderId: "{{orderId}}"
      amount: "{{orderAmount}}"
    
  - id: "notify-customer"
    type: "agent"
    name: "通知客户"
    agent: "notification-agent"
    parameters:
      customerId: "{{customerId}}"
      message: "您的订单已取消，退款已处理"
```

## 监控和调试

### 1. 工作流监控

#### 监控指标
```java
@Component
public class WorkflowMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordWorkflowStart(String workflowId) {
        meterRegistry.counter("workflow.started", "workflowId", workflowId).increment();
    }
    
    public void recordStepExecution(String workflowId, String stepId, 
                                   long duration, boolean success) {
        Timer timer = meterRegistry.timer("workflow.step.duration", 
            "workflowId", workflowId, "stepId", stepId);
        timer.record(duration, TimeUnit.MILLISECONDS);
        
        meterRegistry.counter("workflow.step.executed",
            "workflowId", workflowId, "stepId", stepId, "success", String.valueOf(success))
            .increment();
    }
    
    public void recordWorkflowCompletion(String workflowId, 
                                        WorkflowStatus status, 
                                        long totalDuration) {
        meterRegistry.counter("workflow.completed",
            "workflowId", workflowId, "status", status.name())
            .increment();
        
        Timer timer = meterRegistry.timer("workflow.total.duration",
            "workflowId", workflowId, "status", status.name());
        timer.record(totalDuration, TimeUnit.MILLISECONDS);
    }
}
```

### 2. 调试工具

#### 工作流调试端点
```java
@RestController
@RequestMapping("/api/v1/workflows/debug")
public class WorkflowDebugController {
    
    @Autowired
    private WorkflowExecutor workflowExecutor;
    
    @GetMapping("/{executionId}/state")
    public WorkflowContext getWorkflowState(@PathVariable String executionId) {
        return workflowExecutor.getExecutionState(executionId);
    }
    
    @PostMapping("/{executionId}/step")
    public StepResult executeStep(@PathVariable String executionId,
                                 @RequestBody DebugStepRequest request) {
        return workflowExecutor.debugStep(executionId, 
                                         request.getStepId(),
                                         request.getParameters());
    }
    
    @PostMapping("/{executionId}/resume")
    public WorkflowResult resumeWorkflow(@PathVariable String executionId,
                                        @RequestBody ResumeRequest request) {
        return workflowExecutor.resumeFromCheckpoint(executionId,
                                                    request.getCheckpointId());
    }
}
```

#### 可视化调试界面
```html
<!-- 工作流执行可视化 -->
<div id="workflow-visualization">
  <div class="workflow-diagram">
    <!-- 动态渲染工作流步骤 -->
  </div>
  
  <div class="execution-details">
    <div class="variables-panel">
      <h3>变量状态</h3>
      <table id="variables-table">
        <!-- 显示变量当前值 -->
      </table>
    </div>
    
    <div class="history-panel">
      <h3>执行历史</h3>
      <ul id="execution-history">
        <!-- 显示步骤执行历史 -->
      </ul>
    </div>
    
    <div class="controls-panel">
      <button onclick="pauseExecution()">暂停</button>
      <button onclick="stepOver()">单步执行</button>
      <button onclick="resumeExecution()">继续</button>
      <button onclick="inspectVariable('varName')">检查变量</button>
    </div>
  </div>
</div>
```

## 最佳实践

### 1. 设计原则
- **单一职责**：每个步骤只做一件事
- **可组合性**：步骤可以组合成更复杂的工作流
- **幂等性**：工作流可以安全重试
- **可观测性**：每个步骤都有完整的监控

### 2. 性能优化
- **并行化**：独立步骤使用并行执行
- **缓存**：频繁使用的数据添加缓存
- **批处理**：相似操作批量处理
- **异步执行**：耗时操作异步执行

### 3. 错误处理
- **优雅降级**：失败时提供降级方案
- **重试策略**：合理设置重试次数和间隔
- **补偿事务**：失败时执行补偿操作
- **告警通知**：关键失败及时通知

### 4. 可维护性
- **文档化**：每个工作流都有详细文档
- **版本控制**：工作流定义进行版本管理
- **测试覆盖**：工作流有完整的测试用例
- **配置外置**：参数和配置与代码分离

## 示例：完整工作流

### 电商订单处理工作流
```yaml
name: "ecommerce-order-processing"
version: "2.1"
description: "电商订单全流程处理"

variables:
  - name: "orderId"
    type: "string"
    required: true
  - name: "customerId"
    type: "string"
    required: true

steps:
  # 阶段1：订单验证
  - id: "validate-order"
    type: "parallel"
    name: "并行订单验证"
    branches:
      - steps:
        - id: "check-inventory"
          type: "tool"
          name: "检查库存"
          tool: "inventory-checker"
          parameters:
            orderId: "{{orderId}}"
        - id: "validate-address"
          type: "tool"
          name: "验证地址"
          tool: "address-validator"
          parameters:
            customerId: "{{customerId}}"
      - steps:
        - id: "fraud-check"
          type: "agent"
          name: "欺诈检测"
          agent: "fraud-detector"
          parameters:
            orderId: "{{orderId}}"
            customerId: "{{customerId}}"
    next: "process-payment"
    
  # 阶段2：支付处理
  - id: "process-payment"
    type: "tool"
    name: "处理支付"
    tool: "payment-processor"
    parameters:
      orderId: "{{orderId}}"
      amount: "{{orderTotal}}"
    retry:
      maxAttempts: 3
      backoffMs: 2000
    next: "fulfill-order"
    
  # 阶段3：订单履行
  - id: "fulfill-order"
    type: "parallel"
    name: "并行订单履行"
    branches:
      - steps:
        - id: "pick-items"
          type: "tool"
          name: "拣货"
          tool: "warehouse-picker"
          parameters:
            orderId: "{{orderId}}"
        - id: "pack-order"
          type: "tool"
          name: "打包"
          tool: "packing-system"
          parameters:
            orderId: "{{orderId}}"
      - steps:
        - id: "arrange-shipping"
          type: "agent"
          name: "安排物流"
          agent: "shipping-agent"
          parameters:
            orderId: "{{orderId}}"
            destination: "{{customerAddress}}"
    next: "notify-customer"
    
  # 阶段4：客户通知
  - id: "notify-customer"
    type: "parallel"
    name: "并行客户通知"
    branches:
      - steps:
        - id: "send-confirmation"
          type: "tool"
          name: "发送确认邮件"
          tool: "email-sender"
          parameters:
            to: "{{customerEmail}}"
            template: "order-confirmation"
            variables:
              orderId: "{{orderId}}"
        - id: "send-sms"
          type: "tool"
          name: "发送短信通知"
          tool: "sms-sender"
          parameters:
            phone: "{{customerPhone}}"
            message: "您的订单 {{orderId}} 已确认"
      - steps:
        - id: "update-dashboard"
          type: "tool"
          name: "更新客户面板"
          tool: "customer-dashboard"
          parameters:
            customerId: "{{customerId}}"
            orderId: "{{orderId}}"
            status: "confirmed"
    next: "end"
```

## 下一步

创建工作流后，建议：

1. **测试工作流**：使用调试工具测试工作流执行
2. **监控性能**：设置监控指标，观察工作流性能
3. **优化设计**：根据监控数据优化工作流设计
4. **文档完善**：完善工作流文档和使用说明

如果需要更高级的功能，请查看 [扩展框架](05-extending-framework.md) 章节。