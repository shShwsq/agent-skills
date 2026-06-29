# Browserless 浏览器服务配置指南

本文档说明如何部署和配置 Browserless 浏览器服务，供 `manual-generate-fudan` skill 使用。

## 快速启动（测试环境）

最简单的方式，直接运行官方 Docker 镜像。有两个镜像源可选：

**Docker Hub：**
```bash
docker run -d \
  --name browserless \
  -p 3000:3000 \
  browserless/chrome
```

**GitHub Container Registry（推荐，更新更快）：**
```bash
docker run -d \
  --name browserless \
  -p 3000:3000 \
  ghcr.io/browserless/chromium
```

> **提示**：`ghcr.io/browserless/chromium` 是官方推荐的镜像，版本更新更及时。两者功能相同。

启动后，Browserless 服务地址为：`http://localhost:3000`（如果是远程服务器，替换为服务器 IP）。

### 验证服务是否正常

```bash
# 检查服务状态
curl http://localhost:3000/pressure

# 预期返回类似：
# {"isAvailable":true,"queued":0,"recentlyDone":0,"concurrent":0,"maxConcurrent":10}
```

## 生产环境部署

生产环境建议配置认证 Token 和资源限制：

```bash
docker run -d \
  --name browserless \
  -p 3000:3000 \
  -e TOKEN=your-secret-token \
  -e MAX_CONCURRENT_SESSIONS=10 \
  -e CONNECTION_TIMEOUT=60000 \
  -e KEEP_ALIVE=true \
  -e ENABLE_DEBUGGER=true \
  --restart unless-stopped \
  browserless/chrome
```

### 关键环境变量说明

| 变量 | 说明 | 默认值 |
|---|---|---|
| `TOKEN` | API 认证 Token，客户端请求需携带 | 无（不认证） |
| `MAX_CONCURRENT_SESSIONS` | 最大并发浏览器会话数 | 10 |
| `CONNECTION_TIMEOUT` | 单个请求超时时间（毫秒） | 30000 |
| `KEEP_ALIVE` | 是否保持浏览器实例存活 | false |
| `ENABLE_DEBUGGER` | 是否启用 Chrome DevTools Protocol | true |

## 配置 Skill 环境变量

部署 Browserless 后，需要在运行 skill 的环境中设置以下环境变量：

### 必填变量

```bash
# Browserless 服务地址
export BROWSERLESS_URL=http://your-server-ip:3000
```

### 可选变量

```bash
# 认证 Token（如果服务配置了 TOKEN）
export BROWSERLESS_TOKEN=your-secret-token

# 最大并发数（用于 skill 内部并发控制）
export BROWSERLESS_MAX_CONCURRENT=10
```

### 不同平台的设置方式

**Linux / macOS（终端）：**
```bash
export BROWSERLESS_URL=http://192.168.1.100:3000
export BROWSERLESS_TOKEN=your-token
```

**Windows PowerShell：**
```powershell
$env:BROWSERLESS_URL = "http://192.168.1.100:3000"
$env:BROWSERLESS_TOKEN = "your-token"
```

**Docker Compose（推荐生产环境）：**
```yaml
version: '3'
services:
  browserless:
    image: browserless/chrome
    ports:
      - "3000:3000"
    environment:
      - TOKEN=your-secret-token
      - MAX_CONCURRENT_SESSIONS=10
    restart: unless-stopped

  # 如果你的 agent 也用 Docker 运行
  agent:
    image: your-agent-image
    environment:
      - BROWSERLESS_URL=http://browserless:3000
      - BROWSERLESS_TOKEN=your-secret-token
    depends_on:
      - browserless
```

## 常见问题

### 1. 连接超时或无法访问

**检查项：**
- 确认 Docker 容器正在运行：`docker ps`
- 确认端口正确暴露：`docker port browserless`
- 如果是远程服务器，确认防火墙允许 3000 端口访问

**解决：**
```bash
# 查看容器日志
docker logs browserless

# 重启容器
docker restart browserless
```

### 2. 请求返回 401 Unauthorized

**原因：** 服务配置了 `TOKEN`，但请求未携带。

**解决：** 设置 `BROWSERLESS_TOKEN` 环境变量。

### 3. 并发请求过多导致队列阻塞

**现象：** `/pressure` 返回 `isAvailable: false` 或 `queued` 较高。

**解决：**
- 增加 `MAX_CONCURRENT_SESSIONS`
- 在 skill 执行时检查 `/pressure` 状态，等待队列清空
- 完成任务后及时关闭会话释放资源

### 4. 内存占用过高

Browserless 会为每个会话启动一个 Chrome 实例，内存消耗较大。

**解决：**
```bash
# 限制容器内存
docker run -d \
  --name browserless \
  -p 3000:3000 \
  -m 4g \
  --memory-swap 4g \
  browserless/chrome
```

### 5. 截图/PDF 生成失败

**常见原因：**
- 页面加载超时（网络慢或页面复杂）
- Chrome 实例崩溃

**解决：**
- 增加 `CONNECTION_TIMEOUT` 值
- 检查目标网站是否可正常访问
- 查看容器日志排查具体错误

## API 端点参考

Browserless 提供的主要 HTTP 端点：

| 端点 | 方法 | 用途 |
|---|---|---|
| `/pressure` | GET | 查看服务负载状态 |
| `/screenshot` | POST | 截取页面截图 |
| `/pdf` | POST | 生成页面 PDF |
| `/evaluate` | POST | 在页面中执行 JavaScript |
| `/session` | POST | 创建持久会话（返回 WebSocket URL） |
| `/content` | POST | 获取页面 HTML 内容 |

详细 API 文档参考：[Browserless 官方文档](https://docs.browserless.io/)

## 安全建议

1. **生产环境务必设置 TOKEN**，避免服务被滥用
2. **不要将服务暴露在公网**，使用内网或 VPN 访问
3. **限制并发和超时**，防止资源耗尽
4. **定期重启容器**，清理可能残留的 Chrome 实例
5. **监控内存使用**，设置 Docker 内存限制