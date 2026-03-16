

# 🚀 阿里云 2核4G 服务器：私有化 RAG 全链路调优指南

**背景：** 在 2核4G 的资源受限环境下部署 RAG（检索增强生成）系统，必须走“资源精细化管理”路线。本文涵盖了从内核限额、端口冲突解决、代码实现到可视化管理的完整链路。

---

## 一、 核心生存法则：资源资产负债表

在 4G 内存中运行 **Milvus + Ollama + Spring Boot**，必须执行以下分配方案，严防 OOM（内存溢出）：

| 组件 | 内存分配上限 | 调优核心手段 |
| :--- | :--- | :--- |
| **Milvus 数据库** | **1.0 GB** | Docker `deploy.resources.limits` 硬限制 |
| **Ollama (LLM+Embed)** | **2.2 GB** | 设置 `OLLAMA_KEEP_ALIVE=5m` 动态释放 |
| **Spring Boot (JVM)** | **0.6 GB** | 启动参数 `-Xmx600m` 限制堆内存 |
| **OS 预留** | **0.4 GB** | 保持系统基础响应 |

---

## 二、 数据库层：Milvus 高可靠部署 (Docker Compose)

### 1. 解决端口冲突 (卸载 Portainer)
如果遇到 `9000 failed: port is already allocated`，通常是 Portainer 占用了端口。建议**卸载 Portainer** 以节省约 100MB 内存：
```bash
docker stop portainer && docker rm portainer
docker volume rm portainer_data
```

### 2. 精化版 `docker-compose.yml`
通过配置 **健康检查依赖** 和 **缓存限制**，确保系统在 2核 CPU 下平稳启动：
```yaml
services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    deploy:
      resources:
        limits: { memory: 256M }

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    ports: ["9000:9000", "9001:9001"]
    command: minio server /minio_data
    deploy:
      resources:
        limits: { memory: 512M }

  standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.4.0
    environment:
      CACHE_SIZE: 512M # 核心优化：限制内部索引缓存
    ports: ["19530:19530"]
    depends_on:
      etcd: { condition: service_healthy }
      minio: { condition: service_healthy }
    deploy:
      resources:
        limits: { memory: 1024M } # 严控 1G 线
```

---

## 三、 模型层：Ollama 推理优化

### 1. 配置动态释放
修改 Ollama 服务，防止模型长期霸占内存：
```bash
sudo systemctl edit ollama.service
# 添加以下两行
Environment="OLLAMA_KEEP_ALIVE=5m"   # 5分钟不使用自动释放
Environment="OLLAMA_NUM_PARALLEL=1"   # 串行处理防止瞬时崩溃
```

### 2. 预热加速技巧
启动 Java 应用前，手动预热 BGE-M3 模型（1024维），避免首次搜索超时：
```bash
curl http://127.0.0.1:11434/api/embeddings -d '{"model": "bge-m3", "prompt": "warmup"}'
```

---

## 四、 应用层：Spring AI 实现与避坑

### 1. `application.yml` 配置对齐 (关键)
必须使用 `127.0.0.1` 以绕过阿里云公网防火墙，并统一维度：
```yaml
spring:
  ai:
    ollama:
      base-url: http://127.0.0.1:11434
    vectorstore:
      milvus:
        client:
          host: 127.0.0.1
          port: 19530
        collection-name: policy_docs # 名字需与数据库中一致
        embedding-dimension: 1024   # BGE-M3 对应 1024 维
```

### 2. 解决 `Collection Not Found` 错误
**痛点：** Milvus 不会自动在 `Search` 时创建集合。
**对策：** 在执行查询前，必须确保系统已经执行过一次 `vectorStore.add(documents)`。
*   建议逻辑：程序启动后或第一次上传文件时调用 `add` 触发集合自动创建。

### 3. Java 服务启动
```bash
java -Xms512m -Xmx600m -jar policy-rag-system.jar
```

---

## 五、 维护与监控

### 1. 可视化管理 (Attu)
*   **工具选择：** 强烈建议使用桌面客户端（Zilliz Cloud Desktop 或 Attu 桌面版），从本地电脑连接阿里云 IP。**严禁在服务器上额外部署可视化镜像**以防内存爆表。
*   **登录：** Milvus 默认无密码（或尝试 `root / Milvus`）。
*   **检查项：** 通过可视化工具确认 `policy_docs` 集合已被 `Load` 进内存，否则无法搜索。

### 2. 故障排查排次表 (TSH)
1.  **连不上数据库？**
    *   执行 `netstat -tunlp | grep 19530` 确认端口开启。
    *   确认应用配置文件使用的是 `127.0.0.1` 而非公网 IP。
2.  **Search failed？**
    *   确认集合维度 (1024) 是否与模型一致。
    *   检查库里是否有数据（entities > 0）。
3.  **系统卡顿？**
    *   执行 `docker stats` 检查哪个容器撞到了 `Limit` 墙。

---

## 💡 总结

通过 **“内网互通 + 动态显压 + 维度对齐”** 三位一体的调优，你的阿里云 4G 服务器已变身为生产级的 RAG 节点。后续扩展建议将 Embedding/LLM 逐步迁移至**阿里云 DashScope (API)**，将本地服务器转型为纯粹的向量存储计算节点，可获得更好的稳定性！