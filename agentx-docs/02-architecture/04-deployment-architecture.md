# 部署架构

本章详细说明 AgentX 平台在不同环境下的部署架构，包括开发环境、测试环境和生产环境，以及针对低配服务器的优化策略。

## 部署环境概览

AgentX 支持多种部署模式，适应不同规模和需求：

| 环境类型 | 目标 | 部署模式 | 资源要求 | 特点 |
|---------|------|---------|---------|------|
| **开发环境** | 本地开发和调试 | Docker Compose | 2C4G | 快速启动，便于调试 |
| **测试环境** | 功能验证和集成测试 | 单机/小型集群 | 4C8G | 接近生产配置，数据隔离 |
| **预生产环境** | 上线前最终验证 | 高可用集群 | 8C16G | 与生产环境完全一致 |
| **生产环境** | 稳定可靠服务 | 高可用集群 | 按需扩展 | 高可用，自动扩缩容 |

## 开发环境部署

### 1. Docker Compose 部署（推荐）

#### 架构设计
```
┌─────────────────────────────────────────────────────┐
│                开发机器（本地）                     │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ AgentX应用  │  │   Redis     │  │   Milvus    │  │
│  │   :8080     │  │   :6379     │  │   :19530    │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │   MinIO     │  │    etcd     │  │  监控工具   │  │
│  │   :9000     │  │   :2379     │  │  :9090      │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────┘
```

#### docker-compose.yml 配置
```yaml
version: '3.8'

services:
  # AgentX 应用
  agentx-app:
    build: .
    image: agentx:latest
    container_name: agentx-app
    ports:
      - "8080:8080"
      - "8081:8081"  # 管理端口
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_HOST=redis
      - MILVUS_HOST=milvus
      - JAVA_OPTS=-Xmx1g -Xms512m
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    depends_on:
      - redis
      - milvus
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis
  redis:
    image: redis:7.4-alpine
    container_name: agentx-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --maxmemory 512mb
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s

  # Milvus
  milvus:
    image: milvusdb/milvus:v2.3.3
    container_name: agentx-milvus
    ports:
      - "19530:19530"
    environment:
      - ETCD_ENDPOINTS=etcd:2379
      - MINIO_ADDRESS=minio:9000
    depends_on:
      - etcd
      - minio
    volumes:
      - milvus-data:/var/lib/milvus

  # 依赖服务
  etcd:
    image: quay.io/coreos/etcd:v3.5.10
    container_name: etcd
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
    volumes:
      - etcd-data:/etcd

  minio:
    image: minio/minio:RELEASE.2023-12-20T01-00-02Z
    container_name: minio
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - minio-data:/data

  # 监控（可选）
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

volumes:
  redis-data:
  milvus-data:
  etcd-data:
  minio-data:
  prometheus-data:
```

#### 启动脚本
```bash
#!/bin/bash
# start-dev.sh

echo "🚀 启动 AgentX 开发环境..."

# 检查 Docker 和 Docker Compose
if ! command -v docker &> /dev/null; then
    echo "❌ Docker 未安装，请先安装 Docker"
    exit 1
fi

if ! command -v docker-compose &> /dev/null; then
    echo "❌ Docker Compose 未安装，请先安装 Docker Compose"
    exit 1
fi

# 检查环境变量
if [ ! -f .env ]; then
    echo "⚠️  未找到 .env 文件，正在创建模板..."
    cp .env.example .env
    echo "请编辑 .env 文件设置 OpenAI API 密钥"
    exit 1
fi

# 启动服务
echo "📦 启动服务..."
docker-compose up -d

# 等待服务就绪
echo "⏳ 等待服务就绪..."
sleep 10

# 检查服务状态
echo "🔍 检查服务状态..."
docker-compose ps

echo ""
echo "✅ AgentX 开发环境启动完成！"
echo ""
echo "📋 访问地址："
echo "   - AgentX 应用：http://localhost:8080"
echo "   - Swagger UI：http://localhost:8080/swagger-ui.html"
echo "   - 健康检查：http://localhost:8080/actuator/health"
echo "   - Prometheus：http://localhost:9090"
echo ""
echo "📝 常用命令："
echo "   - 查看日志：docker-compose logs -f"
echo "   - 停止服务：docker-compose down"
echo "   - 重新构建：docker-compose build --no-cache"
```

