# 专题九：《RAG 系统运维与故障排查：从部署到监控的全流程指南》

## 📚 企业级 RAG 智能问答系统全栈实施指南

### 第九部分：运维部署与故障排查

---

## 第 1 章 Docker Compose 容器化部署

### 1.1 完整服务编排架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RAG 系统 Docker Compose 服务架构                       │
├─────────────────────────────────────────────────────────────────────────┤
│  前端层：Vue3 前端 (Nginx) → 端口 80/443                                  │
│     ↓                                                                   │
│  网关层：Spring Boot API → 端口 8080                                     │
│     ↓                                                                   │
│  服务层：Ollama 大模型 → 端口 11434                                      │
│          Milvus 向量库 → 端口 19530/9091                                 │
│          Redis 缓存 → 端口 6379                                          │
│          MySQL 数据库 → 端口 3306                                        │
│          MinIO 对象存储 → 端口 9000/9001                                 │
│     ↓                                                                   │
│  监控层：Prometheus → 端口 9090                                          │
│          Grafana → 端口 3000                                             │
│          ELK Stack → 端口 5601/9200/5044                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Docker Compose 完整配置

```yaml
# docker-compose.yml

services:
  # ==================== 前端服务 ====================
  rag-frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: rag-frontend
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./frontend/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./frontend/ssl:/etc/nginx/ssl:ro
    environment:
      - NGINX_HOST=rag.yourdomain.com
      - NGINX_PORT=80
    depends_on:
      - rag-backend
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M

  # ==================== 后端服务 ====================
  rag-backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: rag-backend
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/rag_db?useSSL=false&serverTimezone=UTC
      - SPRING_DATASOURCE_USERNAME=${DB_USERNAME}
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
      - SPRING_AI_OLLAMA_BASE_URL=http://ollama:11434
      - SPRING_AI_VECTORSTORE_MILVUS_HOST=milvus
      - SPRING_AI_VECTORSTORE_MILVUS_PORT=19530
      - JWT_SECRET=${JWT_SECRET}
      - AES_KEY=${AES_KEY}
    volumes:
      - ./backend/logs:/app/logs
      - ./backend/data:/app/data
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
      milvus:
        condition: service_healthy
      ollama:
        condition: service_healthy
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G

  # ==================== 大模型服务 ====================
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ./ollama-data:/root/.ollama
      - ./ollama-models:/root/models
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_MODELS=/root/models
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 60s
      timeout: 30s
      retries: 3
      start_period: 120s
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 8G
        reservations:
          cpus: '2.0'
          memory: 4G
    # GPU 支持（NVIDIA）
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: 1
    #           capabilities: [gpu]

  # ==================== 向量数据库 ====================
  milvus:
    image: milvusdb/milvus:v2.6.0
    container_name: milvus
    ports:
      - "19530:19530"
      - "9091:9091"
    environment:
      - ETCD_ENDPOINTS=etcd:2379
      - MINIO_ADDRESS=minio:9000
    volumes:
      - ./milvus-data:/var/lib/milvus
    depends_on:
      - etcd
      - minio
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 8G
        reservations:
          cpus: '2.0'
          memory: 4G

  # Milvus 依赖：etcd
  etcd:
    image: quay.io/coreos/etcd:v3.5.16
    container_name: etcd
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
    volumes:
      - ./etcd-data:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G

  # Milvus 依赖：MinIO
  minio:
    image: minio/minio:RELEASE.2024-01-01T16-36-33Z
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}
    volumes:
      - ./minio-data:/minio_data
    command: minio server /minio_data --console-address ":9001"
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G

  # ==================== 缓存服务 ====================
  redis:
    image: redis:7.2-alpine
    container_name: redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - ./redis-data:/data
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G

  # ==================== 关系数据库 ====================
  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=rag_db
      - MYSQL_USER=${DB_USERNAME}
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - ./mysql-data:/var/lib/mysql
      - ./backend/sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    command: --default-authentication-plugin=mysql_native_password
             --character-set-server=utf8mb4
             --collation-server=utf8mb4_unicode_ci
             --max_connections=500
             --innodb_buffer_pool_size=2G
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G

  # ==================== 监控服务 ====================
  prometheus:
    image: prom/prometheus:v2.52.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    networks:
      - rag-network
    restart: unless-stopped
    depends_on:
      - rag-backend
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G

  grafana:
    image: grafana/grafana:10.4.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource
    volumes:
      - ./grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - rag-network
    restart: unless-stopped
    depends_on:
      - prometheus
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G

  # ==================== 日志服务 ====================
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
      - cluster.name=rag-cluster
    volumes:
      - ./elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - rag-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200/_cluster/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    container_name: logstash
    ports:
      - "5044:5044"
    volumes:
      - ./monitoring/logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./monitoring/logstash/config:/usr/share/logstash/config:ro
    depends_on:
      - elasticsearch
    networks:
      - rag-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - rag-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G

  # ==================== 日志收集器 ====================
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.12.0
    container_name: filebeat
    user: root
    volumes:
      - ./monitoring/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./backend/logs:/var/log/rag:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash
    networks:
      - rag-network
    restart: unless-stopped

networks:
  rag-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

volumes:
  ollama-data:
  milvus-data:
  mysql-data:
  redis-data:
  prometheus-data:
  grafana-data:
  elasticsearch-data:
  minio-data:
  etcd-data:
```

