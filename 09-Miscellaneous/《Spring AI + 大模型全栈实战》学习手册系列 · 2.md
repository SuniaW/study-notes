# 专题二：《Milvus 向量数据库：从零开始搭建 RAG 系统的核心组件》

## 📚 企业级 RAG 智能问答系统全栈实施指南

### 第二部分：向量数据库实战

---

## 第 1 章 Milvus 向量数据库概述

### 1.1 为什么选择 Milvus？

Milvus 是一款开源的向量数据库，专门用于处理由深度学习神经网络和其他机器学习模型生成的海量嵌入向量。

#### 向量数据库选型对比

| 数据库 | 类型 | 开源 | 中文支持 | 社区活跃度 | 适用场景 | 内存占用 |
|--------|------|------|----------|------------|----------|----------|
| **Milvus** | 专用向量库 | ✅ | ✅ | ⭐⭐⭐⭐⭐ | 生产环境首选 | 中等 |
| **Chroma** | 轻量向量库 | ✅ | ⚠️ | ⭐⭐⭐ | 快速原型 | 低 |
| **Pinecone** | 云服务 | ❌ | ✅ | ⭐⭐⭐⭐ | 云端部署 | N/A |
| **Weaviate** | 混合向量库 | ✅ | ⚠️ | ⭐⭐⭐⭐ | 知识图谱 | 高 |
| **Qdrant** | 专用向量库 | ✅ | ⚠️ | ⭐⭐⭐⭐ | 高精度检索 | 中等 |

#### Milvus 核心优势

| 优势 | 说明 | 技术实现 | 性能提升 |
|------|------|----------|----------|
| 🚀 云原生架构 | 存算分离，弹性伸缩 | Kubernetes 原生支持 | 资源利用率提升 40% |
| 📊 混合查询 | 向量+标量联合过滤 | 多索引协同 | 查询效率提升 3 倍 |
| 🔧 丰富索引 | 支持 10+ 索引类型 | HNSW/IVF/DiskANN 等 | 适配不同场景 |
| 🌐 多语言 SDK | Python/Java/Go/Node.js | 官方维护 | 开发效率提升 50% |
| 💰 开源免费 | Apache 2.0 协议 | 社区驱动 | 无授权费用 |

### 1.2 Milvus 2.6 核心升级（2025 最新版）

> 📢 **重大更新**：Milvus 2.6 版本于 2025 年 Q1 发布，带来多项性能突破

| 升级项 | 2.5 版本 | 2.6 版本 | 提升幅度 | 实际影响 |
|--------|----------|----------|----------|----------|
| 内存占用 | 基准 | **降低 72%** | 🔴 72% | 2G 内存可运行 |
| 检索速度 | 基准 | **提升 4 倍** | 🔴 400% | 响应延迟<50ms |
| 写入吞吐 | 基准 | **提升 2.5 倍** | 🟡 250% | 批量导入更快 |
| 索引构建 | 基准 | **提升 3 倍** | 🟡 300% | 冷启动时间缩短 |
| 并发连接 | 1000 | **5000+** | 🟡 500% | 支持更多用户 |

#### 2.6 版本新特性详解

```yaml
# 新增配置项
milvus:
  version: 2.6.0
  features:
    - 动态负载平衡：自动分配查询节点负载
    - 增量索引：支持在线索引更新，无需重建
    - 多租户隔离：逻辑隔离不同业务数据
    - 联邦学习：支持跨节点向量计算
    - GPU 加速：支持 NVIDIA CUDA 加速检索
```

---

## 第 2 章 Docker 部署实战

### 2.1 部署前环境检查

```bash
# 1. 检查 Docker 版本（要求 20.10+）
docker --version

# 2. 检查 Docker Compose 版本（要求 2.0+）
docker compose version

# 3. 检查系统内存（最低 4GB，推荐 8GB+）
free -h

# 4. 检查磁盘空间（至少 20GB 可用）
df -h

# 5. 关键：调整 Linux 内核参数（必须！）
sudo sysctl -w vm.max_map_count=262144

# 6. 永久生效（写入配置文件）
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

> ⚠️ **避坑警告**：`vm.max_map_count` 未设置会导致 Milvus 启动失败，报错 `map count exceeds limit`

### 2.2 精化版 docker-compose.yml（2 核 4G 优化版）

```yaml
version: '3.8'