### 2. 本地源码部署

#### 环境要求
```bash
# 安装 Java 和 Maven
# Java 21
sudo apt install openjdk-21-jdk

# Maven
sudo apt install maven

# 验证安装
java -version  # 应该显示 Java 21
mvn -version   # 应该显示 Maven 3.8+
```

#### 启动步骤
```bash
# 1. 克隆项目
git clone https://github.com/your-org/agentx.git
cd agentx

# 2. 启动依赖服务
docker-compose up -d redis milvus

# 3. 编译项目
mvn clean package -DskipTests

# 4. 运行应用
java -jar target/agentx-*.jar

# 或使用 Maven 插件
mvn spring-boot:run
```

## 测试环境部署

### 架构设计
```
┌─────────────────────────────────────────────────────────┐
│                  测试服务器                             │
│                                                         │
│  ┌─────────────┐      ┌─────────────┐                  │
│  │  负载均衡器 │─────▶│ AgentX实例1 │                  │
│  │   :80       │      │   :8080     │                  │
│  └─────────────┘      └─────────────┘                  │
│         │                                              │
│         │          ┌─────────────┐                    │
│         └─────────▶│ AgentX实例2 │                    │
│                    │   :8081     │                    │
│                    └─────────────┘                    │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Redis     │  │   Milvus    │  │ PostgreSQL  │    │
│  │   集群      │  │   集群      │  │             │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Kubernetes 部署（推荐）

#### 命名空间配置
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: agentx-test
  labels:
    name: agentx-test
    environment: test
```

#### 配置映射
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agentx-config
  namespace: agentx-test
data:
  application.yml: |
    spring:
      profiles:
        active: test
      
      ai:
        openai:
          api-key: ${OPENAI_API_KEY}
          chat:
            options:
              model: gpt-3.5-turbo
      
      data:
        redis:
          host: redis-master.agentx-test.svc.cluster.local
          port: 6379
        
        milvus:
          host: milvus.agentx-test.svc.cluster.local
          port: 19530
    
    server:
      port: 8080
    
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
```

#### 部署配置
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agentx-deployment
  namespace: agentx-test
  labels:
    app: agentx
    environment: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: agentx
  template:
    metadata:
      labels:
        app: agentx
        environment: test
    spec:
      containers:
      - name: agentx
        image: agentx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8081
          name: management
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "test"
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: agentx-secrets
              key: openai-api-key
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8081
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 5
```

#### 服务配置
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: agentx-service
  namespace: agentx-test
spec:
  selector:
    app: agentx
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: management
    port: 8081
    targetPort: 8081
  type: ClusterIP
```

#### Ingress 配置
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: agentx-ingress
  namespace: agentx-test
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: agentx-test.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: agentx-service
            port:
              number: 80
```

### 自动化测试部署

#### CI/CD 流水线
```yaml
# .gitlab-ci.yml 示例
stages:
  - build
  - test
  - deploy-test
  - deploy-prod

variables:
  DOCKER_IMAGE: registry.example.com/agentx:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  script:
    - mvn clean package -DskipTests
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE

test:
  stage: test
  script:
    - mvn test
    - ./run-integration-tests.sh

deploy-test:
  stage: deploy-test
  script:
    - kubectl config use-context test-cluster
    - kubectl set image deployment/agentx-deployment agentx=$DOCKER_IMAGE -n agentx-test
    - kubectl rollout status deployment/agentx-deployment -n agentx-test
  only:
    - develop

deploy-prod:
  stage: deploy-prod
  script:
    - kubectl config use-context prod-cluster
    - kubectl set image deployment/agentx-deployment agentx=$DOCKER_IMAGE -n agentx-prod
    - kubectl rollout status deployment/agentx-deployment -n agentx-prod
  only:
    - main
  when: manual
```

