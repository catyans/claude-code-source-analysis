# 第5章 工具详解 - AgentTool 与多 Agent

> **作者**: [Wang Yanshu](https://catyans.github.io)

> 本章分析 AgentTool（17个文件）、TeamCreateTool、SendMessageTool 以及 Coordinator 模式和多 Agent 消息路由。

## 1. AgentTool 输入 Schema

```typescript
const AgentInputSchema = z.object({
  // 任务描述（必需）
  description: z.string().describe("对Agent任务的简要描述"),

  // 发送给子Agent的完整提示
  prompt: z.string().describe("详细的任务指令"),

  // 子Agent类型
  subagent_type: z.enum(['research', 'code', 'review', 'custom']).optional(),

  // 模型选择
  model: z.string().optional().describe("覆盖默认模型"),

  // 后台运行
  run_in_background: z.boolean().optional().describe("是否在后台运行"),

  // 命名
  name: z.string().optional().describe("Agent的唯一名称"),

  // 团队名称（Coordinator模式）
  team_name: z.string().optional(),

  // 运行模式
  mode: z.enum(['sync', 'async']).optional(),

  // 隔离级别
  isolation: z.enum(['none', 'worktree', 'fork']).optional(),

  // 工作目录
  cwd: z.string().optional(),
});
```

### 为什么有 subagent_type

`subagent_type` 不仅仅是标签，它影响：
- **research**：只读工具集（Grep、Glob、FileRead、WebSearch），不能修改文件
- **code**：完整工具集，可以编辑文件和执行命令
- **review**：只读 + Diff 工具，用于代码审查
- **custom**：由 `allowedTools` 参数指定

这种分类限制了子 Agent 的能力范围，遵循最小权限原则。

## 2. Agent 生成流程

```
模型调用 AgentTool
    │
    ▼
解析定义（Resolve Definition）
    │ 从 agentDefinitions 中查找或创建
    ▼
检查 MCP 要求
    │ 子Agent是否需要特定MCP服务器
    ▼
装配工具池
    │ 根据 subagent_type 过滤工具
    ▼
生成（Spawn）
    ├─ sync:  在当前进程内运行
    ├─ async: 创建后台任务
    └─ fork:  spawn子进程（FORK_AGENT feature）
    │
    ▼
自动后台化
    │ 超过120秒未完成 → 转为后台任务
    ▼
结果收集
```

### 2.1 定义解析

```typescript
function resolveAgentDefinition(
  input: AgentInput,
  existingDefs: AgentDefinition[]
): AgentDefinition {
  // 如果指定了名称，查找已有定义
  if (input.name) {
    const existing = existingDefs.find(d => d.name === input.name);
    if (existing) return existing;
  }

  // 创建新定义
  return {
    name: input.name || generateUniqueName(),
    type: input.subagent_type || 'custom',
    model: input.model,
    allowedTools: getToolsForType(input.subagent_type),
    systemPrompt: buildAgentSystemPrompt(input),
  };
}
```

### 2.2 MCP 要求检查

```typescript
async function checkMCPRequirements(
  def: AgentDefinition,
  mcpClients: MCPClient[]
): Promise<void> {
  if (def.mcpServerRequirements) {
    for (const serverName of def.mcpServerRequirements) {
      const client = mcpClients.find(c => c.serverName === serverName);
      if (!client) {
        throw new Error(`Agent requires MCP server '${serverName}' which is not available`);
      }
      if (!client.isConnected()) {
        await client.connect();
      }
    }
  }
}
```

## 3. runAgent 生成器

```typescript
async function* runAgent(
  def: AgentDefinition,
  input: AgentInput,
  context: ToolUseContext
): AsyncGenerator<AgentEvent, AgentResult> {
  // 1. MCP服务器初始化
  const mcpServers = await initAgentMCPServers(def, context);

  // 2. 创建子Agent上下文
  const subContext = createSubagentContext(def, context, mcpServers);

  // 3. 触发钩子
  yield* executeHooks('subagent_start', { agent: def, input });

  // 4. 查询循环
  const queryEngine = new QueryEngine(subContext);
  const result = yield* queryEngine.run(input.prompt);

  // 5. 记录transcript
  await recordTranscript(def.name, result.messages);

  // 6. 清理
  await cleanupMCPServers(mcpServers, def);

  return result;
}
```

## 4. Agent MCP 服务器初始化

MCP 服务器有两种引用方式，对应不同的生命周期管理：

### 4.1 按名称引用（共享）

```typescript
// 引用已存在的MCP服务器
{
  mcpServers: [{ ref: "filesystem-server" }]
}
```

按名称引用的服务器与主进程共享同一个 MCP 客户端连接。Agent 退出时**不清理**这个连接，因为其他 Agent 或主进程可能仍在使用。

### 4.2 内联定义（独占）

```typescript
// 定义新的MCP服务器
{
  mcpServers: [{
    inline: {
      command: "npx",
      args: ["-y", "@modelcontextprotocol/server-github"],
      env: { GITHUB_TOKEN: "..." }
    }
  }]
}
```

内联定义的服务器在 Agent 启动时创建，退出时清理。

### 安全措施

```typescript
// O_EXCL: 防止TOCTOU竞态
// O_NOFOLLOW: 防止符号链接遍历攻击
const fd = fs.openSync(configPath, O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW, 0o600);
```

MCP 服务器的配置文件使用原子写入（`O_EXCL` 确保文件不存在时才创建）和符号链接保护（`O_NOFOLLOW` 拒绝打开符号链接），防止攻击者通过预先放置符号链接来劫持配置。

## 5. Fork 子进程隔离

当 `feature('FORK_AGENT')` 启用时，子 Agent 在独立的子进程中运行：

```typescript
if (input.isolation === 'fork' || feature('FORK_AGENT')) {
  const child = fork(AGENT_WORKER_SCRIPT, {
    env: {
      ...process.env,
      AGENT_DEFINITION: JSON.stringify(def),
      AGENT_INPUT: JSON.stringify(input),
    },
    execArgv: ['--max-old-space-size=512'], // 限制V8堆大小
  });

  // 通过IPC通信
  child.on('message', handleAgentMessage);
}
```

### 为什么需要进程隔离

- **内存隔离**：独立的 V8 堆，Agent 的内存泄漏不会影响主进程
- **GC 隔离**：子进程的垃圾回收不会导致主进程卡顿
- **崩溃隔离**：子进程崩溃不影响主进程和其他 Agent
- **资源限制**：可以独立设置内存和 CPU 限制

## 6. 后端检测顺序

当需要在独立环境中运行 Agent 时，系统按以下顺序检测可用后端：

```typescript
function detectAndGetBackend(): AgentBackend {
  // 1. 检查是否启用了进程内模式
  if (isInProcessEnabled()) {
    return new InProcessBackend();
  }

  // 2. 检测iTerm2
  if (isITerm2Available()) {
    return new ITerm2Backend(); // 使用iTerm2的标签页
  }

  // 3. 检测tmux
  if (isTmuxAvailable()) {
    return new TmuxBackend(); // 使用tmux的窗格
  }

  // 4. 回退到进程内
  return new InProcessBackend();
}
```

### 为什么需要多种后端

- **iTerm2 标签页**：macOS 用户最佳体验，每个 Agent 一个标签页
- **tmux 窗格**：跨平台，支持 SSH 远程
- **进程内**：兜底方案，不需要外部工具

## 7. Coordinator 模式

### 7.1 模式检测

```typescript
function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE') &&
    getAppState().permissionMode === 'coordinator';
}
```

### 7.2 Coordinator 专用工具

Coordinator 模式下，模型只能使用以下内部工具：

```typescript
const INTERNAL_WORKER_TOOLS = [
  'TeamCreate',       // 创建Agent团队
  'TeamDelete',       // 删除Agent团队
  'SendMessage',      // 向Agent发送消息
  'SyntheticOutput',  // 合成输出
];
```

Coordinator **不能直接执行** Bash、FileEdit 等操作工具——它只能通过创建 Worker 来间接完成工作。

### 7.3 系统提示（370行）

Coordinator 的系统提示定义了四个工作阶段：

```
Phase 1: Research（调研）
  → 创建research类型的Agent
  → 收集项目信息、代码结构、依赖关系

Phase 2: Synthesis（综合）
  → 分析调研结果
  → 制定实施计划
  → 分配任务

Phase 3: Implementation（实施）
  → 创建code类型的Agent
  → 分配具体编码任务
  → 监控进度

Phase 4: Verification（验证）
  → 创建review类型的Agent
  → 代码审查
  → 运行测试
```

### 为什么强制四阶段

- **防止信息不完整的行动**：先调研再行动，避免 Agent 基于假设编写代码
- **综合优先**：Worker 收集的信息需要 Coordinator 综合判断，而非各自为政
- **分离关注点**：调研、编码、审查由不同的 Agent 执行，避免角色混乱

## 8. 团队文件生命周期

### 8.1 创建

```typescript
async function createTeam(name: string, leaderAgent: string): Promise<TeamFile> {
  // 生成唯一名称
  const teamName = generateUniqueTeamName(name);

  // 创建leader条目
  const leader: TeamMember = {
    name: leaderAgent,
    role: 'leader',
    status: 'active',
  };

  // 写入团队文件
  const teamFile: TeamFile = {
    name: teamName,
    members: [leader],
    tasks: [],
    mailbox: [],
    createdAt: Date.now(),
  };

  await writeTeamFile(teamName, teamFile);

  // 注册会话清理
  registerSessionCleanup(() => deleteTeamFile(teamName));

  return teamFile;
}
```

### 8.2 任务列表

```typescript
interface TeamTask {
  id: string;
  assignee: string;     // Agent名称
  description: string;
  status: 'pending' | 'in_progress' | 'completed' | 'failed';
  result?: string;
}
```

## 9. 跨 Agent 消息路由

消息路由按优先级尝试多种传输方式：

```
发送消息(from: A, to: B)
    │
    ├─ 1. 本地进程内 → A和B在同一进程
    │     直接调用B的消息处理函数
    │
    ├─ 2. UDS Socket → A和B在同一机器的不同进程
    │     通过Unix Domain Socket发送
    │
    ├─ 3. Bridge远程 → B在远程机器
    │     通过Bridge WebSocket转发
    │
    ├─ 4. Team文件邮箱 → 异步通信
    │     写入团队文件的mailbox字段
    │
    └─ 5. 广播("*") → 发送给所有Agent
          遍历所有传输方式
```

### 9.1 本地进程内路由

```typescript
if (isInProcess(targetAgent)) {
  const handler = agentRegistry.get(targetAgent);
  await handler.onMessage(message);
  return;
}
```

### 9.2 UDS Socket 路由

```typescript
if (feature('UDS_INBOX')) {
  const socketPath = getAgentSocketPath(targetAgent);
  if (fs.existsSync(socketPath)) {
    const client = net.createConnection(socketPath);
    client.write(JSON.stringify(message));
    return;
  }
}
```

### 9.3 Team 文件邮箱

```typescript
// 最终回退：写入文件
async function sendViaMailbox(teamName: string, to: string, message: AgentMessage) {
  const teamFile = await readTeamFile(teamName);
  teamFile.mailbox.push({
    from: message.from,
    to,
    content: message.content,
    timestamp: Date.now(),
  });
  await writeTeamFile(teamName, teamFile);
}
```

### 为什么用多种传输方式

不同场景需要不同的通信机制：
- **进程内**：最低延迟，同步调用
- **UDS**：跨进程但同机器，低延迟，可靠
- **Bridge**：跨机器，依赖网络
- **文件邮箱**：异步兜底，不需要双方同时在线

## 10. 结构化消息

```typescript
type StructuredMessage =
  | { type: 'shutdown_request'; reason: string }
  | { type: 'shutdown_response'; acknowledged: boolean }
  | { type: 'plan_approval_response'; approved: boolean; feedback?: string }
  | { type: 'task_update'; taskId: string; status: string; result?: string }
  | { type: 'text'; content: string };
```

结构化消息使得系统级操作（如关闭请求）与用户级操作（如文本消息）可以通过同一通道传输，由消息类型区分处理方式。

## 11. Scratchpad 共享

```typescript
// tengu_scratch gate 控制的共享便签
if (checkGate('tengu_scratch')) {
  const scratchPath = getScratchpadPath(teamName);

  // 所有worker可以无权限提示地读写
  const scratchContent = await readFile(scratchPath);
  await writeFile(scratchPath, updatedContent);
}
```

Scratchpad 是团队内所有 Worker 共享的临时文件，用于：
- 共享调研结果
- 记录中间状态
- 协调工作进展

### 为什么无需权限提示

Scratchpad 是团队的"白板"——在创建团队时隐含了对共享空间的授权。频繁的权限提示会严重拖慢多 Agent 协作的效率。

## 12. 自动后台化

```typescript
const AUTO_BACKGROUND_TIMEOUT = 120_000; // 120秒

async function* executeWithAutoBackground(
  agentGenerator: AsyncGenerator<AgentEvent, AgentResult>,
  taskManager: TaskManager
): AsyncGenerator<AgentEvent, AgentResult> {
  const timer = setTimeout(() => {
    // 将Agent转为后台任务
    const task = taskManager.createTask({
      type: 'LocalAgent',
      name: agent.name,
      generator: agentGenerator,
    });
    // 通知UI
    yield { type: 'backgrounded', taskId: task.id };
  }, AUTO_BACKGROUND_TIMEOUT);

  try {
    return yield* agentGenerator;
  } finally {
    clearTimeout(timer);
  }
}
```

### 为什么120秒

- 大多数简单任务在 120 秒内完成（文件搜索、小修改）
- 超过 120 秒的任务通常是复杂操作（大规模重构、多文件修改），用户不应该干等
- 后台化后用户可以继续使用 REPL 进行其他工作

## 13. 设计决策总结

| 决策 | 原因 |
|------|------|
| 子Agent类型限制工具集 | 最小权限原则 |
| MCP共享vs独占 | 避免重复连接vs防止资源泄漏 |
| O_EXCL + O_NOFOLLOW | 防止TOCTOU和符号链接攻击 |
| Fork进程隔离 | 内存/GC/崩溃隔离 |
| Coordinator四阶段 | 防止信息不完整的行动 |
| 多传输方式 | 覆盖不同的部署拓扑 |
| 文件邮箱兜底 | 不需要双方同时在线 |
| 自动后台化120s | 避免用户干等长任务 |

相关章节：
- [[03 工具系统]]：AgentTool 的注册和执行
- [[07 权限系统]]：Coordinator/SwarmWorker 的权限处理
- [[09 服务层]]：MCP 客户端的生命周期
- [[08 状态管理]]：Agent 相关的 AppState 字段
