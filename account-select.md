# 架构文档

## API 转发机制与上游选择策略

### 1. API 转发位置

项目采用三层架构进行转发：

#### 接入层
- **`internal/api/modules/amp/proxy.go`** - 使用 `httputil.ReverseProxy` 处理代理
  - `Director` 逻辑 (第67行): 修改请求头（`Authorization`, `X-Api-Key`）、清理敏感参数（`key`, `auth_token`）
  - `ModifyResponse` (第104行): 处理响应，自动探测和解压 gzip 数据

#### 业务逻辑层
- **`sdk/api/handlers/handlers.go`** - `BaseAPIHandler` 是所有模型请求（OpenAI, Claude, Gemini）的核心入口
  - `ExecuteWithAuthManager` - 非流式请求
  - `ExecuteStreamWithAuthManager` - 流式请求
  - 通过 `h.AuthManager.Execute` 将请求分发到下游执行器

#### 执行层
- **`sdk/cliproxy/auth/conductor.go`** - `Manager.executeWithProvider` 发起最终 HTTP 调用

---

### 2. 上游选择策略

#### A. 提供商选择 (Provider Selection)
1. 通过 `getRequestDetails` 将请求的模型名解析为对应的提供商列表（如 `anthropic`, `openai` 等）
2. `Manager.rotateProviders` (第650行): 对多提供商列表进行原子性 **Round-Robin 轮询**，确保并发请求从不同提供商开始尝试

#### B. 凭据/账号选择 (Credential/Auth Selection)

在 `sdk/cliproxy/auth/selector.go` 中实现了两种策略：

| 策略 | 说明 |
|------|------|
| **RoundRobinSelector** (默认) | 在所有可用账号间轮询，平衡流量 |
| **FillFirstSelector** | 优先使用第一个账号直到耗尽配额，再切换下一个。适用于按窗口计费或有固定限制的场景 |

选择流程：
1. `pickNext` 方法过滤掉已禁用、不匹配模型或处于冷却期（Blocked）的账号
2. 通过 `Selector` 接口实现具体的负载均衡策略

#### C. 容错与重试机制 (Error Handling & Retry)

- **多层重试**: 账号执行失败后，在 `tried` map 中记录，调用 `pickNext` 尝试同提供商下的其他可用账号
- **冷却与降级** (`MarkResult` 第794行):
  - `429 (Quota)`: 将账号/模型组合标记为 `QuotaExceeded`，根据 `Retry-After` 或指数退避算法计算恢复时间
  - `401/403`: 触发较长时间挂起（约30分钟）
- **流式回退**: 对于流式请求，如果在发送数据前发生错误，支持 `StreamingBootstrapRetries` 静默重试或切换备份账号

---

### 3. 架构总结

该项目实现了一个**基于状态感知的动态路由系统**：

1. **轮询负载均衡** - Round-Robin 分发请求到多个提供商和账号
2. **实时状态监控** - 根据 HTTP 响应状态码动态调整账号权重和可用性
3. **多层容错重试** - 确保高可用性和配额利用率