## 生产环境部署

### 高可用架构

#### 多可用区部署
```
┌─────────────────────────────────────────────────────────────────────┐
│                          生产环境                                   │
│                                                                     │
│       可用区 A                           可用区 B                   │
│  ┌─────────────────┐            ┌─────────────────┐                │
│  │  负载均衡器     │            │  负载均衡器     │                │
│  │  ELB/NLB        │            │  ELB/NLB        │                │
│  └────────┬────────┘            └────────┬────────┘                │
│           │                               │                         │
│  ┌────────▼────────┐            ┌────────▼────────┐                │
│  │  AgentX 集群    │            │  AgentX 集群    │                │
│  │  (3+ 节点)      │◀──────────▶│  (3+ 节点)      │                │
│  └────────┬────────┘  同步      └────────┬────────┘                │
│           │                               │                         │
│  ┌────────▼────────┐            ┌────────▼────────┐                │
│  │  数据存储层     │            │  数据存储层     │                │
│  │  Redis集群      │◀──────────▶│  Redis集群      │                │
│  │  Milvus集群     │  跨区同步  │  Milvus集群     │                │
│  │  PostgreSQL集群 │            │  PostgreSQL集群 │                │
│  └─────────────────┘            └─────────────────┘                │
│                                                                     │
│        ┌─────────────────────────────────────────────────────┐     │
│        │                   全局负载均衡                      │     │
│        │                 (GeoDNS/CDN)                        │     │
│        └──────────────────────────┬──────────────────────────┘     │
│                                   │                                │
│                        ┌──────────▼──────────┐                    │
│                        │     终端用户        │                    │
│                        │                     │                    │
│                        └─────────────────────┘                    │
└─────────────────────────────────────────────────────────────────────┘
```

#### Kubernetes 生产配置

**资源配额**
```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: agentx-quota
  namespace: agentx-prod
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    services: "20"
```

**Pod 安全策略**
```yaml
# psp.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: agentx-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'secret'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
```

**网络策略**
```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agentx-network-policy
  namespace: agentx-prod
spec:
  podSelector:
    matchLabels:
      app: agentx
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - podSelector:
        matchLabels:
          app: milvus
    ports:
    - protocol: TCP
      port: 19530
  - to:
    - podSelector:
        matchLabels:
          app: postgresql
    ports:
    - protocol: TCP
      port: 5432
```

### 自动扩缩容

#### Horizontal Pod Autoscaler
```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agentx-hpa
  namespace: agentx-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agentx-deployment
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 100
```

#### 自定义指标扩缩容
```yaml
# custom-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agentx-custom-hpa
  namespace: agentx-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agentx-deployment
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Object
    object:
      metric:
        name: agent_requests_per_second
      describedObject:
        apiVersion: v1
        kind: Service
        name: agentx-service
      target:
        type: Value
        value: 500
  - type: Object
    object:
      metric:
        name: average_response_time
      describedObject:
        apiVersion: v1
        kind: Service
        name: agentx-service
      target:
        type: Value
        value: 2000
```

### 监控和告警

#### Prometheus 监控配置
```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: agentx-alerts
  namespace: agentx-prod
spec:
  groups:
  - name: agentx.rules
    rules:
    - alert: HighErrorRate
      expr: rate(agentx_request_errors_total[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "高错误率"
        description: "AgentX 请求错误率超过 5%"
    
    - alert: HighLatency
      expr: histogram_quantile(0.95, rate(agentx_request_duration_seconds_bucket[5m])) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "高延迟"
        description: "AgentX 95% 请求延迟超过 3 秒"
    
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total{container="agentx"}[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod 频繁重启"
        description: "AgentX Pod 在 15 分钟内重启超过 0 次"
```

