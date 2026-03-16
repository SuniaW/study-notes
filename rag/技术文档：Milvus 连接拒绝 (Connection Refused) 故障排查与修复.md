

# 技术文档：Milvus 连接拒绝 (Connection Refused) 故障排查与修复

## 1. 问题描述
**错误信息：**
`io.grpc.netty.shaded.io.netty.channel.AbstractChannel$AnnotatedConnectException: Connection refused: getsockopt: /8.140.221.150:19530`

**现象：**
客户端无法通过 gRPC 协议连接到位于 `8.140.221.150` 的 Milvus 服务，端口 `19530` 拒绝连接请求。

---

## 2. 核心原因分析
导致此错误通常有以下四种可能：
1. **服务未启动**：Milvus 容器或进程已停止。
2. **云端防火墙阻断**：阿里云安全组未放行 19530 端口。
3. **监听地址错误**：Milvus 仅监听在 `127.0.0.1`，未监听公网 IP。
4. **依赖组件异常**：Milvus 依赖的 etcd 或 MinIO 挂掉，导致 Proxy 无法正常工作。

---

## 3. 故障排查步骤

### 第一步：验证网络连通性
在您的**开发机（客户端）**上尝试连接该端口，判断是网络阻断还是服务崩溃。
```bash
# 使用 telnet 或 nc
telnet 8.140.221.150 19530
```
*   **若提示 Timeout**：通常是防火墙或安全组问题。
*   **若提示 Connection Refused**：通常是服务端进程未运行。

### 第二步：检查服务端状态
登录到服务器 `8.140.221.150`，查看容器运行情况：
```bash
# 查看所有 Milvus 相关容器
docker ps | grep milvus
```
*   确保 `milvus-standalone` 或 `milvus-proxy` 的状态为 **Up**。
*   检查端口监听：`netstat -tuln | grep 19530`。

### 第三步：检查阿里云安全组
由于 IP 属于阿里云，需确认云端设置：
1. 登录 **阿里云控制台** -> **ECS 实例**。
2. 进入该实例的 **安全组** 配置。
3. 在 **入方向规则** 中添加：**协议 TCP，端口 19530，授权对象 0.0.0.0/0**（或您的特定 IP）。

---

## 4. 解决方案：重启服务

根据部署方式选择对应的重启命令：

### 方案 A：Docker Compose 部署（推荐）
进入 `docker-compose.yml` 所在的目录执行：
```bash
# 1. 彻底停止并清理旧容器
docker-compose down

# 2. 重新后台启动
docker-compose up -d

# 3. 查看日志确保没有报错
docker-compose logs -f
```

### 方案 B：Kubernetes 部署
```bash
# 重启 Deployment
kubectl rollout restart deployment milvus-standalone
```

---

## 5. 常见日志异常及对策
如果重启后依然无法连接，请检查日志：
`docker logs milvus-standalone`

| 日志关键字 | 原因 | 对策 |
| :--- | :--- | :--- |
| `Failed to connect to etcd` | etcd 服务异常 | 检查 etcd 容器状态，确保其磁盘未满 |
| `Failed to connect to MinIO` | MinIO 服务异常 | 检查 MinIO 端口和存储卷权限 |
| `Address already in use` | 端口冲突 | 停止占用 19530 端口的其他进程 |

---

## 6. 预防措施
1. **监控报警**：建议对 19530 端口进行存活监控。
2. **固定 IP/DNS**：确保客户端访问的 IP 没有因为服务器重启而发生变更。
3. **资源限制**：检查服务器内存，防止因 OOM (Out of Memory) 导致 Milvus 进程被操作系统杀掉。

---

**文档说明：**
*   **创建日期**：2024年5月
*   **适用范围**：Milvus 2.x 系列版本连接故障排除。