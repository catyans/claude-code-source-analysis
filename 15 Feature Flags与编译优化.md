# 第15章 Feature Flags 与编译优化

> **作者**: [Wang Yanshu](https://catyans.github.io)

> 本章分析 Claude Code 的构建时 DCE（Dead Code Elimination）、运行时 Feature Gate 和配置迁移系统。

## 1. 构建时 DCE（Dead Code Elimination）

### 1.1 feature() 宏

```typescript
// bun:bundle 提供的编译时函数
import { feature } from 'bun:bundle';

// 编译时求值
if (feature('COORDINATOR_MODE')) {
  const { CoordinatorTools } = require('./coordinator/tools');
  registerTools(CoordinatorTools);
}
```

`feature()` 在 Bun 的 bundler 阶段被替换为 `true` 或 `false` 字面量。当替换为 `false` 时，整个 `if` 块在后续的 tree-shaking 阶段被移除，包括 `require()` 的模块也不会被打包。

### 1.2 三元 require 模式

```typescript
// 编译时条件require
const CoordinatorModule = feature('COORDINATOR_MODE')
  ? require('./coordinator')
  : null;

// 运行时安全使用
function maybeCreateCoordinator() {
  if (CoordinatorModule) {
    return new CoordinatorModule.Coordinator();
  }
  return null;
}
```

这种模式比 `if` 块更适合模块级变量——三元表达式可以赋值给 `const`，而 `if` 块中的 `require` 需要 `let` 或额外的函数封装。

### 1.3 DCE 后的代码

```typescript
// 源码
if (feature('KAIROS')) {
  const { KairosAssistant } = require('./kairos');
  registerAssistant(KairosAssistant);
}

// 外部构建后（KAIROS=false）
if (false) {
  // 整个块被移除
}

// 最终产物：完全没有这段代码
```

## 2. 已知构建时标志

| 标志 | 说明 | 外部构建值 |
|------|------|-----------|
| `PROACTIVE` | 主动建议系统 | false |
| `KAIROS` | Anthropic 内部助手模式 | false |
| `BRIDGE_MODE` | Bridge IDE 集成 | true |
| `DAEMON` | 守护进程模式 | false |
| `VOICE_MODE` | 语音输入 | false |
| `AGENT_TRIGGERS` | Agent 触发器 | false |
| `MONITOR_TOOL` | 系统监控工具 | false |
| `COORDINATOR_MODE` | 多 Agent 协调器 | false |
| `WEB_BROWSER_TOOL` | 浏览器自动化工具 | false |
| `TEAMMEM` | 团队记忆共享 | false |
| `DIRECT_CONNECT` | DirectConnect 服务器 | false |
| `SSH_REMOTE` | SSH 远程模式 | false |
| `CHICAGO_MCP` | 芝加哥 MCP 扩展 | false |
| `TEMPLATES` | 模板系统 | false |
| `UPLOAD_USER_SETTINGS` | 上传用户设置遥测 | true |
| `FORK_AGENT` | Fork 进程隔离 Agent | false |
| `UDS_INBOX` | Unix Domain Socket 消息箱 | false |

### 为什么大部分标志在外部构建中为 false

这些功能要么是：
- **Anthropic 内部功能**（KAIROS, PROACTIVE）：不应出现在公开版本中
- **实验性功能**（FORK_AGENT, UDS_INBOX）：尚未稳定
- **平台特定功能**（VOICE_MODE）：需要额外权限和服务支持
- **企业功能**（COORDINATOR_MODE, TEAMMEM）：仅在特定部署中启用

## 3. 运行时 Feature Gate（GrowthBook）

### 3.1 Gate 列表

| Gate 名称 | 说明 |
|----------|------|
| `tengu_ccr_bridge` | CCR Bridge v2 启用 |
| `tengu_bridge_repl_v2` | Bridge REPL v2 |
| `tengu_ccr_bridge_multi_session` | Bridge 多会话 |
| `tengu_scratch` | Scratchpad 共享 |
| `tengu_coral_fern` | 未知功能（内部代号） |
| `tengu_amber_quartz_disabled` | 语音输入禁用开关 |
| `tengu_otk_slot_v1` | OTK Slot v1 |
| `tengu_satin_quoll` | 未知功能（内部代号） |
| `tengu_disable_streaming_to_non_streaming_fallback` | 禁用流式到非流式回退 |

### 3.2 两种检查模式

#### getFeatureValue_CACHED_MAY_BE_STALE（非阻塞）

```typescript
// 从本地缓存读取，可能是旧值
const value = getFeatureValue_CACHED_MAY_BE_STALE('tengu_scratch', false);
```

**使用场景**：UI 渲染、非关键路径。使用缓存值避免阻塞渲染。如果值过期了，下次刷新会更新。

#### checkGate_CACHED_OR_BLOCKING（可阻塞）

```typescript
// 优先使用缓存，缓存过期则阻塞等待刷新
const isEnabled = await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge');
```

**使用场景**：安全关键路径。如果 gate 控制的是安全功能（如权限绕过），必须使用最新值。

### 为什么有两种模式

- **CACHED_MAY_BE_STALE**：对于 UI 功能（如是否显示某个面板），使用几秒前的值完全可以接受，但阻塞 UI 渲染不可接受
- **CACHED_OR_BLOCKING**：对于安全功能（如是否允许某种操作），使用过期值可能导致安全漏洞，宁可等待几百毫秒

## 4. 配置迁移

### 4.1 迁移文件

```typescript
// migrations/001_fennec_to_opus.ts
export const migration = {
  id: '001_fennec_to_opus',
  description: '将模型名 Fennec 更新为 Opus',

  shouldRun(settings: Settings): boolean {
    return settings.model?.includes('fennec') ?? false;
  },

  run(settings: Settings): Settings {
    return {
      ...settings,
      model: settings.model!.replace('fennec', 'opus'),
    };
  }
};
```

### 4.2 12个迁移追踪模型演进

| 迁移 ID | 说明 |
|---------|------|
| 001 | Fennec → Opus |
| 002 | Sonnet 4.5 → Sonnet 4.6 |
| 003 | Opus → Opus 1M |
| 004 | 旧权限格式 → 新格式 |
| 005 | MCP 配置格式更新 |
| 006 | 主题配置迁移 |
| 007 | 快捷键配置迁移 |
| 008 | 模型别名更新 |
| 009 | API端点迁移 |
| 010 | 插件配置格式更新 |
| 011 | 记忆目录迁移 |
| 012 | 会话格式版本更新 |

### 4.3 迁移执行

```typescript
async function runMigrations(): Promise<void> {
  const settings = await loadSettings();
  const appliedMigrations = await getAppliedMigrations();

  for (const migration of ALL_MIGRATIONS) {
    // 跳过已应用的
    if (appliedMigrations.has(migration.id)) continue;

    // 检查是否需要运行
    if (!migration.shouldRun(settings)) {
      await markMigrationApplied(migration.id);
      continue;
    }

    // 运行迁移
    const updatedSettings = migration.run(settings);
    await saveSettings(updatedSettings);
    await markMigrationApplied(migration.id);

    logger.info(`Applied migration: ${migration.id}`);
  }
}
```

### 为什么需要迁移系统

Claude Code 频繁更新模型名称（跟随 Anthropic 的模型发布周期）。没有迁移系统的话：
- 用户的 settings.json 中会留有过时的模型名称
- 手动更新容易出错且容易遗忘
- 企业部署中可能有大量需要统一更新的配置

## 5. 部署变体

### 5.1 四种变体

| 变体 | 目标用户 | 构建特点 |
|------|---------|---------|
| External | 公开用户 | 大多数feature=false，最小体积 |
| Ant-internal | Anthropic员工 | 所有feature=true |
| Assistant | KAIROS助手 | KAIROS=true，其他选择性 |
| Bridge | IDE集成 | BRIDGE_MODE=true |

### 5.2 构建流程

```typescript
// build.ts
function buildVariant(variant: BuildVariant) {
  const features = getFeatureFlags(variant);

  return Bun.build({
    entrypoints: ['./src/entrypoints/main.tsx'],
    define: {
      // 将feature flags注入为编译时常量
      ...Object.fromEntries(
        Object.entries(features).map(([k, v]) => [
          `feature('${k}')`,
          String(v),
        ])
      ),
    },
    minify: variant === 'external',
    sourcemap: variant === 'external' ? 'external' : 'inline',
  });
}
```

### 5.3 为什么用单代码库而非多仓库

| 方案 | 优点 | 缺点 |
|------|------|------|
| 多仓库 | 完全隔离 | 代码重复、同步困难 |
| 单仓库+分支 | 独立演进 | 合并冲突、特性传播慢 |
| 单仓库+DCE | 共享核心、精确控制 | 构建复杂度 |

Claude Code 选择了单仓库+DCE，因为：
- 核心逻辑（QueryEngine、工具系统、权限系统）在所有变体间共享
- Feature flags 明确标记了变体差异，便于审查
- DCE 保证外部构建不包含内部代码（安全性）
- 修复一个 bug 自动传播到所有变体

## 6. Feature Flag 的生命周期

```
1. 开发者添加新功能
   ├─ 添加 feature('NEW_FEATURE') 守卫
   └─ 默认 false（不影响现有构建）

2. 内部测试
   └─ Ant-internal 构建中 NEW_FEATURE=true

3. A/B 测试
   └─ 添加 GrowthBook gate，对部分用户启用

4. 全量发布
   └─ External 构建中 NEW_FEATURE=true

5. 清理
   └─ 移除 feature() 守卫，代码成为默认路径
```

### 为什么需要清理阶段

长期保留的 feature flags 会导致：
- 代码理解困难（需要追踪哪些 flag 在哪些构建中启用）
- 测试矩阵爆炸（2^N 种 flag 组合）
- 死代码积累（永远不会启用的旧 flag）

## 7. 设计决策总结

| 决策 | 原因 |
|------|------|
| 构建时DCE | 外部构建不包含内部代码 |
| 运行时GrowthBook | A/B测试和kill-switch |
| 两种gate检查模式 | 平衡一致性和性能 |
| 12个配置迁移 | 追踪模型命名变化 |
| 单代码库+DCE | 共享核心，精确控制变体差异 |
| Feature Flag生命周期 | 从实验到全量到清理的完整流程 |

相关章节：
- [[01 启动流程]]：feature flags 影响启动路径
- [[03 工具系统]]：工具的条件注册
- [[09 服务层]]：GrowthBook 服务初始化
- [[05 工具详解-AgentTool与多Agent]]：Coordinator 的 feature gate
