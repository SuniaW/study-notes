这份技术手册汇总了我们在 **2核4G 阿里云服务器**上，从系统假死到实现 **2.7GB 内存盈余** 并成功运行 **Spring Boot 3.4.3 + Spring AI 1.0.0-M6 + Ollama + Milvus** 的全过程实战经验。

---

# 🚀 阿里云 2核4G 服务器：私有化 RAG 系统全栈调优与避坑手册

> **项目背景：** 在极度受限的 4G 内存环境下，运行大模型推理（Ollama）、向量数据库（Milvus）及微服务（Spring Boot）。
> **核心挑战：** 物理内存溢出导致系统频繁使用 Swap 产生 I/O Wait（wa），使 CPU 陷入假死；Spring AI 早期版本配置读取失效。

---

## 第一阶段：系统级“深层脱水”（释放物理资源）

在资源受限环境下，必须清理所有非必要系统开销，将物理内存留给 AI 核心任务。

### 1. 移除云厂商冗余监控 (回血 ~200MB)
阿里云盾（AliYunDun）及其监控插件在内存告急时会产生明显的资源争抢。
*   **执行指令：**
    ```bash
    # 运行官方卸载脚本
    wget http://update.aegis.aliyun.com/download/uninstall.sh && chmod +x uninstall.sh && ./uninstall.sh
    wget http://update.aegis.aliyun.com/download/quartz_uninstall.sh && chmod +x quartz_uninstall.sh && ./quartz_uninstall.sh
    # 强制清理残留进程
    pkill aliyun-service && rm -rf /usr/local/aegis /usr/sbin/aliyun-service
    ```

### 2. 内核虚拟内存优化 (Swappiness)
默认 `swappiness=60` 会过早动用硬盘 Swap。将其调低，强制系统优先驻留物理内存。
*   **优化指令：**
    ```bash
    sudo sysctl vm.swappiness=10
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    ```

### 3. Docker 冗余清理与日志限额
*   **深度清理：** 移除无用镜像、挂起容器和网络插件缓存。
    ```bash
    docker system prune -a -f
    ```
*   **日志限制：** 修改 `/etc/docker/daemon.json` 限制日志文件大小。
    ```json
    { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }
    ```

### 4. 实时释放系统缓存
在启动大型组件（如 Spring Boot）前，手动释放 PageCache。
```bash
# 限制系统日志在 100MB 并在内存告急时释放缓存
sudo journalctl --vacuum-size=100M
sync; echo 3 > /proc/sys/vm/drop_caches
```

---

## 第二阶段：中间件“节食”方案（资源硬性配额）

### 1. Ollama 模型生命周期管控
防止大模型推理结束后长期霸占物理内存。
*   **配置修改：** 执行 `systemctl edit ollama.service` 添加：
    ```ini
    [Service]
    Environment="OLLAMA_KEEP_ALIVE=5m"   # 闲置5分钟自动释放内存
    Environment="OLLAMA_NUM_PARALLEL=1"   # 串行推理，严防内存峰值
    ```

### 2. Milvus 向量数据库限额 (Docker 层面)
*   **内存硬限制：** 在 `docker-compose.yml` 中严控 Standalone 容器上限。
    ```yaml
    deploy:
      resources:
        limits:
          memory: 1024M  # 严控在 1GB 以内
    environment:
      CACHE_SIZE: 512M   # 限制内部缓存
    ```

---

## 第三阶段：Spring AI 1.0.0-M6 配置修复（“配置降维打击”）

### 1. 解决 `collection-name` 配置失效问题
**现象：** YAML 配置被忽略，程序顽固寻找默认的 `vector_store` 集合。
**根源：** M6 版本自动配置逻辑不完善，或包扫描路径（com.wx.policyragsystem）与配置路径未对齐。
**必杀技：** 编写 Java 配置类，使用 `@Primary` 强制接管并指定 `initializeSchema(true)`。

```java
@Configuration
@EnableConfigurationProperties(MilvusVectorStoreProperties.class) // 💡核心：强制绑定源码属性类
public class VectorStoreConfig {
    @Bean
    @Primary
    public MilvusVectorStore vectorStore(MilvusServiceClient client, 
                                         EmbeddingModel model, 
                                         MilvusVectorStoreProperties properties) {
        // 显式指定集合名和维度，防止维度冲突 (BGE-M3 = 1024)
        MilvusVectorStore.MilvusVectorStoreConfig config = MilvusVectorStore.MilvusVectorStoreConfig.builder()
            .withCollectionName(properties.getCollectionName())
            .withEmbeddingDimension(properties.getEmbeddingDimension())
            .build();
        return MilvusVectorStore.builder(client, model)
            .collectionName(properties.getCollectionName())
            .initializeSchema(true) // 💡启动时自动创建集合，防止 similaritySearch 报错
            .build();
    }
}
```

### 2. JVM 内存红线限制
```bash
# 除了 Xmx，必须限制 Metaspace (元空间)，否则物理内存会持续增长
java -Xms512m -Xmx1024m -XX:MaxMetaspaceSize=256m -jar app.jar
```

---

## 第四阶段：数据一致性与 Attu 运维指南

1.  **维度对齐：** BGE-M3 模型对应 **1024 维**。如果 Attu 中存在旧的 768 维集合，搜索必报错。
2.  **清空重建：** 在 Attu 中 **Drop** 掉旧集合。由于开启了 `initializeSchema(true)`，程序重启或第一次调用 `.add()` 时会自动重建 1024 维的正确集合。
3.  **集合加载：** 确保集合在 Attu 中显示为 `Loaded` (已加载) 状态，否则检索无结果。

---

## 第五阶段：运行维护“生存法则”

### 1. 严禁服务器本地编译
*   **原则：** 绝对不要在 4G 服务器运行 `mvn clean install`。
*   **风险：** Maven 编译过程极大消耗内存，会诱发系统 OOM 导致服务器假死。
*   **建议：** 本地编译 JAR 包，通过 SCP/FTP 上传。

### 2. 稳健启动顺序
1.  **检查 Avail Mem**：确保空闲（理想状态：> 2.5GB）。
2.  **启动 Milvus**。
3.  **启动 Spring Boot**：等待启动完成，维持 Avail Mem 在 1.0GB - 1.5GB。
4.  **模型预热**：通过 `curl` 提前让 Ollama 加载模型。

### 3. 核心监控看板 (Top 关键指标)
*   **Avail Mem (可用内存)：**
    *   **> 1.0GB**：标准平衡区（可支撑推理）。
    *   **< 200MB**：卡顿警戒区（建议重启 Ollama 释放模型）。
*   **Swap used：** 只要 **I/O Wait (wa)** 接近 0，Swap 占用 1GB 左右是健康的，说明系统成功完成了非活跃数据的置换。

---

**✨ 调优成果：**
通过本手册操作，系统可用内存已从 **400MB** 的假死临界点回升至 **2.7GB**（清爽态）及 **1.0GB**（标准运行态），成功解决了私有化 RAG 系统的性能瓶颈。