services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/etcd:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - milvus
    deploy:
      resources:
        limits:
          memory: 512M

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-10Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    ports:
      - "9001:9001"
      - "9000:9000"
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data
    command: minio server /minio_data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
    networks:
      - milvus
    deploy:
      resources:
        limits:
          memory: 512M

  milvus-standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.6.0
    command: ["milvus", "run", "standalone"]
    security_opt:
      - seccomp:unconfined
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
      # 2.6 版本新增内存优化配置
      MILVUS_DB_MEMORY: 1024MB
      MILVUS_CACHE_SIZE: 256MB
      MILVUS_QUERY_NODE_MEMORY: 512MB
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus
    ports:
      - "19530:19530"
      - "9091:9091"
    depends_on:
      - "etcd"
      - "minio"
    networks:
      - milvus
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
      interval: 30s
      start_period: 90s
      timeout: 20s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 1024M

  attu:
    container_name: milvus-attu
    image: zilliz/attu:v2.3.0
    environment:
      MILVUS_URL: http://milvus-standalone:19530
    ports:
      - "3000:3000"
    depends_on:
      - "milvus-standalone"
    networks:
      - milvus
    deploy:
      resources:
        limits:
          memory: 256M

networks:
  milvus:
    driver: bridge
```

### 2.3 一键启动脚本

```bash
#!/bin/bash
# start-milvus.sh

echo "🚀 开始部署 Milvus 2.6..."

# 1. 创建数据目录
mkdir -p volumes/etcd volumes/minio volumes/milvus

# 2. 检查内核参数
MAP_COUNT=$(sysctl vm.max_map_count | awk '{print $3}')
if [ "$MAP_COUNT" -lt 262144 ]; then
    echo "⚠️  警告：vm.max_map_count 需要 >= 262144"
    echo "   执行：sudo sysctl -w vm.max_map_count=262144"
    sudo sysctl -w vm.max_map_count=262144
fi

# 3. 停止旧容器（如果有）
docker compose down

# 4. 清理旧数据（可选）
# docker compose down -v

# 5. 启动服务
docker compose up -d

# 6. 等待服务就绪（约 60-90 秒）
echo "⏳ 等待 Milvus 启动..."
sleep 60

# 7. 检查状态
docker compose ps

# 8. 查看日志
echo "📋 查看启动日志："
docker compose logs -f milvus-standalone

echo "✅ Milvus 部署完成！"
echo "   - 服务端口：19530"
echo "   - Web 管理：http://localhost:3000 (Attu)"
echo "   - MinIO 控制台：http://localhost:9001"
```

### 2.4 30 秒验证安装成功

```python
# verify_milvus.py
from pymilvus import connections, utility

# 连接 Milvus
connections.connect(host="localhost", port="19530")

# 检查连接状态
is_connected = connections.is_connected()
print(f"✅ 连接状态：{is_connected}")

# 检查服务器版本
version = utility.get_server_version()
print(f"📦 Milvus 版本：{version}")

# 列出所有集合
collections = utility.list_collections()
print(f"📚 现有集合：{collections}")

# 测试创建集合
from pymilvus import CollectionSchema, FieldSchema, DataType

fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=512)
]
schema = CollectionSchema(fields)
collection = utility.create_collection("test_collection", schema)

print("✅ 测试集合创建成功！")

# 清理测试集合
utility.drop_collection("test_collection")
print("✅ 验证完成，Milvus 运行正常！")
```

```bash
# 运行验证脚本
python verify_milvus.py
```

---

## 第 3 章 Milvus 架构详解

### 3.1 核心组件架构

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端层                              │
│    (Python SDK / Java SDK / REST API / Attu Web)            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                        接入层                                │
│                    Milvus Proxy                             │
│         (请求解析、负载均衡、权限验证)                        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                        协调层                                │
│    ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│    │   Root   │  │  Query   │  │  Data    │                │
│    │ Coordinator│ │ Coordinator│ │ Coordinator│            │
│    └──────────┘  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                        执行层                                │
│    ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│    │  Query   │  │  Data    │  │  Index   │                │
│    │  Node    │  │  Node    │  │  Node    │                │
│    └──────────┘  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                        存储层                                │
│    ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│    │  Etcd    │  │  MinIO   │  │  Pulsar  │                │
│    │(元数据)   │  │(对象存储) │  │(消息队列) │                │
│    └──────────┘  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 各组件职责说明

| 组件 | 职责 | 内存占用 | 是否可省略 | 说明 |
|------|------|----------|------------|------|
| **Proxy** | 请求解析、负载均衡、权限验证 | 256MB | ❌ 否 | 所有请求的入口 |
| **Root Coordinator** | 集群管理、DDL 操作 | 256MB | ❌ 否 | 负责集合创建/删除 |
| **Query Coordinator** | 查询任务调度 | 256MB | ❌ 否 | 分配查询请求 |
| **Data Coordinator** | 数据管理、Compaction | 256MB | ❌ 否 | 负责数据分段 |
| **Query Node** | 执行向量检索 | 512MB+ | ❌ 否 | 加载索引到内存 |
| **Data Node** | 数据写入、Flush | 256MB | ❌ 否 | 处理插入请求 |
| **Index Node** | 索引构建 | 512MB+ | ⚠️ 可选 | 可复用 Data Node |
| **Etcd** | 元数据存储 | 256MB | ❌ 否 | 存储配置和状态 |
| **MinIO** | 对象存储 | 256MB | ❌ 否 | 存储向量数据文件 |
| **Pulsar/Kafka** | 消息队列 | 512MB | ⚠️ 可选 | 单机版可省略 |

### 3.3 存算分离架构优势

```
传统架构：
┌─────────────────┐
│   计算 + 存储    │  ← 耦合，扩展困难
│   同一节点      │
└─────────────────┘

