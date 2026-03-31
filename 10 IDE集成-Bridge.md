# 第10章 IDE集成 - Bridge

> **作者**: [Wang Yanshu](https://catyans.github.io)

> 本章分析 bridge/ 目录（31个文件），涵盖 v1/v2 双传输路径、JWT 刷新调度、工作轮询协议和多会话管理。

## 1. 双传输路径

Bridge 系统同时支持 v1 和 v2 两个版本的传输协议，用于与 IDE（VSCode、JetBrains）通信。

### 1.1 V1：HybridTransport（WS 读 + POST 写）

```typescript
class V1HybridTransport implements BridgeTransport {
  private ws: WebSocket;               // 读取通道
  private httpClient: AxiosInstance;    // 写入通道

  async connect(environmentId: string): Promise<void> {
    // WebSocket 用于接收实时事件
    this.ws = new WebSocket(`${WS_URL}/v1/environments/${environmentId}/events`);
    this.ws.on('message', this.handleEvent.bind(this));

    // HTTP POST 用于发送响应和状态更新
    this.httpClient = axios.create({
      baseURL: `${HTTP_URL}/v1/environments/${environmentId}`,
    });
  }

  // 读取：通过WebSocket
  onEvent(handler: (event: BridgeEvent) => void): void {
    this.eventHandlers.push(handler);
  }

  // 写入：通过HTTP POST
  async send(data: BridgeMessage): Promise<void> {
    await this.httpClient.post('/messages', data);
  }
}
```

V1 使用 Environments API 模型：每个 Claude Code 实例注册为一个"环境"，IDE 通过环境 ID 与之通信。

### 1.2 V2：SSE 读 + CCR 写

```typescript
class V2Transport implements BridgeTransport {
  private eventSource: EventSource;     // 读取通道（SSE）
  private ccrClient: CCRClient;         // 写入通道

  async connect(sessionToken: string): Promise<void> {
    // SSE 用于接收服务器推送事件
    this.eventSource = new EventSource(`${SSE_URL}/v2/sessions/events`, {
      headers: { Authorization: `Bearer ${sessionToken}` },
    });

    // CCR (Claude Code Remote) 客户端用于写入
    this.ccrClient = new CCRClient({
      baseUrl: CCR_URL,
      sessionToken,
    });
  }

  onEvent(handler: (event: BridgeEvent) => void): void {
    this.eventSource.onmessage = (msg) => {
      handler(JSON.parse(msg.data));
    };
  }

  async send(data: BridgeMessage): Promise<void> {
    await this.ccrClient.send(data);
  }
}
```

V2 移除了 Environment 概念，直接使用 OAuth session token 进行无状态通信。

### 为什么同时支持 V1 和 V2

- **渐进迁移**：claude.ai 应用中的 session-list 查询正在逐步切换到 v2
- **向后兼容**：旧版 IDE 扩展仍使用 v1
- **回退策略**：v2 连接失败时可以回退到 v1

## 2. JWT 刷新调度

### 2.1 刷新时机计算

```typescript
function scheduleJwtRefresh(token: string): void {
  // 解码JWT payload（不验证签名，只读取expiry）
  const payload = decodeJwtPayload(token);
  const expiryMs = payload.exp * 1000;
  const now = Date.now();

  // 在过期前5分钟刷新
  const refreshAt = expiryMs - 5 * 60 * 1000;
  const delay = Math.max(0, refreshAt - now);

  setTimeout(() => refreshToken(), delay);
}
```

### 2.2 Generation Counter（代次计数器）

```typescript
class TokenRefreshScheduler {
  private generation = 0;  // 当前代次

  async refreshToken(): Promise<void> {
    const myGeneration = ++this.generation;

    try {
      const newToken = await fetchNewToken();

      // 检查是否被更新的刷新取代
      if (myGeneration !== this.generation) {
        // 另一个刷新已经启动，丢弃这个结果
        return;
      }

      applyToken(newToken);
      scheduleJwtRefresh(newToken);
    } catch (error) {
      if (myGeneration !== this.generation) return;

      this.failureCount++;
      if (this.failureCount < 3) {
        // 60秒后重试
        setTimeout(() => this.refreshToken(), 60_000);
      } else {
        // 30分钟后最终重试
        setTimeout(() => this.refreshToken(), 30 * 60_000);
      }
    }
  }
}
```

### 为什么需要 Generation Counter

在异步环境中，旧的刷新请求可能在新的刷新请求发起后才返回。例如：

```
T=0:  refresh A 发起 (generation=1)
T=1:  token过期，refresh B 发起 (generation=2)
T=2:  refresh B 返回新token
T=3:  refresh A 返回（更旧的token） ← 如果不检查generation，会覆盖更新的token
```

通过比较 `myGeneration` 和 `this.generation`，过时的异步结果会被丢弃。

### 2.3 失败重试策略

| 失败次数 | 行为 |
|---------|------|
| 1 | 60 秒后重试 |
| 2 | 60 秒后重试 |
| 3+ | 30 分钟后最终重试 |

渐进退让避免了在服务不可用时频繁请求。30 分钟的最终重试足够让临时的服务中断恢复。

## 3. 工作轮询协议

### 3.1 轮询请求

```typescript
async function pollForWork(environmentId: string): Promise<WorkItem | null> {
  const response = await httpClient.get(
    `/v1/environments/${environmentId}/work/poll`,
    { timeout: LONG_POLL_TIMEOUT }
  );

  if (response.status === 204) return null; // 无工作
  return response.data as WorkItem;
}
```

### 3.2 容量依赖的轮询间隔

```typescript
function getPollingInterval(capacity: CapacityStatus): number {
  switch (capacity) {
    case 'not-at-capacity':
      return 1000;   // 1秒：有空闲，积极轮询
    case 'partial':
      return 5000;   // 5秒：部分忙碌
    case 'at-capacity':
      return 30000;  // 30秒：满载，减少轮询
  }
}
```

### 为什么容量影响轮询频率

- **空闲时快速轮询**：用户等待响应时，希望尽快获取工作
- **满载时慢速轮询**：所有会话都在工作时，频繁轮询浪费服务器资源
- **部分忙碌折中**：在响应速度和服务器负载之间平衡

### 3.3 空轮询计数

```typescript
class WorkPoller {
  private emptyPollCount = 0;

  async poll(): Promise<void> {
    const work = await pollForWork(this.environmentId);

    if (!work) {
      this.emptyPollCount++;

      // 连续空轮询后逐渐降低频率
      if (this.emptyPollCount > 10) {
        this.interval = Math.min(this.interval * 1.5, MAX_POLL_INTERVAL);
      }
    } else {
      this.emptyPollCount = 0;
      this.interval = getPollingInterval(this.capacity);
      await this.processWork(work);
    }
  }
}
```

### 3.4 可回收工作（Reclaimable Work）

```typescript
async function checkReclaimableWork(): Promise<WorkItem[]> {
  // 查找因Worker崩溃而未完成的工作
  const reclaimable = await httpClient.get(
    `/v1/environments/${this.environmentId}/work/reclaimable`
  );
  return reclaimable.data;
}
```

当一个 Worker 进程崩溃时，它正在处理的工作不会丢失——下一个轮询的 Worker 可以"回收"这些未完成的工作。

## 4. 确认与心跳

### 4.1 工作确认（Acknowledgement）

```typescript
async function acknowledgeWork(workId: string, sessionToken: string): Promise<void> {
  await httpClient.post(
    `/v1/environments/${environmentId}/work/${workId}/ack`,
    { sessionToken }
  );
}
```

确认告诉服务器"我已收到这个工作，开始处理"。如果 Worker 在确认前崩溃，工作会被其他 Worker 回收。

### 4.2 心跳（Heartbeat）

```typescript
class WorkHeartbeat {
  private intervalId: NodeJS.Timer;

  start(workId: string): void {
    this.intervalId = setInterval(async () => {
      try {
        await httpClient.post(
          `/v1/environments/${environmentId}/work/${workId}/heartbeat`
        );
      } catch {
        // 心跳失败不致命，服务器有宽限期
      }
    }, HEARTBEAT_INTERVAL);
  }

  stop(): void {
    clearInterval(this.intervalId);
  }
}
```

### 为什么心跳和轮询分离

- **轮询**是获取新工作，涉及复杂的响应解析
- **心跳**是续租当前工作的"租约"，只需确认存活
- 分离后，心跳可以更频繁（保持租约），轮询可以更智能（基于容量）

## 5. 多会话管理

### 5.1 maxSessions 容量控制

```typescript
class SessionManager {
  private sessions = new Map<string, Session>();
  private readonly maxSessions: number;

  canAcceptWork(): boolean {
    return this.sessions.size < this.maxSessions;
  }

  getCapacityStatus(): CapacityStatus {
    if (this.sessions.size === 0) return 'not-at-capacity';
    if (this.sessions.size < this.maxSessions) return 'partial';
    return 'at-capacity';
  }
}
```

### 5.2 Worktree 隔离

每个 Bridge 会话使用独立的 Git Worktree，确保并发会话的文件修改不会冲突：

```typescript
async function createSessionWorktree(sessionId: string): Promise<string> {
  const branchName = `claude-session-${sessionId}`;
  const worktreePath = path.join(GIT_WORKTREE_DIR, sessionId);

  await exec(`git worktree add -b ${branchName} ${worktreePath}`);

  return worktreePath;
}

async function cleanupSessionWorktree(sessionId: string): Promise<void> {
  const worktreePath = path.join(GIT_WORKTREE_DIR, sessionId);
  await exec(`git worktree remove ${worktreePath}`);
}
```

### 5.3 Work-Stealing（工作窃取）

```typescript
// Worker可以"窃取"其他已崩溃Worker的工作
async function stealWork(reclaimableWorkId: string): Promise<WorkItem> {
  const response = await httpClient.post(
    `/v1/environments/${environmentId}/work/${reclaimableWorkId}/steal`
  );
  return response.data;
}
```

## 6. 会话生命周期

```
1. 注册环境
   POST /v1/environments
   → environmentId

2. 轮询工作
   GET /v1/environments/{id}/work/poll
   → workItem

3. 确认工作
   POST /v1/environments/{id}/work/{workId}/ack
   → sessionToken

4. 心跳循环
   POST /v1/environments/{id}/work/{workId}/heartbeat
   (每N秒)

5. 处理工作
   创建会话 → 执行查询 → 返回结果

6. 完成/取消
   POST /v1/environments/{id}/work/{workId}/complete
   或
   POST /v1/environments/{id}/work/{workId}/cancel

7. 注销环境
   DELETE /v1/environments/{id}
```

## 7. 权限流转

### 7.1 Bridge 权限回调

```typescript
interface BridgePermissionCallbacks {
  // 从Claude Code发给IDE的权限请求
  requestPermission(request: PermissionRequest): void;

  // 从IDE发给Claude Code的权限响应
  onPermissionResponse(callback: (response: PermissionResponse) => void): void;
}
```

### 7.2 control_response 事件

```typescript
// IDE发送的控制响应
interface ControlResponseEvent {
  type: 'control_response';
  requestId: string;
  decision: 'allow' | 'deny';
  rememberForSession?: boolean;
}
```

当 Claude Code 需要权限时，通过 Bridge 发送请求给 IDE，IDE 显示对话框让用户决策，然后通过 `control_response` 事件返回结果。

## 8. Session Runner

### 8.1 createSessionSpawner

```typescript
function createSessionSpawner(config: SpawnerConfig) {
  return {
    async spawn(workItem: WorkItem): Promise<ChildProcess> {
      const worktreePath = await createSessionWorktree(workItem.sessionId);

      const child = fork(SESSION_WORKER_SCRIPT, {
        cwd: worktreePath,
        env: {
          ...process.env,
          CCR_V2: config.useV2 ? '1' : '0',
          WORKER_EPOCH: String(config.epoch),
          SESSION_ID: workItem.sessionId,
          BRIDGE_SESSION_TOKEN: workItem.sessionToken,
        },
      });

      // 注册清理
      child.on('exit', async () => {
        await cleanupSessionWorktree(workItem.sessionId);
      });

      return child;
    }
  };
}
```

### 8.2 环境变量传递

| 变量 | 说明 |
|------|------|
| `CCR_V2` | 是否使用 v2 协议 |
| `WORKER_EPOCH` | Worker 代次，用于识别旧进程 |
| `SESSION_ID` | 会话 ID |
| `BRIDGE_SESSION_TOKEN` | 会话认证 token |

### 8.3 Session ID 映射

```typescript
// 内部session ID到Bridge work ID的映射
const sessionWorkMap = new Map<string, string>();

function mapSession(sessionId: string, workId: string): void {
  sessionWorkMap.set(sessionId, workId);
}

function getWorkId(sessionId: string): string | undefined {
  return sessionWorkMap.get(sessionId);
}
```

## 9. 设计决策总结

| 决策 | 原因 |
|------|------|
| V1/V2 共存 | 渐进迁移，向后兼容 |
| JWT generation counter | 防止旧的异步刷新覆盖新token |
| 容量依赖轮询 | 空闲时快速响应，满载时省资源 |
| 心跳与轮询分离 | 不同频率，不同职责 |
| Worktree隔离 | 防止并发会话的文件冲突 |
| Work-stealing | 崩溃恢复，不丢失工作 |
| 子进程隔离 | 会话崩溃不影响主Bridge进程 |

相关章节：
- [[01 启动流程]]：Bridge 模式的启动路径
- [[07 权限系统]]：Bridge 权限回调在权限管道中的角色
- [[09 服务层]]：OAuth 认证支持 Bridge JWT
- [[05 工具详解-AgentTool与多Agent]]：多Agent 使用 Worktree 隔离
