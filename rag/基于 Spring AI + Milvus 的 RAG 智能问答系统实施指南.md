

# 📘 基于 Spring AI + Milvus 的 RAG 智能问答系统实施指南

**版本控制**：v1.0 (终极融合版)
**核心目标**：构建一个支持本地文档（PDF/Word）解析、向量存储检索、大模型流式响应（SSE）的智能政务/企业问答系统。
**技术底座**：`Spring Boot 3.4.3` + `Spring AI 1.0.0-M1` + `Milvus` + `Ollama` + `Vue3` + `JDK 21`

---

## 📑 目录
1. [方案与硬件选型](#1-方案与硬件选型)
2. [基础设施部署 (服务器/容器)](#2-基础设施部署-服务器容器)
3. [AI 模型层配置 (Ollama)](#3-ai-模型层配置-ollama)
4. [后端工程开发 (Spring Boot)](#4-后端工程开发-spring-boot)
5. [前端交互实现 (Vue 3)](#5-前端交互实现-vue-3)
6. [上线前核对清单](#6-上线前核对清单)

---

## 1. 方案与硬件选型

实施前，请务必根据实际服务器资源评估并选择对应的技术路线：

| 评估维度 | 🌟 路线 A：标准企业级方案 (推荐) | 💻 路线 B：低配实验方案 (无 GPU) |
| :--- | :--- | :--- |
| **适用场景** | 生产环境、演示汇报、对准确率要求高 | 个人学习验证、低配阿里云 ECS 环境 |
| **硬件底线** | ≥ 16GB 内存，建议配备 NVIDIA 显卡 | 2核4G / 4核8G，纯 CPU 运算 |
| **对话大模型** | `qwen:7b` (4-bit 量化，逻辑严密) | `qwen2.5:1.5b` (极致轻量，速度极快) |
| **向量模型** | `bge-m3` (多语言优异，维度 **1024**) | `bge-m3` (强烈推荐统一使用此模型) |
| **预期性能** | 首字延迟 < 1s，推理顺滑 | 首字延迟 1~3s，勉强流畅 |
| **保命手段** | 模型常驻显存、GPU 硬件加速 | **必须开启 Swap 虚拟内存**、限制 JVM |

---

## 2. 基础设施部署 (服务器/容器)

### 2.1 内存扩容防崩溃 (仅低配机器必做)
若服务器内存不足 8G（如阿里云 2C4G），为防止编译 Java 或启动 Milvus 时触发系统 OOM（内存溢出），需预先挂载虚拟内存：

```bash
# 1. 创建 8G 虚拟内存文件并赋权
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. 设置开机永久生效
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 3. 验证是否成功 (查看 Swap 分区是否不为 0)
free -h
```

### 2.2 启动 Milvus 向量数据库
采用 Docker Standalone 模式快速拉起服务：

```bash
# 1. 下载官方 Docker Compose 文件
wget https://github.com/milvus-io/milvus/releases/download/v2.3.0/milvus-standalone-docker-compose.yml -O docker-compose.yml

# (可选) 低配机器建议编辑 docker-compose.yml，为 milvus 容器添加 mem_limit: 2g

# 2. 启动服务
sudo docker compose up -d
```

---

## 3. AI 模型层配置 (Ollama)

### 3.1 拉取所需大模型
> 💡 **最佳实践**：无论硬件高低，向量模型均强烈推荐使用官方库自带的 `bge-m3`，避免手动编译小型模型的繁琐，且 RAG 检索精度大幅提升。

```bash
# 1. 路线 A (高配) 对话模型：
ollama pull qwen:7b   

# 2. 路线 B (低配) 对话模型：
ollama pull qwen2.5:1.5b

# 3. 统一向量模型 (Embedding 维度: 1024)
ollama pull bge-m3    
```

### 3.2 开启跨域与远程访问 (解决 Connection refused)
默认 Ollama 仅限本地（`127.0.0.1`）访问，云端部署必须修改监听地址：

```bash
# 1. 直接创建并写入 systemd 覆盖配置
sudo mkdir -p /etc/systemd/system/ollama.service.d/
echo -e '[Service]\nEnvironment="OLLAMA_HOST=0.0.0.0"' | sudo tee /etc/systemd/system/ollama.service.d/override.conf

# 2. 重载配置并重启服务
sudo systemctl daemon-reload
sudo systemctl restart ollama

# 3. 验证端口监听 (应看到 0.0.0.0:11434 或 :::11434)
netstat -tulnp | grep 11434
```
⚠️ **云安全策略**：请在阿里云/腾讯云控制台的“安全组”中，放行入方向的 **`11434`** (Ollama) 和 **`19530`** (Milvus) 端口。

---

## 4. 后端工程开发 (Spring Boot)

### 4.1 核心依赖配置 (解决 JDK 21 与 Lombok 冲突)
出现 `TypeTag :: UNKNOWN` 报错的核心原因是 Maven 对 XML 闭合标签的校验升级，以及旧版 Lombok 不兼容 JDK 21。请使用以下经过严格验证的 `pom.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.3</version>
        <relativePath/>
    </parent>

    <groupId>com.ailearn.governmentaffairsrag</groupId>
    <artifactId>spring-ai-policy-rag-system</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <java.version>21</java.version>
        <!-- 锁定兼容 JDK 21 的 Lombok 版本 -->
        <lombok.version>1.18.36</lombok.version> 
        <spring-ai.version>1.0.0-M6</spring-ai.version>
    </properties>

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

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-milvus-store-spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-tika-document-reader</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <!-- Spring AI M版依赖必需的里程碑仓库 -->
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots><enabled>false</enabled></snapshots>
        </repository>
    </repositories>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
*(注：替换后请清除本地 Maven 的 lombok 缓存，并执行 `mvn clean install -U` 强制更新)*

### 4.2 环境配置 (application.yml)
此配置确保向量数据库的维度设置与所选模型绝对一致。

```yaml
spring:
  application:
    name: policy-rag-system
  ai:
    ollama:
      base-url: http://<你的服务器公网IP>:11434 
      chat:
        model: qwen:7b   # 路线B 用户请改为 qwen2.5:1.5b
        options:
          temperature: 0.3 # 降低发散性，适合严谨问答
      embedding:
        model: bge-m3    
    vectorstore:
      milvus:
        client:
          host: <你的服务器公网IP>
          port: 19530
        collection-name: policy_docs
        embedding-dimension: 1024  # ⚠️ 必须与 bge-m3 匹配
        index-type: HNSW           # 启用高性能索引
```

### 4.3 核心业务逻辑
**1. 知识库摄入服务 (`IngestionService.java`)**
```java
@Service
@RequiredArgsConstructor
public class IngestionService {
    private final VectorStore vectorStore;

    public void processDocument(MultipartFile file) throws IOException {
        // Tika 万能文档解析
        TikaDocumentReader loader = new TikaDocumentReader(new InputStreamResource(file.getInputStream()));
        
        // Token 智能切片 (保留上下文重叠度)
        TextSplitter splitter = new TokenTextSplitter(500, 100, 10, 10000, true);
        List<Document> splitDocuments = splitter.apply(loader.get());
        
        // 注入元数据 (溯源标识)
        splitDocuments.forEach(doc -> doc.getMetadata().put("filename", file.getOriginalFilename()));
        
        // 向量化并持久化至 Milvus
        vectorStore.add(splitDocuments);
    }
}
```

**2. RAG 流式检索服务 (`RagService.java`)**
```java
@Service
@RequiredArgsConstructor
public class RagService {
    private final ChatClient.Builder chatClientBuilder;
    private final VectorStore vectorStore;

    public Flux<String> streamAnswer(String query) {
        ChatClient client = chatClientBuilder.build();

        // 1. 向量相似度检索 (Top 3，阈值 0.6)
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(3).withSimilarityThreshold(0.6)
        );
            
        // 2. 组装上下文与参考来源
        String context = docs.stream().map(Document::getContent).collect(Collectors.joining("\n\n"));
        String refs = docs.stream().map(d -> (String) d.getMetadata().get("filename"))
                          .distinct().collect(Collectors.joining(", "));

        // 3. 构建 Prompt
        String prompt = "基于以下政策上下文回答问题：\n" + context + "\n\n问题：" + query;

        // 4. 流式(SSE)输出并追加来源
        return client.prompt(prompt).stream().content()
                .concatWith(Flux.just("\n\n---\n📚 来源: " + refs));
    }
}
```

---

## 5. 前端交互实现 (Vue 3)

利用原生 `fetch` 结合 `ReadableStream` 接收后端的 `Flux` 数据流，实现打字机效果。

```javascript
const sendMessage = async () => {
  // 假设 query 为用户输入，assistantMsg 为响应对象
  try {
    const response = await fetch(`http://localhost:8080/api/chat?query=${encodeURIComponent(query)}`);
    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      // 流式解码并实时追加渲染到 UI
      const chunk = decoder.decode(value, { stream: true });
      assistantMsg.content += chunk; 
    }
  } catch (error) {
    console.error("流式响应中断:", error);
  }
};
```

---

## 6. 上线前核对清单

部署联调前，请执行最后一次 Checklist：

- [ ] **基础环境**：低配服务器已通过 `free -h` 确认 Swap 分区成功挂载。
- [ ] **网络连通**：本地电脑执行 `curl http://<公网IP>:11434/api/tags` 能成功获取模型 JSON。
- [ ] **依赖编译**：执行 `mvn clean compile` 无报错（已彻底解决 `TypeTag` 问题）。
- [ ] **维度匹配**：确认 `application.yml` 中 `embedding-dimension: 1024` 与 `bge-m3` 模型绑定无误。
- [ ] **内存限制**：(低配路线) 启动 Java 后端时添加了 `java -Xms1g -Xmx2g -jar app.jar` 参数。