### 1.3 环境变量配置

```bash
# .env 文件
# ==================== 数据库配置 ====================
DB_USERNAME=rag_user
DB_PASSWORD=YourSecurePassword123!
MYSQL_ROOT_PASSWORD=YourRootPassword456!

# ==================== Redis 配置 ====================
REDIS_PASSWORD=YourRedisPassword789!

# ==================== JWT 配置 ====================
JWT_SECRET=YourSuperSecretJWTKeyThatIsAtLeast32CharactersLong!

# ==================== 加密配置 ====================
AES_KEY=YourAES256BitKeyMustBeExactly32Bytes!!

# ==================== MinIO 配置 ====================
MINIO_ROOT_USER=minio_admin
MINIO_ROOT_PASSWORD=YourMinioPassword123!

# ==================== Grafana 配置 ====================
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=YourGrafanaPassword456!

# ==================== 网络配置 ====================
HOST_IP=192.168.1.100
DOMAIN=rag.yourdomain.com
```

### 1.4 服务启动脚本

```bash
#!/bin/bash
# deploy.sh - RAG 系统一键部署脚本

set -e

echo "🚀 开始部署 RAG 系统..."

# 1. 检查 Docker 和 Docker Compose
check_docker() {
    if ! command -v docker &> /dev/null; then
        echo "❌ Docker 未安装，请先安装 Docker"
        exit 1
    fi
    
    if ! command -v docker-compose &> /dev/null; then
        echo "❌ Docker Compose 未安装，请先安装 Docker Compose"
        exit 1
    fi
    
    echo "✅ Docker 版本：$(docker --version)"
    echo "✅ Docker Compose 版本：$(docker-compose --version)"
}

# 2. 创建必要目录
create_directories() {
    echo "📁 创建数据目录..."
    mkdir -p ./backend/logs ./backend/data
    mkdir -p ./mysql-data ./redis-data ./milvus-data
    mkdir -p ./ollama-data ./ollama-models
    mkdir -p ./prometheus-data ./grafana-data
    mkdir -p ./elasticsearch-data ./minio-data ./etcd-data
    mkdir -p ./monitoring/prometheus ./monitoring/grafana ./monitoring/logstash
    mkdir -p ./frontend/ssl
}

# 3. 设置文件权限
set_permissions() {
    echo "🔐 设置文件权限..."
    chmod -R 755 ./backend/logs ./backend/data
    chmod -R 777 ./mysql-data ./redis-data ./milvus-data
    chmod -R 777 ./elasticsearch-data
}

# 4. 加载环境变量
load_env() {
    echo "📋 加载环境变量..."
    if [ ! -f .env ]; then
        echo "❌ .env 文件不存在，请复制 .env.example 并修改配置"
        exit 1
    fi
    export $(cat .env | grep -v '^#' | xargs)
}

# 5. 启动服务
start_services() {
    echo "🐳 启动 Docker 服务..."
    docker-compose up -d
    
    echo "⏳ 等待服务启动..."
    sleep 30
}

# 6. 健康检查
health_check() {
    echo "🏥 执行健康检查..."
    
    services=("rag-frontend" "rag-backend" "ollama" "milvus" "redis" "mysql" "prometheus" "grafana")
    
    for service in "${services[@]}"; do
        if docker inspect --format='{{.State.Health.Status}}' $service 2>/dev/null | grep -q "healthy"; then
            echo "✅ $service 健康"
        else
            echo "⚠️  $service 未就绪，继续等待..."
        fi
    done
}

# 7. 加载 Ollama 模型
load_ollama_model() {
    echo "🤖 加载 Ollama 模型..."
    docker exec ollama ollama pull qwen2.5:1.5b
    docker exec ollama ollama pull bge-m3
}

# 8. 初始化数据库
init_database() {
    echo "💾 初始化数据库..."
    sleep 10
    docker exec mysql mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "CREATE DATABASE IF NOT EXISTS rag_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
    docker exec mysql mysql -uroot -p${MYSQL_ROOT_PASSWORD} rag_db < ./backend/sql/init.sql
}

# 9. 显示访问信息
show_access_info() {
    echo ""
    echo "=========================================="
    echo "🎉 RAG 系统部署完成！"
    echo "=========================================="
    echo ""
    echo "📱 前端访问：http://${HOST_IP}"
    echo "🔧 后端 API：http://${HOST_IP}:8080/api"
    echo "📊 Grafana：http://${HOST_IP}:3000"
    echo "🔍 Prometheus：http://${HOST_IP}:9090"
    echo "📝 Kibana：http://${HOST_IP}:5601"
    echo "💾 MinIO：http://${HOST_IP}:9000"
    echo ""
    echo "=========================================="
    echo "⚠️  请妥善保管 .env 文件中的密码信息！"
    echo "=========================================="
}

# 主流程
main() {
    check_docker
    create_directories
    set_permissions
    load_env
    start_services
    health_check
    load_ollama_model
    init_database
    show_access_info
}

main "$@"
```

