# AgentX 企业级AI智能体平台 - 全量文档

![Java](https://img.shields.io/badge/Java-21-blue)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4-green)
![License](https://img.shields.io/badge/License-MIT-yellow)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)
![Low Resource](https://img.shields.io/badge/2C4G-Supported-orange)

## 📖 文档简介

欢迎阅读 AgentX 企业级 AI 智能体平台的完整文档。本文档面向三类读者：

- **开发者**：需要了解如何使用、扩展和定制 AgentX 框架
- **架构师**：关注系统设计、技术选型和性能优化
- **业务决策者**：希望评估 AgentX 的业务价值、适用场景和部署成本

文档基于 AgentX 实际代码和项目经验编写，涵盖从概念到生产部署的完整生命周期。

## 🎯 AgentX 是什么？

AgentX 是一个基于 Java Spring Boot 构建的企业级 AI 智能体平台，专为金融、政务等垂直场景优化。它不仅提供通用的 AI Agent 能力，更深度集成了行业特定的业务逻辑和安全合规要求。

### 核心优势

| 特性 | 说明 | 商业价值 |
|------|------|---------|
| **垂直领域优化** | 预置金融风控、政务流程等业务工具 | 开箱即用，减少定制开发成本 |
| **低配服务器支持** | 2核4G内存即可运行完整系统 | 降低硬件投入，适合中小企业 |
| **工作流编排** | 可视化流程设计，支持复杂业务逻辑 | 满足企业级业务流程需求 |
| **安全合规** | 敏感词过滤、审计日志、数据加密 | 满足金融/政务合规要求 |
| **一键部署** | Docker Compose 快速启动 | 降低运维门槛，快速上线 |

### 技术栈亮点

- **Java 21 + Spring Boot 3.4**：企业级稳定性与性能
- **LangChain + MCP 协议**：标准化 AI 工具生态
- **Milvus + Redis**：高效的向量检索与会话管理
- **虚拟线程**：高并发支持，资源利用率提升
- **OpenTelemetry**：完整的可观测性体系

## 🚀 快速开始

如果你是第一次接触 AgentX，建议按以下顺序阅读：

1. **[项目总览](01-project-overview/README.md)** - 了解 AgentX 的核心概念和价值主张
2. **[架构设计](02-architecture/README.md)** - 理解系统设计原理和组件关系
3. **[开发指南](03-development-guide/README.md)** - 学习如何开发和扩展 AgentX
4. **[配置部署](05-config-deployment/README.md)** - 掌握各种环境下的部署方法

### 5分钟快速体验

```bash
# 克隆项目
git clone https://github.com/your-org/agentx.git
cd agentx

# 使用 Docker Compose 启动（2C4G 环境优化）
docker-compose up -d

# 验证服务
curl http://localhost:8080/actuator/health

# 测试 AI 对话
curl -X POST http://localhost:8080/api/v1/agents/process \
  -H "Content-Type: application/json" \
  -d '{"message": "北京今天天气怎么样？", "agentType": "general"}'
```

## 📚 完整文档目录

### 第一部分：[项目总览](01-project-overview/)
- [项目简介](01-project-overview/01-introduction.md) - AgentX 是什么，解决什么问题
- [核心特性](01-project-overview/02-features.md) - 详细功能列表和技术亮点
- [适用场景](01-project-overview/03-use-cases.md) - 金融、政务等垂直行业应用案例
- [快速体验](01-project-overview/04-quick-start.md) - 5分钟上手教程

### 第二部分：[架构设计](02-architecture/)
- [系统架构图](02-architecture/01-system-architecture.md) - 分层架构与组件关系
- [核心组件详解](02-architecture/02-core-components.md) - Agent 服务、工作流编排、工具管理等
- [数据流与工作流](02-architecture/03-data-workflow.md) - 请求处理流程和状态管理
- [部署架构](02-architecture/04-deployment-architecture.md) - 开发、测试、生产环境设计

### 第三部分：[开发指南](03-development-guide/)
- [项目结构](03-development-guide/01-project-structure.md) - 代码目录说明和模块划分
- [环境搭建](03-development-guide/02-environment-setup.md) - 开发环境配置指南
- [添加新工具](03-development-guide/03-adding-tools.md) - 如何扩展工具市场
- [创建工作流](03-development-guide/04-creating-workflows.md) - 业务逻辑编排方法
- [扩展框架](03-development-guide/05-extending-framework.md) - 自定义 Agent 和集成新模型

### 第四部分：[API 参考](04-api-reference/)
- [REST API](04-api-reference/01-rest-api.md) - 所有端点的详细说明
- [MCP 协议接口](04-api-reference/02-mcp-api.md) - 标准化工具调用接口
- [工具调用示例](04-api-reference/03-tool-examples.md) - 内置工具使用示例

### 第五部分：[配置与部署](05-config-deployment/)
- [环境变量](05-config-deployment/01-environment-variables.md) - 关键配置项说明
- [配置文件详解](05-config-deployment/02-configuration-files.md) - application.yml 解析
- [低配服务器优化](05-config-deployment/03-low-resource-optimization.md) - 2C4G 环境调优指南
- [生产部署指南](05-config-deployment/04-production-deployment.md) - Kubernetes、云平台部署

### 第六部分：[安全与合规](06-security-compliance/)
- [安全特性](06-security-compliance/01-security-features.md) - 认证、授权、数据保护
- [合规指南](06-security-compliance/02-compliance-guide.md) - 金融/政务合规实现
- [灾备与恢复](06-security-compliance/03-disaster-recovery.md) - 高可用和数据备份方案

### 第七部分：[运维与监控](07-operations-monitoring/)
- [健康检查](07-operations-monitoring/01-health-checks.md) - 服务状态监控
- [性能监控](07-operations-monitoring/02-performance-monitoring.md) - 指标收集和告警
- [日志管理](07-operations-monitoring/03-log-management.md) - 结构化日志和聚合
- [故障排查](07-operations-monitoring/04-troubleshooting.md) - 常见问题解决方法

### 第八部分：[高级主题](08-advanced-topics/)
- [多智能体协作](08-advanced-topics/01-multi-agent-collaboration.md) - 复杂任务协同处理
- [自定义模型集成](08-advanced-topics/02-custom-model-integration.md) - 本地模型和第三方模型
- [大规模部署](08-advanced-topics/03-large-scale-deployment.md) - 水平扩展和跨区域部署

### 附录
- [常见问题（FAQ）](appendix/01-faq.md)
- [版本历史](appendix/02-version-history.md)
- [贡献指南](appendix/03-contributing-guide.md)
- [资源链接](appendix/04-resource-links.md)

## 🔄 文档更新说明

本文档基于以下来源整合：
- `D:\workspace\agentx` - 实际项目代码和文档
- `D:\res\study-notes\AgentX` - 技术文章、营销材料和开发计划
- 实际部署和调优经验

文档会持续更新，最新版本请查看 [GitHub](https://github.com/your-org/agentx)。

## 🤝 获取帮助

- **GitHub Issues**: [问题反馈](https://github.com/your-org/agentx/issues)
- **Discord 社区**: [技术交流](https://discord.gg/agentx)
- **邮箱支持**: support@agentx.example.com

## 📄 许可证

AgentX 采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件。

---

<p align="center">
  Made with ❤️ by the AgentX Team | 专为金融/政务场景优化的企业级 AI 智能体平台
</p>