Milvus 存算分离：
┌─────────────┐    ┌─────────────┐
│  计算节点    │    │  存储节点    │
│  (Query/Data)│ ←→ │  (MinIO)    │
│  可独立扩展  │    │  可独立扩展  │
└─────────────┘    └─────────────┘
```

| 特性 | 传统架构 | 存算分离 | 优势 |
|------|----------|----------|------|
| 扩展性 | 垂直扩展 | 水平扩展 | 成本降低 60% |
| 资源利用 | 计算存储绑定 | 独立调度 | 利用率提升 40% |
| 故障恢复 | 数据迁移慢 | 快速重建 | RTO 降低 80% |
| 成本 | 高配服务器 | 低配计算 + 廉价存储 | TCO 降低 50% |

---

## 第 4 章 集合（Collection）操作实战

### 4.1 创建集合（Python SDK）

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType

# 连接 Milvus
connections.connect(host="localhost", port="19530")

# 定义字段
fields = [
    # 主键字段
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    
    # 向量字段（核心）
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=512),
    
    # 元数据字段（用于过滤）
    FieldSchema(name="doc_id", dtype=DataType.VARCHAR, max_length=64),
    FieldSchema(name="content", dtype=DataType.VARCHAR, max_length=65535),
    FieldSchema(name="create_time", dtype=DataType.INT64)
]

# 创建 Schema
schema = CollectionSchema(
    fields=fields,
    description="RAG 文档向量集合",
    enable_dynamic_field=True  # 允许动态添加字段
)

# 创建集合
collection = Collection(name="rag_documents", schema=schema)

print(f"✅ 集合创建成功：{collection.name}")
print(f"📊 字段数量：{len(schema.fields)}")
print(f"📏 向量维度：{512}")
```

### 4.2 创建索引（关键步骤）

```python
from pymilvus import Index

# 索引参数配置
index_params = {
    "metric_type": "COSINE",      # 余弦相似度
    "index_type": "HNSW",         # 索引类型
    "params": {
        "M": 16,                  # 每个节点的最大连接数
        "efConstruction": 128     # 构建索引时的搜索深度
    }
}

# 创建索引
collection.create_index(
    field_name="embedding",
    index_params=index_params,
    index_name="embedding_index"
)

# 加载集合到内存
collection.load()

print("✅ 索引创建成功！")
print("✅ 集合已加载到内存！")
```

### 4.3 插入向量数据

```python
import random

# 准备测试数据
num_vectors = 1000
vectors = [[random.random() for _ in range(512)] for _ in range(num_vectors)]
doc_ids = [f"doc_{i}" for i in range(num_vectors)]
contents = [f"文档内容片段 {i}" for i in range(num_vectors)]
create_times = [1704067200 + i for i in range(num_vectors)]

# 插入数据
entities = [
    {"name": "embedding", "values": vectors},
    {"name": "doc_id", "values": doc_ids},
    {"name": "content", "values": contents},
    {"name": "create_time", "values": create_times}
]

# 执行插入
insert_result = collection.insert(entities)

# 刷新索引
collection.flush()

print(f"✅ 插入 {insert_result.insert_count} 条记录")
print(f"📊 集合总数：{collection.num_entities}")
```