---

## 第 2 章 日志聚合方案

### 2.1 ELK Stack 日志架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ELK 日志聚合架构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  Filebeat → Logstash → Elasticsearch → Kibana                           │
│     ↓           ↓            ↓              ↓                            │
│  日志收集    日志处理      日志存储        日志可视化                      │
│                                                                        │
│  支持来源：                                                             │
│  - Spring Boot 应用日志                                                 │
│  - Nginx 访问日志                                                       │
│  - Docker 容器日志                                                      │
│  - 系统日志                                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Filebeat 配置

```yaml
# monitoring/filebeat/filebeat.yml
filebeat.inputs:
  # Spring Boot 应用日志
  - type: log
    enabled: true
    paths:
      - /var/log/rag/*.log
    fields:
      service: rag-backend
      log_type: application
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after
    
  # Nginx 访问日志
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      service: nginx
      log_type: access
    json.keys_under_root: true
    
  # Docker 容器日志
  - type: container
    enabled: true
    paths:
      - /var/lib/docker/containers/*/*.log
    fields:
      log_type: docker
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"

# Logstash 输出
output.logstash:
  hosts: ["logstash:5044"]

# 监控配置
monitoring:
  enabled: true
  elasticsearch:
    hosts: ["http://elasticsearch:9200"]
```

### 2.3 Logstash 管道配置

```ruby
# monitoring/logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # 解析 Spring Boot 日志
  if [fields][service] == "rag-backend" {
    grok {
      match => { 
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} $$%{DATA:thread}$$ %{DATA:class} - %{GREEDYDATA:log_message}"
      }
    }
    
    # 解析 JSON 日志
    json {
      source => "log_message"
      target => "json_data"
    }
    
    # 添加环境标签
    mutate {
      add_field => { "environment" => "production" }
    }
  }
  
  # 解析 Nginx 日志
  if [fields][service] == "nginx" {
    grok {
      match => { 
        "message" => "%{IPORHOST:clientip} - %{USER:ident} %{USER:auth} $$%{HTTPDATE:timestamp}$$ \"%{WORD:verb} %{DATA:request} HTTP/%{NUMBER:httpversion}\" %{NUMBER:status} %{NUMBER:bytes}"
      }
    }
    
    # 地理位置解析
    geoip {
      source => "clientip"
      target => "geoip"
    }
  }
  
  # 删除不必要字段
  mutate {
    remove_field => ["host", "agent", "ecs"]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "rag-logs-%{+YYYY.MM.dd}"
  }
  
  # 调试输出（生产环境关闭）
  # stdout { codec => rubydebug }
}
```

### 2.4 日志索引模板

```json
// monitoring/elasticsearch/index-template.json
{
  "index_patterns": ["rag-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "30s",
      "index.lifecycle.name": "rag-logs-policy",
      "index.lifecycle.rollover_alias": "rag-logs"
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" },
        "log_type": { "type": "keyword" },
        "message": { 
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "class": { "type": "keyword" },
        "thread": { "type": "keyword" },
        "clientip": { "type": "ip" },
        "status": { "type": "integer" },
        "geoip": { "type": "object" }
      }
    }
  }
}
```