#### Grafana 仪表板
```json
{
  "dashboard": {
    "title": "AgentX 生产监控",
    "panels": [
      {
        "title": "请求率",
        "targets": [
          {
            "expr": "rate(agentx_request_total[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ]
      },
      {
        "title": "响应时间",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(agentx_request_duration_seconds_bucket[5m]))",
            "legendFormat": "p95"
          }
        ]
      },
      {
        "title": "错误率",
        "targets": [
          {
            "expr": "rate(agentx_request_errors_total[5m]) / rate(agentx_request_total[5m])",
            "legendFormat": "错误率"
          }
        ]
      }
    ]
  }
}
```

## 低配服务器优化

### 2C4G 服务器部署策略

#### 资源分配策略
```
2C4G 服务器资源分配：
┌─────────────────────────────────────────┐
│ 总资源：2 CPU 核心，4GB 内存            │
│                                         │
│ 应用分配：                              │
│   • AgentX 应用：1 CPU，2GB 内存       │
│   • Redis：0.5 CPU，1GB 内存           │
│   • Milvus：0.5 CPU，1GB 内存          │
│                                         │
│ 内存分配细节：                          │
│   • JVM 堆内存：1.5GB                  │
│   • 堆外内存：256MB                    │
│   • 系统预留：256MB                    │
└─────────────────────────────────────────┘
```

#### Docker Compose 优化配置
```yaml
services:
  agentx-app:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
    environment:
      - JAVA_OPTS=-Xmx1536m -Xms768m -XX:MaxDirectMemorySize=256m
      - SPRING_PROFILES_ACTIVE=low-resource
  
  redis:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 512M
    command: redis-server --maxmemory 768mb --maxmemory-policy allkeys-lru
  
  milvus:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 512M
```

#### JVM 调优参数
```bash
# 低配服务器 JVM 参数
JAVA_OPTS="
  -Xmx1536m                # 最大堆内存 1.5GB
  -Xms768m                 # 初始堆内存 768MB
  -XX:MaxDirectMemorySize=256m  # 最大直接内存 256MB
  -XX:+UseG1GC             # 使用 G1 垃圾回收器
  -XX:MaxGCPauseMillis=200 # 最大 GC 暂停时间 200ms
  -XX:ParallelGCThreads=2  # 并行 GC 线程数
  -XX:ConcGCThreads=1      # 并发 GC 线程数
  -XX:InitiatingHeapOccupancyPercent=45  # 触发混合 GC 的堆占用率
  -XX:+UseStringDeduplication  # 字符串去重
  -XX:+OptimizeStringConcat    # 字符串连接优化
"
```

### 性能优化策略

#### 内存优化
1. **JVM 调优**：精细调整堆内存和垃圾回收参数
2. **缓存优化**：减少 Redis 内存使用，使用 LRU 淘汰策略
3. **连接池优化**：合理设置连接池大小，避免内存泄漏
4. **内存监控**：实时监控内存使用，及时告警

#### CPU 优化
1. **线程池优化**：合理设置线程池大小，避免上下文切换
2. **批处理优化**：合并小请求，减少处理开销
3. **异步处理**：耗时操作异步执行，不阻塞主线程
4. **CPU 亲和性**：关键进程绑定 CPU 核心

#### 磁盘优化
1. **日志轮转**：自动轮转日志文件，避免磁盘写满
2. **数据压缩**：存储数据时进行压缩
3. **磁盘缓存**：合理使用操作系统磁盘缓存
4. **监控告警**：磁盘使用率监控和告警

### 监控和告警

#### 资源监控指标
```prometheus
# CPU 使用率
process_cpu_usage{application="agentx"} 0.45

# 内存使用
jvm_memory_used_bytes{area="heap"} 1.2e9
jvm_memory_max_bytes{area="heap"} 1.5e9

# GC 情况
jvm_gc_pause_seconds_count{gc="G1 Young Generation"} 120
jvm_gc_pause_seconds_sum{gc="G1 Young Generation"} 12.5

# 线程情况
jvm_threads_live_threads 45
jvm_threads_peak_threads 60
```