### 4.4 向量检索查询

```python
# 准备查询向量
query_vector = [random.random() for _ in range(512)]

# 搜索参数
search_params = {
    "metric_type": "COSINE",
    "params": {"ef": 64}  # 搜索深度
}

# 执行搜索
results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param=search_params,
    limit=5,  # 返回 Top 5
    output_fields=["doc_id", "content", "create_time"]
)

# 解析结果
for hits in results:
    for hit in hits:
        print(f"📄 文档 ID: {hit.entity.get('doc_id')}")
        print(f"📝 内容：{hit.entity.get('content')}")
        print(f"📊 相似度：{hit.score}")
        print(f"⏰ 创建时间：{hit.entity.get('create_time')}")
        print("-" * 50)
```

### 4.5 混合检索（向量 + 标量过滤）

```python
# 向量检索 + 时间范围过滤
results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param=search_params,
    limit=5,
    expr="create_time > 1704067200 and create_time < 1704153600",  # 标量过滤
    output_fields=["doc_id", "content"]
)

print("✅ 混合检索完成（向量相似度 + 时间过滤）")
```

---

## 第 5 章 性能优化与监控

### 5.1 内存优化配置

```yaml
# milvus.yaml 关键配置项
queryNode:
  loadMemoryUsageFactor: 0.9    # 内存使用因子（默认 0.9）
  overloadMemoryThreshold: 0.8  # 过载阈值
  
dataNode:
  flushInsertBufferSize: 16777216  # 16MB 触发 flush
  
indexNode:
  buildIndexMemoryLimit: 0.5    # 索引构建内存限制 50%
```

### 5.2 索引类型选择指南

| 索引类型 | 适用场景 | 内存占用 | 检索速度 | 构建速度 | 推荐度 |
|----------|----------|----------|----------|----------|--------|
| **HNSW** | 高精度、内存充足 | 高 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **IVF_FLAT** | 平衡方案 | 中 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **IVF_PQ** | 海量数据、内存受限 | 低 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **FLAT** | 小数据集 (<10 万) | 最低 | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **DISKANN** | 超大数据集 | 低 | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

### 5.3 监控指标与告警

```python
# 监控脚本：check_milvus_health.py
import requests
import json

def check_milvus_health():
    """检查 Milvus 健康状态"""
    
    # 1. 健康检查接口
    health_url = "http://localhost:9091/healthz"
    response = requests.get(health_url)
    
    if response.status_code == 200:
        print("✅ Milvus 服务健康")
    else:
        print("❌ Milvus 服务异常")
        return
    
    # 2. 获取系统指标
    metrics_url = "http://localhost:9091/metrics"
    metrics = requests.get(metrics_url).text
    
    # 3. 解析关键指标
    for line in metrics.split('\n'):
        if 'milvus_query_node_memory_usage' in line:
            print(f"📊 Query Node 内存：{line}")
        if 'milvus_data_node_insert_count' in line:
            print(f"📈 插入数量：{line}")
    
    # 4. 集合统计
    from pymilvus import utility
    collections = utility.list_collections()
    for col in collections:
        collection = utility.get_collection_stats(col)
        print(f"📚 集合 {col}: {collection['row_count']} 条记录")

check_milvus_health()
```

### 5.4 常见问题排查清单

| 问题 | 症状 | 原因 | 解决方案 | 优先级 |
|------|------|------|----------|--------|
| 启动失败 | 容器反复重启 | vm.max_map_count 未设置 | `sudo sysctl -w vm.max_map_count=262144` | 🔴 高 |
| 内存溢出 | OOM Killed | 内存限制过低 | 调整 docker-compose 内存限制 | 🔴 高 |
| 连接超时 | 19530 端口无法连接 | 防火墙/安全组 | 放行 19530/9091 端口 | 🟡 中 |
| 检索慢 | 响应>1s | 索引未加载 | `collection.load()` | 🟡 中 |
| 数据丢失 | 重启后数据消失 | 未持久化 | 检查 volumes 挂载 | 🟡 中 |
| 索引构建失败 | 报错 memory limit | 索引节点内存不足 | 增加 indexNode 内存 | 🟢 低 |

---

## 第 6 章 Milvus Lite：30 秒极速上手

### 6.1 什么是 Milvus Lite？

> 🎯 **轻量级版本**：无需 Docker，Python 直接运行，适合本地开发和测试

### 6.2 安装与使用