### 2.5 日志生命周期管理

```json
// monitoring/elasticsearch/ilm-policy.json
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "7d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## 第 3 章 监控告警体系

### 3.1 Prometheus 监控配置

```yaml
# monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'rag-production'
    monitor: 'rag-monitor'

# 告警规则
rule_files:
  - "/etc/prometheus/rules/*.yml"

# 抓取配置
scrape_configs:
  # Spring Boot 应用
  - job_name: 'rag-backend'
    static_configs:
      - targets: ['rag-backend:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    
  # Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    
  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    
  # MySQL
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']
    
  # Redis
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
    
  # Milvus
  - job_name: 'milvus'
    static_configs:
      - targets: ['milvus:9091']
    
  # Ollama
  - job_name: 'ollama'
    static_configs:
      - targets: ['ollama:11434']
    metrics_path: '/api/metrics'
```

### 3.2 告警规则配置

```yaml
# monitoring/prometheus/rules/alerts.yml
groups:
  - name: rag-alerts
    rules:
      # 服务宕机告警
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服务 {{ $labels.job }} 宕机"
          description: "{{ $labels.instance }} 已宕机超过 1 分钟"
      
      # 高延迟告警
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "API 延迟过高"
          description: "P99 延迟 {{ $value }}s 超过阈值 2s"
      
      # 错误率告警
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "错误率过高"
          description: "5xx 错误率 {{ $value | humanizePercentage }} 超过阈值 5%"
      
      # 内存使用率告警
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "内存使用率过高"
          description: "内存使用率 {{ $value | humanizePercentage }}"
      
      # 磁盘使用率告警
      - alert: HighDiskUsage
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "磁盘使用率过高"
          description: "磁盘使用率 {{ $value | humanizePercentage }}"
      
      # Ollama 模型加载失败
      - alert: OllamaModelNotLoaded
        expr: ollama_model_loaded == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Ollama 模型未加载"
          description: "请检查模型是否正确加载"
      
      # Milvus 连接池耗尽
      - alert: MilvusConnectionPoolExhausted
        expr: milvus_connection_pool_active / milvus_connection_pool_total > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Milvus 连接池即将耗尽"
          description: "连接池使用率 {{ $value | humanizePercentage }}"
```

### 3.3 Grafana 仪表盘配置

```json
{
  "dashboard": {
    "title": "RAG 系统监控大盘",
    "panels": [
      {
        "title": "系统概览",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"rag-backend\"}",
            "legendFormat": "后端服务"
          },
          {
            "expr": "up{job=\"milvus\"}",
            "legendFormat": "Milvus"
          },
          {
            "expr": "up{job=\"ollama\"}",
            "legendFormat": "Ollama"
          }
        ]
      },
      {
        "title": "请求 QPS",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[1m])",
            "legendFormat": "QPS"
          }
        ]
      },
      {
        "title": "响应延迟（P50/P95/P99）",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P99"
          }
        ]
      },
      {
        "title": "缓存命中率",
        "type": "gauge",
        "targets": [
          {
            "expr": "rate(cache_hits_total[5m]) / rate(cache_requests_total[5m]) * 100",
            "legendFormat": "Hit Rate %"
          }
        ]
      },
      {
        "title": "JVM 内存使用",
        "type": "graph",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"}",
            "legendFormat": "Heap Used"
          },
          {
            "expr": "jvm_memory_max_bytes{area=\"heap\"}",
            "legendFormat": "Heap Max"
          }
        ]
      },
      {
        "title": "Milvus 检索延迟",
        "type": "graph",
        "targets": [
          {
            "expr": "milvus_search_latency_seconds",
            "legendFormat": "Search Latency"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

### 3.4 告警通知集成

```yaml
# monitoring/prometheus/alertmanager.yml
global:
  smtp_smarthost: 'smtp.yourdomain.com:587'
  smtp_from: 'alert@rag.yourdomain.com'
  smtp_auth_username: 'alert@rag.yourdomain.com'
  smtp_auth_password: '${SMTP_PASSWORD}'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'default'
    email_configs:
      - to: 'ops-team@yourdomain.com'
        send_resolved: true
  
  - name: 'critical-alerts'
    email_configs:
      - to: 'ops-team@yourdomain.com'
        send_resolved: true
    webhook_configs:
      - url: 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_TOKEN}'
        send_resolved: true
  
  - name: 'warning-alerts'
    email_configs:
      - to: 'dev-team@yourdomain.com'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

---

## 第 4 章 故障诊断流程

### 4.1 故障排查检查清单

```markdown
## 🔧 RAG 系统故障排查清单

### 服务不可用
- [ ] 检查 Docker 容器状态：`docker-compose ps`
- [ ] 查看容器日志：`docker-compose logs <service>`
- [ ] 检查端口监听：`netstat -tlnp | grep <port>`
- [ ] 验证健康检查：`curl http://localhost:8080/actuator/health`
- [ ] 检查网络连接：`docker network inspect rag-network`

### 响应缓慢
- [ ] 查看 CPU 使用率：`docker stats`
- [ ] 查看内存使用：`docker stats`
- [ ] 检查慢查询日志
- [ ] 查看 GC 日志：`docker exec rag-backend cat logs/gc.log`
- [ ] 检查数据库连接池

### 向量检索失败
- [ ] 检查 Milvus 状态：`docker exec milvus curl localhost:9091/healthz`
- [ ] 查看 Milvus 日志：`docker-compose logs milvus`
- [ ] 检查集合状态：`milvus-cli show collection`
- [ ] 验证索引状态：`milvus-cli show index`
- [ ] 检查 etcd 和 MinIO 依赖

### Ollama 模型问题
- [ ] 检查模型列表：`docker exec ollama ollama list`
- [ ] 重新拉取模型：`docker exec ollama ollama pull qwen2.5:1.5b`
- [ ] 查看 Ollama 日志：`docker-compose logs ollama`
- [ ] 检查 GPU 状态（如有）：`nvidia-smi`

### 数据库问题
- [ ] 检查 MySQL 状态：`docker exec mysql mysqladmin status`
- [ ] 查看慢查询：`docker exec mysql cat /var/log/mysql/slow.log`
- [ ] 检查连接数：`docker exec mysql mysql -e "SHOW PROCESSLIST"`
- [ ] 验证数据完整性：`docker exec mysql mysqlcheck -u root -p --all-databases`

### Redis 问题
- [ ] 检查 Redis 状态：`docker exec redis redis-cli ping`
- [ ] 查看内存使用：`docker exec redis redis-cli info memory`
- [ ] 检查键数量：`docker exec redis redis-cli dbsize`
- [ ] 查看慢日志：`docker exec redis redis-cli slowlog get 10`
```

### 4.2 常用诊断命令

```bash
#!/bin/bash
# diagnose.sh - RAG 系统诊断脚本

echo "=========================================="
echo "🔍 RAG 系统诊断报告"
echo "=========================================="
echo ""

# 1. 容器状态
echo "📦 容器状态:"
docker-compose ps
echo ""

# 2. 资源使用
echo "💻 资源使用情况:"
docker stats --no-stream
echo ""

# 3. 网络检查
echo "🌐 网络检查:"
docker network inspect rag-network --format='{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{end}}'
echo ""

# 4. 服务健康检查
echo "🏥 服务健康检查:"
echo -n "后端服务: "
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health || echo "失败"
echo ""

echo -n "Milvus: "
docker exec milvus curl -s -o /dev/null -w "%{http_code}" http://localhost:9091/healthz || echo "失败"
echo ""

echo -n "Ollama: "
docker exec ollama ollama list > /dev/null 2>&1 && echo "正常" || echo "失败"
echo ""

echo -n "Redis: "
docker exec redis redis-cli ping > /dev/null 2>&1 && echo "正常" || echo "失败"
echo ""

echo -n "MySQL: "
docker exec mysql mysqladmin -uroot -p${MYSQL_ROOT_PASSWORD} status > /dev/null 2>&1 && echo "正常" || echo "失败"
echo ""

# 5. 磁盘使用
echo "💾 磁盘使用:"
df -h | grep -E "Filesystem|/dev/sda"
echo ""

# 6. 最近错误日志
echo "📝 最近错误日志:"
docker-compose logs --tail=50 rag-backend | grep -i "error\|exception" | tail -20
echo ""

# 7. 端口监听
echo "🔌 端口监听:"
netstat -tlnp | grep -E "8080|11434|19530|6379|3306|9090|3000"
echo ""

echo "=========================================="
echo "诊断完成！"
echo "=========================================="
```

### 4.3 线程 dump 分析

```bash
# 获取 Java 应用线程 dump
docker exec rag-backend jps -l
docker exec rag-backend jstack <pid> > thread_dump_$(date +%Y%m%d_%H%M%S).txt

# 分析线程状态
grep -A 10 "BLOCKED\|DEADLOCK" thread_dump_*.txt

# 获取 JVM 内存信息
docker exec rag-backend jstat -gc <pid> 1000 10

# 生成堆 dump（谨慎使用，可能导致服务暂停）
docker exec rag-backend jmap -dump:format=b,file=/app/data/heap_dump.hprof <pid>
```

### 4.4 数据库诊断

```sql
-- 查看当前连接
SHOW PROCESSLIST;

-- 查看慢查询
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- 查看表大小
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.TABLES
WHERE table_schema = 'rag_db'
ORDER BY (data_length + index_length) DESC;

-- 查看锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 查看索引使用情况
SHOW INDEX FROM audit_log;

-- 优化表
OPTIMIZE TABLE audit_log;

-- 分析表
ANALYZE TABLE audit_log;
```

---

## 第 5 章 灾备恢复策略

### 5.1 备份策略

```yaml
# 备份计划配置
backup:
  # 数据库备份
  mysql:
    enabled: true
    schedule: "0 2 * * *"  # 每天凌晨 2 点
    retention_days: 30
    backup_path: /backup/mysql
    
  # 向量数据备份
  milvus:
    enabled: true
    schedule: "0 3 * * *"  # 每天凌晨 3 点
    retention_days: 14
    backup_path: /backup/milvus
    
  # 配置文件备份
  config:
    enabled: true
    schedule: "0 4 * * 0"  # 每周日凌晨 4 点
    retention_days: 90
    backup_path: /backup/config
    
  # 日志备份
  logs:
    enabled: true
    schedule: "0 5 * * *"  # 每天凌晨 5 点
    retention_days: 7
    backup_path: /backup/logs
```

### 5.2 自动化备份脚本

```bash
#!/bin/bash
# backup.sh - RAG 系统备份脚本

set -e

BACKUP_DIR="/backup/rag_$(date +%Y%m%d_%H%M%S)"
RETENTION_DAYS=30

echo "🔄 开始备份 RAG 系统..."
mkdir -p $BACKUP_DIR

# 1. 备份 MySQL 数据库
echo "💾 备份 MySQL 数据库..."
docker exec mysql mysqldump -uroot -p${MYSQL_ROOT_PASSWORD} \
    --all-databases \
    --single-transaction \
    --quick \
    --lock-tables=false \
    > $BACKUP_DIR/mysql_backup.sql

# 2. 备份 Redis 数据
echo "💾 备份 Redis 数据..."
docker exec redis redis-cli BGSAVE
sleep 5
cp ./redis-data/dump.rdb $BACKUP_DIR/redis_dump.rdb 2>/dev/null || true

# 3. 备份 Milvus 数据
echo "💾 备份 Milvus 数据..."
docker cp milvus:/var/lib/milvus $BACKUP_DIR/milvus_data

# 4. 备份 Ollama 模型
echo "💾 备份 Ollama 模型..."
cp -r ./ollama-data $BACKUP_DIR/ollama_data

# 5. 备份配置文件
echo "💾 备份配置文件..."
cp docker-compose.yml $BACKUP_DIR/
cp .env $BACKUP_DIR/
cp -r ./backend $BACKUP_DIR/backend
cp -r ./frontend $BACKUP_DIR/frontend
cp -r ./monitoring $BACKUP_DIR/monitoring

# 6. 压缩备份
echo "📦 压缩备份文件..."
tar -czf $BACKUP_DIR.tar.gz -C $(dirname $BACKUP_DIR) $(basename $BACKUP_DIR)
rm -rf $BACKUP_DIR

# 7. 清理旧备份
echo "🧹 清理 $RETENTION_DAYS 天前的备份..."
find /backup -name "rag_*.tar.gz" -mtime +$RETENTION_DAYS -delete

# 8. 上传到远程存储（可选）
# echo "☁️  上传到远程存储..."
# aws s3 cp $BACKUP_DIR.tar.gz s3://your-bucket/backups/

echo "✅ 备份完成：$BACKUP_DIR.tar.gz"
echo "📊 备份大小：$(du -h $BACKUP_DIR.tar.gz | cut -f1)"
```

### 5.3 灾难恢复流程

```bash
#!/bin/bash
# restore.sh - RAG 系统恢复脚本

set -e

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "❌ 请指定备份文件"
    echo "用法：./restore.sh /backup/rag_20260115_020000.tar.gz"
    exit 1
fi

if [ ! -f "$BACKUP_FILE" ]; then
    echo "❌ 备份文件不存在：$BACKUP_FILE"
    exit 1
fi

echo "⚠️  警告：此操作将覆盖现有数据！"
read -p "确定要继续吗？(yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    echo "❌ 取消恢复操作"
    exit 0
fi

echo "🔄 开始恢复 RAG 系统..."

# 1. 停止所有服务
echo "🛑 停止所有服务..."
docker-compose down

# 2. 解压备份
echo "📦 解压备份文件..."
BACKUP_DIR=$(dirname $BACKUP_FILE)/$(basename $BACKUP_FILE .tar.gz)
tar -xzf $BACKUP_FILE -C $(dirname $BACKUP_FILE)

# 3. 恢复配置文件
echo "📋 恢复配置文件..."
cp $BACKUP_DIR/docker-compose.yml ./
cp $BACKUP_DIR/.env ./

# 4. 恢复 MySQL 数据
echo "💾 恢复 MySQL 数据..."
docker-compose up -d mysql
sleep 30
docker exec -i mysql mysql -uroot -p${MYSQL_ROOT_PASSWORD} < $BACKUP_DIR/mysql_backup.sql

# 5. 恢复 Redis 数据
echo "💾 恢复 Redis 数据..."
if [ -f "$BACKUP_DIR/redis_dump.rdb" ]; then
    cp $BACKUP_DIR/redis_dump.rdb ./redis-data/dump.rdb
fi
docker-compose up -d redis

# 6. 恢复 Milvus 数据
echo "💾 恢复 Milvus 数据..."
rm -rf ./milvus-data/*
cp -r $BACKUP_DIR/milvus_data/* ./milvus-data/
docker-compose up -d milvus etcd minio

# 7. 恢复 Ollama 模型
echo "💾 恢复 Ollama 模型..."
rm -rf ./ollama-data/*
cp -r $BACKUP_DIR/ollama_data/* ./ollama-data/
docker-compose up -d ollama

# 8. 启动所有服务
echo "🚀 启动所有服务..."
docker-compose up -d

# 9. 验证恢复
echo "🏥 验证服务状态..."
sleep 60
docker-compose ps

echo "✅ 恢复完成！"
echo "请检查各服务是否正常启动"
```

### 5.4 高可用部署方案

```yaml
# docker-compose.ha.yml - 高可用配置
version: '3.8'

services:
  # MySQL 主从复制
  mysql-master:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_REPLICATION_MODE=master
    volumes:
      - ./mysql-master-data:/var/lib/mysql
    networks:
      - rag-network
  
  mysql-slave:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_REPLICATION_MODE=slave
      - MYSQL_MASTER_HOST=mysql-master
    volumes:
      - ./mysql-slave-data:/var/lib/mysql
    depends_on:
      - mysql-master
    networks:
      - rag-network
  
  # Redis Sentinel
  redis-master:
    image: redis:7.2-alpine
    networks:
      - rag-network
  
  redis-slave:
    image: redis:7.2-alpine
    command: redis-server --slaveof redis-master 6379
    depends_on:
      - redis-master
    networks:
      - rag-network
  
  redis-sentinel:
    image: redis:7.2-alpine
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
      - ./redis-sentinel.conf:/usr/local/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-slave
    networks:
      - rag-network
  
  # Milvus 集群
  milvus-standalone:
    # 生产环境建议使用 Milvus 集群版
    image: milvusdb/milvus:v2.6.0
    deploy:
      replicas: 2
    networks:
      - rag-network

networks:
  rag-network:
    driver: overlay
```

---

## 第 6 章 性能优化与容量规划

### 6.1 资源配额建议

| 服务 | CPU | 内存 | 磁盘 | 适用规模 |
|------|-----|------|------|----------|
| **前端 (Nginx)** | 0.5 核 | 512MB | 10GB | 1000 QPS |
| **后端 (Spring Boot)** | 2 核 | 2GB | 20GB | 500 QPS |
| **Ollama** | 4 核 | 8GB | 50GB | 1.5B 模型 |
| **Milvus** | 4 核 | 8GB | 100GB | 100 万向量 |
| **MySQL** | 2 核 | 4GB | 50GB | 1000 万记录 |
| **Redis** | 1 核 | 2GB | 10GB | 100 万键 |
| **Elasticsearch** | 2 核 | 4GB | 100GB | 1 亿日志 |

### 6.2 扩容策略

```bash
#!/bin/bash
# scale.sh - 服务扩容脚本

SERVICE=$1
REPLICAS=$2

if [ -z "$SERVICE" ] || [ -z "$REPLICAS" ]; then
    echo "用法：./scale.sh <service> <replicas>"
    echo "示例：./scale.sh rag-backend 3"
    exit 1
fi

echo "🔄 扩容 $SERVICE 到 $REPLICAS 个实例..."

# Docker Swarm 模式
docker service scale ${SERVICE}=${REPLICAS}

# 或 Kubernetes 模式
# kubectl scale deployment ${SERVICE} --replicas=${REPLICAS}

echo "✅ 扩容完成"
docker service ps ${SERVICE}
```

### 6.3 性能基准测试

```bash
#!/bin/bash
# benchmark.sh - 性能基准测试

echo "🚀 开始性能基准测试..."

# 1. 单请求延迟测试
echo "📊 单请求延迟测试..."
ab -n 100 -c 1 http://localhost:8080/api/rag/query

# 2. 并发测试
echo "📊 并发测试..."
ab -n 1000 -c 50 http://localhost:8080/api/rag/query

# 3. 压力测试
echo "📊 压力测试..."
wrk -t12 -c400 -d30s http://localhost:8080/api/rag/query

# 4. 向量检索性能
echo "📊 向量检索性能..."
for i in {1..100}; do
    curl -s -X POST http://localhost:8080/api/rag/query \
        -H "Content-Type: application/json" \
        -d '{"question":"测试问题"}' \
        | jq '.latency'
done | awk '{sum+=$1} END {print "平均延迟:", sum/NR, "ms"}'
```

---

## 第 7 章 运维最佳实践

### 7.1 日常运维检查清单

```markdown
## 📋 RAG 系统日常运维检查清单

### 每日检查
- [ ] 检查服务健康状态
- [ ] 查看错误日志
- [ ] 监控 CPU/内存使用率
- [ ] 检查磁盘空间
- [ ] 验证备份完整性

### 每周检查
- [ ] 分析慢查询日志
- [ ] 清理过期日志
- [ ] 更新安全补丁
- [ ] 审查审计日志
- [ ] 性能趋势分析

### 每月检查
- [ ] 容量规划评估
- [ ] 灾难恢复演练
- [ ] 安全漏洞扫描
- [ ] 成本优化分析
- [ ] 文档更新
```

### 7.2 安全加固建议

```bash
# 1. 限制容器权限
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE

# 2. 使用只读文件系统
docker run --read-only --tmpfs /tmp

# 3. 限制资源
docker run --memory=2g --cpus=2.0

# 4. 网络隔离
docker network create --driver bridge --internal rag-internal

# 5. 定期更新镜像
docker-compose pull
docker-compose up -d
```

### 7.3 故障升级流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        故障升级流程                                      │
├─────────────────────────────────────────────────────────────────────────┤
│  L1: 自动恢复 → 健康检查失败自动重启 (0-5 分钟)                           │
│     ↓                                                                   │
│  L2: 运维处理 → 运维团队介入排查 (5-30 分钟)                              │
│     ↓                                                                   │
│  L3: 开发支持 → 开发团队协助定位 (30-60 分钟)                             │
│     ↓                                                                   │
│  L4: 厂商支持 → 联系技术供应商 (60 分钟+)                                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

**本专题完**

> 📌 **核心要点总结**：
> 1. Docker Compose 是 RAG 系统容器化部署的最佳选择，支持完整服务编排
> 2. ELK Stack 日志聚合方案可实现全链路日志收集和分析
> 3. Prometheus + Grafana 监控体系覆盖应用、数据库、中间件全栈
> 4. 故障诊断需遵循标准化流程，从服务状态→资源使用→日志分析
> 5. 备份策略应包含数据库、向量数据、配置文件、模型数据
> 6. 灾难恢复脚本需定期演练，确保恢复流程有效
> 7. 高可用部署需考虑主从复制、哨兵模式、集群部署

> 📖 **系列完结**：恭喜完成《企业级 RAG 智能问答系统全栈实施指南》全部 9 个专题！从架构设计到运维部署，您已掌握完整的 RAG 系统实施能力。