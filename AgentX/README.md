# AgentX技术文档集

本目录包含AgentX项目的完整技术文档，从架构设计到具体实现细节。

## 文档索引

| 文档 | 内容概述 | 最后更新 |
|------|----------|----------|
| [AgentX技术深度解析](./AgentX技术深度解析.md) | 综合技术文档，包含架构、实现、问题解决方案 | 2026-04-10 |
| [05.md](./05.md) | Spring Boot 3.4.4与Milvus V2 SDK集成实践 | 2026-04-10 |
| [01.md](./01.md) | 项目介绍与背景 | 2026-04-08 |
| [02.md](./02.md) | 架构设计文档 | 2026-04-09 |
| [03.md](./03.md) | 核心组件实现 | 2026-04-09 |
| [04.md](./04.md) | 部署与运维指南 | 2026-04-10 |

## 快速开始

### 1. 项目编译
```bash
# 克隆项目
git clone <repository-url>

# 编译项目
mvn clean compile -DskipTests
```

### 2. 环境配置
```bash
# 设置环境变量（Windows PowerShell）
$env:OPENAI_API_KEY="your-api-key"
$env:MILVUS_ENABLED="false"  # 开发环境可禁用Milvus
```

### 3. 启动应用
```bash
# 开发环境启动
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

## 核心特性

### 现代化技术栈
- **Spring Boot 3.4.4** - 最新企业级框架
- **Java 21** - 虚拟线程支持
- **LangChain4j 1.x** - AI Agent框架
- **Milvus V2** - 向量数据库
- **Redis 7.x** - 分布式缓存

### 工程化解决方案
- ✅ HTTP客户端冲突解决
- ✅ LangChain4j 1.x API适配
- ✅ 条件化Bean配置
- ✅ 虚拟线程性能优化
- ✅ 防御性编程实践

## 问题解决

### 常见问题
1. **HTTP客户端冲突** - 参考[技术文档](./AgentX技术深度解析.md#4-编译问题与解决方案)
2. **LangChain4j API变更** - 参考[API适配章节](./AgentX技术深度解析.md#321-api兼容性处理)
3. **Milvus连接问题** - 参考[向量存储实战](./AgentX技术深度解析.md#6-向量存储实战)

### 性能优化
- 虚拟线程配置
- 批量向量操作
- 最小化网络负载
- Redis序列化优化

## 扩展阅读

- [Spring Boot官方文档](https://docs.spring.io/spring-boot/docs/3.4.4/reference/html/)
- [LangChain4j文档](https://docs.langchain4j.dev/)
- [Milvus V2指南](https://milvus.io/docs/v2.0.x/)

## 贡献指南

欢迎提交Issue和Pull Request：
1. Fork项目仓库
2. 创建特性分支
3. 提交更改
4. 创建Pull Request

## 许可证

本项目采用MIT许可证。详细信息请参阅LICENSE文件。

---

**维护团队**: AgentX技术团队  
**文档版本**: 1.0.0  
**最后更新**: 2026-04-10