```bash
# 安装 Milvus Lite
pip install milvus-lite

# 验证安装
python -c "from milvus_lite import MilvusLite; print('✅ 安装成功')"
```

```python
# 完整示例：milvus_lite_demo.py
from milvus_lite import MilvusLite
from pymilvus import FieldSchema, CollectionSchema, DataType

# 1. 创建本地 Milvus 实例（自动下载二进制）
ml = MilvusLite("milvus_demo.db")

# 2. 连接
from pymilvus import connections
connections.connect(host="127.0.0.1", port=ml.port)

# 3. 创建集合
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=512),
    FieldSchema(name="text", dtype=DataType.VARCHAR, max_length=65535)
]
schema = CollectionSchema(fields)
collection = Collection("demo_collection", schema)

# 4. 创建索引
collection.create_index("embedding", {
    "metric_type": "COSINE",
    "index_type": "HNSW",
    "params": {"M": 16, "efConstruction": 128}
})
collection.load()

# 5. 插入数据
import random
vectors = [[random.random() for _ in range(512)] for _ in range(100)]
texts = [f"测试文本 {i}" for i in range(100)]
collection.insert([vectors, texts])
collection.flush()

# 6. 检索
query_vector = [random.random() for _ in range(512)]
results = collection.search(
    data=[query_vector],
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"ef": 64}},
    limit=5,
    output_fields=["text"]
)

print("✅ Milvus Lite 演示完成！")
print(f"📊 检索结果：{len(results[0])} 条")

# 7. 清理
ml.stop()
```

### 6.3 Milvus Lite vs 完整 Milvus 对比

| 特性 | Milvus Lite | 完整 Milvus | 适用场景 |
|------|-------------|-------------|----------|
| 部署方式 | Python 包 | Docker/K8s | Lite: 本地开发 |
| 启动时间 | <30 秒 | 60-90 秒 | Lite: 快速测试 |
| 内存占用 | ~500MB | ~2GB+ | Lite: 低配环境 |
| 并发能力 | 单用户 | 多用户 | Lite: 个人使用 |
| 持久化 | 本地文件 | 分布式存储 | Lite: 测试数据 |
| 生产就绪 | ❌ 否 | ✅ 是 | Lite: 仅限开发 |

---

## 第 7 章 Spring AI 集成 Milvus

### 7.1 依赖配置

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring AI Milvus 集成 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-milvus-store</artifactId>
        <version>1.0.0-M6</version>
    </dependency>
    
    <!-- Milvus Java SDK -->
    <dependency>
        <groupId>io.milvus</groupId>
        <artifactId>milvus-sdk-java</artifactId>
        <version>2.4.0</version>
    </dependency>
</dependencies>
```

### 7.2 应用配置

```yaml
# application.yml
spring:
  ai:
    vectorstore:
      milvus:
        host: localhost
        port: 19530
        username: 
        password: 
        database: default
        collection-name: rag_documents
        index-type: HNSW
        metric-type: COSINE
        embedding-dimension: 512
        initialize-schema: true
```

### 7.3 向量存储服务

```java
// MilvusVectorStoreService.java
@Service
public class MilvusVectorStoreService {
    
    @Autowired
    private VectorStore vectorStore;
    
    /**
     * 添加文档到向量库
     */
    public void addDocument(String text, Map<String, Object> metadata) {
        Document document = new Document(text, metadata);
        vectorStore.add(List.of(document));
        log.info("✅ 文档已向量化并存储");
    }
    
    /**
     * 相似性检索
     */
    public List<Document> similaritySearch(String query, int topK) {
        return vectorStore.similaritySearch(query, topK);
    }
    
    /**
     * 删除文档
     */
    public void deleteDocument(String id) {
        vectorStore.delete(List.of(id));
        log.info("✅ 文档已删除：{}", id);
    }
}
```

---

**本专题完**

> 📌 **核心要点总结**：
> 1. Milvus 2.6 内存占用降低 72%，2G 内存可运行
> 2. `vm.max_map_count=262144` 是启动成功的必要条件
> 3. HNSW 索引是生产环境推荐选择（M=16, efConstruction=128）
> 4. 存算分离架构支持独立扩展，降低 50% TCO
> 5. Milvus Lite 适合本地开发，30 秒极速上手
> 6. Spring AI 提供原生 Milvus 集成，简化开发

> 📖 **下专题预告**：《Embedding 模型选型指南：从 MMTEB 排名到实际应用》—— 详解模型选型决策树、中文场景优化、文档分块策略进阶