#### 告警规则
```yaml
# 低配服务器专属告警
- alert: HighMemoryUsage
  expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.8
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "高内存使用率"
    description: "JVM 堆内存使用率超过 80%"

- alert: FrequentGC
  expr: rate(jvm_gc_pause_seconds_count[5m]) > 10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "频繁垃圾回收"
    description: "5分钟内 GC 次数超过 10 次"
```

## 多云和混合云部署

### 跨云部署架构
```
┌─────────────────────────────────────────────────────────────────────┐
│                        多云部署架构                                 │
│                                                                     │
│       公有云 A (AWS)                   公有云 B (Azure)             │
│  ┌─────────────────┐            ┌─────────────────┐                │
│  │  AgentX 集群    │            │  AgentX 集群    │                │
│  │  EKS/AKS        │◀──────────▶│  AKS/EKS        │                │
│  └────────┬────────┘  全球DNS   └────────┬────────┘                │
│           │             负载均衡           │                         │
│  ┌────────▼────────┐            ┌────────▼────────┐                │
│  │  云原生存储     │            │  云原生存储     │                │
│  │  ElastiCache    │◀──────────▶│  Azure Cache    │                │
│  │  RDS            │  数据同步  │  Azure SQL      │                │
│  └─────────────────┘            └─────────────────┘                │
│                                                                     │
│                    ┌─────────────────────┐                         │
│                    │    私有云/本地       │                         │
│                    │  ┌─────────────┐    │                         │
│                    │  │  AgentX     │    │                         │
│                    │  │  (边缘部署) │    │                         │
│                    │  └─────────────┘    │                         │
│                    └─────────────────────┘                         │
│                                                                     │
│        ┌─────────────────────────────────────────────────────┐     │
│        │               全局流量管理                          │     │
│        │          (Cloudflare/Akamai)                        │     │
│        └──────────────────────────┬──────────────────────────┘     │
│                                   │                                │
│                        ┌──────────▼──────────┐                    │
│                        │     全球用户        │                    │
│                        │                     │                    │
│                        └─────────────────────┘                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 部署工具选择

| 部署工具 | 适用场景 | 优点 | 缺点 |
|---------|---------|------|------|
| **Kubernetes** | 生产环境，多云部署 | 标准化，生态丰富，多云支持 | 学习曲线陡峭，运维复杂 |
| **Docker Compose** | 开发测试，单机部署 | 简单易用，快速启动 | 扩展性有限，不适合生产 |
| **Terraform** | 基础设施即代码 | 多云支持，版本控制，自动化 | 需要学习新语言，调试复杂 |
| **Ansible** | 配置管理，传统部署 | 简单易学，代理无代理 | 大规模部署性能可能受影响 |

## 部署检查清单

### 预部署检查
- [ ] 环境变量配置正确
- [ ] 数据库连接配置正确
- [ ] 外部服务 API 密钥配置
- [ ] 证书和密钥安全存储
- [ ] 监控和告警配置完成

### 部署过程
- [ ] 备份现有数据和配置
- [ ] 执行数据库迁移（如需要）
- [ ] 部署新版本应用
- [ ] 验证服务健康状态
- [ ] 执行冒烟测试
- [ ] 监控关键指标

### 部署后验证
- [ ] 所有服务正常运行
- [ ] API 接口可用性测试
- [ ] 性能基准测试
- [ ] 监控数据正常收集
- [ ] 告警规则测试

## 故障恢复

### 回滚策略
1. **蓝绿部署**：新旧版本并行，快速切换
2. **金丝雀发布**：逐步放量，降低风险
3. **版本标签**：每个版本打标签，便于回滚
4. **数据库回滚**：数据库变更支持回滚

### 灾难恢复
1. **数据备份**：定期全量和增量备份
2. **异地备份**：跨区域数据备份
3. **恢复演练**：定期灾难恢复演练
4. **文档完善**：完整的恢复流程文档

## 下一步

了解部署架构后，建议：

- **环境搭建**：根据需求选择合适的环境搭建
- **性能测试**：进行部署后的性能测试
- **监控配置**：配置完整的监控和告警
- **安全加固**：根据生产要求进行安全加固