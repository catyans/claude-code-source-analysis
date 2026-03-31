# 第12章 Skill 与插件系统

> **作者**: [Wang Yanshu](https://catyans.github.io)

> 本章分析 skills/（20个文件）和 plugins/ 目录，涵盖捆绑 Skill 注册、惰性文件提取、用户 Skill 加载和插件生命周期。

## 1. 捆绑 Skill 注册

### 1.1 BundledSkillDefinition 类型

```typescript
interface BundledSkillDefinition {
  // 基本信息
  name: string;                    // Skill名称（如 "commit"）
  description: string;             // 功能描述
  aliases: string[];               // 别名列表（如 ["cm", "git-commit"]）

  // 触发条件
  whenToUse: string;               // 何时使用的自然语言描述

  // 能力约束
  allowedTools: string[];          // 允许使用的工具列表
  model?: string;                  // 指定模型（覆盖默认）

  // 生命周期
  hooks?: SkillHooks;              // 前/后置钩子
  context?: SkillContext;          // 额外上下文

  // Agent配置
  agent?: AgentConfig;             // 如果Skill需要子Agent

  // 文件依赖
  files?: Record<string, string>;  // 嵌入的文件内容

  // 提示生成
  getPromptForCommand: (args: string) => string; // 生成执行提示
}
```

### 1.2 registerBundledSkill

```typescript
const bundledSkills: BundledSkillDefinition[] = [];

function registerBundledSkill(skill: BundledSkillDefinition): void {
  // 验证唯一性
  if (bundledSkills.some(s => s.name === skill.name)) {
    throw new Error(`Duplicate skill name: ${skill.name}`);
  }

  bundledSkills.push(skill);
}
```

## 2. 惰性文件提取

### 2.1 闭包局部记忆化

```typescript
function createLazyExtractor(files: Record<string, string>) {
  let extracted = false;  // 闭包局部变量
  let extractedPath: string | null = null;

  return async function extract(): Promise<string> {
    if (extracted && extractedPath) {
      return extractedPath;
    }

    extractedPath = await extractBundledSkillFiles(files);
    extracted = true;
    return extractedPath;
  };
}
```

### 2.2 安全的文件提取

```typescript
async function extractBundledSkillFiles(
  files: Record<string, string>
): Promise<string> {
  // 生成per-process唯一路径
  const nonce = crypto.randomBytes(8).toString('hex');
  const extractDir = path.join(SKILL_EXTRACT_DIR, `skill-${process.pid}-${nonce}`);

  // 创建目录
  await fs.mkdir(extractDir, { mode: 0o700, recursive: true });

  for (const [filename, content] of Object.entries(files)) {
    const filePath = path.join(extractDir, filename);

    // 原子写入：O_EXCL确保文件不存在
    // O_NOFOLLOW防止符号链接遍历
    const fd = await fs.open(filePath, O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW);
    await fd.writeFile(content);
    await fd.chmod(0o700);
    await fd.close();
  }

  return extractDir;
}
```

### 2.3 安全措施详解

#### O_EXCL（排他创建）

```
攻击场景（无O_EXCL）：
1. Claude Code检查文件是否存在 → 不存在
2. 攻击者创建同名符号链接 → /tmp/skill-123/file → /etc/passwd
3. Claude Code写入文件 → 实际写入了 /etc/passwd

防御（有O_EXCL）：
1. Claude Code使用O_EXCL打开文件
2. 如果文件已存在（包括符号链接），open()失败
3. 攻击者无法预先放置文件
```

#### O_NOFOLLOW（拒绝符号链接）

即使文件是新创建的，如果路径中的某个目录是符号链接，`O_NOFOLLOW` 也会拒绝打开。双重保护。

#### Per-Process Nonce

```typescript
const nonce = crypto.randomBytes(8).toString('hex');
```

即使攻击者知道进程 PID，也无法预测 nonce 值，因此无法提前创建匹配的目录或文件。

### 为什么需要惰性提取

大多数 Skill 在一个会话中不会被调用。预先提取所有 Skill 文件：
- 增加启动时间（磁盘 I/O）
- 浪费临时存储空间
- 增加清理负担

惰性提取确保只有实际使用的 Skill 才被解压。

## 3. 用户 Skill 加载

### 3.1 扫描目录

```typescript
async function loadUserSkills(): Promise<UserSkill[]> {
  const skillsDir = path.join(os.homedir(), '.claude', 'skills');

  if (!await fs.pathExists(skillsDir)) return [];

  const entries = await fs.readdir(skillsDir);
  const skills: UserSkill[] = [];

  for (const entry of entries) {
    if (!entry.endsWith('.md')) continue;

    const filePath = path.join(skillsDir, entry);
    const content = await fs.readFile(filePath, 'utf-8');
    const meta = parseFrontmatter(content);

    skills.push({
      name: meta.name || path.basename(entry, '.md'),
      description: meta.description || '',
      allowedTools: meta.allowedTools || [],
      model: meta.model,
      whenToUse: meta.when_to_use || '',
      promptContent: extractBody(content),
      source: 'user',
    });
  }

  return skills;
}
```

### 3.2 Frontmatter 格式

```markdown
---
name: "自定义部署"
description: "使用公司内部工具进行部署"
allowedTools:
  - Bash
  - FileRead
model: claude-sonnet-4-6
when_to_use: "当用户要求部署或发布时"
---

# 部署流程

1. 运行 `./scripts/pre-deploy.sh` 检查前置条件
2. 执行 `deploy-tool push --env production`
3. 验证部署状态 `deploy-tool status`
4. 如果失败，执行 `deploy-tool rollback`
```

### 3.3 包装为 Command

```typescript
function wrapSkillAsCommand(skill: UserSkill): Command {
  return {
    type: 'prompt',
    name: skill.name,
    description: skill.description,
    allowedTools: skill.allowedTools,
    progressMessage: `运行 ${skill.name}...`,
    isEnabled: () => true,
    availability: 'all',
    getPromptForCommand: (args: string) => {
      return `${skill.promptContent}\n\n用户输入: ${args}`;
    },
  };
}
```

## 4. 内置 Skill 目录

| Skill | 功能 | 典型触发 |
|-------|------|---------|
| `remember` | 保存信息到记忆 | /remember |
| `updateConfig` | 修改 settings.json | /config |
| `keybindings` | 键绑定配置 | /keybindings |
| `loop` | 定时循环执行 | /loop 5m /foo |
| `debug` | 调试辅助 | /debug |
| `stuck` | 卡住时的建议 | /stuck |
| `scheduleRemoteAgents` | 调度远程 Agent | /schedule |
| `batch` | 批量操作 | /batch |
| `skillify` | 将流程转为 Skill | /skillify |
| `simplify` | 代码简化审查 | /simplify |
| `claudeApi` | Claude API 开发辅助 | /claude-api |
| `verify` | 验证代码正确性 | /verify |
| `claudeInChrome` | Chrome 浏览器集成 | /chrome |
| `loremIpsum` | 生成占位文本 | /lorem |

## 5. 插件系统

### 5.1 Plugin 定义

```typescript
interface PluginDefinition {
  name: string;
  description: string;
  version: string;

  // 默认启用状态
  defaultEnabled: boolean;

  // 可用性检查
  isAvailable(): boolean;

  // 组件
  skills: BundledSkillDefinition[];
  hooks: HookDefinition[];
  mcpServers: MCPServerConfig[];
}
```

### 5.2 启用/禁用逻辑

```typescript
function isPluginEnabled(plugin: PluginDefinition): boolean {
  // 1. 用户设置覆盖
  const userSetting = getUserPluginSetting(plugin.name);
  if (userSetting !== undefined) return userSetting;

  // 2. 插件默认值
  return plugin.defaultEnabled;
}

function isPluginAvailable(plugin: PluginDefinition): boolean {
  // isAvailable()返回false时，插件完全不可见
  return plugin.isAvailable();
}
```

### 为什么默认启用

```typescript
defaultEnabled: true  // 大多数插件默认启用
```

设计理念是**零配置体验**：用户安装 Claude Code 后，所有有用的功能开箱即用。需要禁用的用户主动关闭，而非需要启用的用户主动开启。这降低了功能发现的门槛。

### isAvailable vs isEnabled

- `isAvailable()`：环境约束（如某个功能需要特定 OS、特定工具安装）。返回 false 时插件**完全不出现**在设置中。
- `isEnabled`：用户选择。返回 false 时插件出现在设置中但被标记为"已禁用"。

区分这两者避免了用户困惑——看到一个不可能工作的插件并试图启用它。

### 5.3 插件 Skill 提取

```typescript
function getBuiltinPluginSkillCommands(): Command[] {
  const commands: Command[] = [];

  for (const plugin of getAllPlugins()) {
    if (!isPluginAvailable(plugin)) continue;
    if (!isPluginEnabled(plugin)) continue;

    for (const skill of plugin.skills) {
      commands.push({
        ...wrapSkillAsCommand(skill),
        source: 'bundled',
        pluginName: plugin.name,
      });
    }
  }

  return commands;
}
```

### 5.4 插件热加载

```typescript
class PluginWatcher {
  private cache = new Map<string, { plugin: PluginDefinition; mtime: number }>();

  async checkForChanges(): Promise<boolean> {
    let changed = false;

    for (const [name, cached] of this.cache) {
      const currentMtime = await getPluginMtime(name);
      if (currentMtime > cached.mtime) {
        this.cache.delete(name);
        changed = true;
      }
    }

    return changed;
  }

  invalidateCache(pluginName: string): void {
    this.cache.delete(pluginName);
  }
}
```

### 5.5 needsRefresh 标志

```typescript
// 检测到变化后设置标志
if (await pluginWatcher.checkForChanges()) {
  store.setState(s => ({ ...s, needsPluginRefresh: true }));
}

// 在下次工具池刷新时应用
if (state.needsPluginRefresh) {
  refreshToolPool();
  store.setState(s => ({ ...s, needsPluginRefresh: false }));
}
```

## 6. Skill 通过 SkillTool 调用

```typescript
const SkillTool = buildTool({
  name: 'SkillInvoke',
  description: '调用已注册的Skill',

  async* call(input: { skillName: string; args: string }, context) {
    const skill = findSkill(input.skillName);
    if (!skill) throw new Error(`Skill not found: ${input.skillName}`);

    // 对于用户脚本类型的Skill，在子进程中运行
    if (skill.source === 'user' && skill.type === 'script') {
      const child = spawn('node', [skill.scriptPath, input.args], {
        cwd: context.getAppState().cwd,
        timeout: 60_000,
      });

      // 流式输出
      for await (const chunk of child.stdout) {
        yield { type: 'progress', output: chunk.toString() };
      }

      const exitCode = await child.exitPromise;
      if (exitCode !== 0) {
        throw new Error(`Skill exited with code ${exitCode}`);
      }

      return { type: 'success' };
    }

    // 对于prompt类型的Skill
    return { type: 'prompt', content: skill.getPromptForCommand(input.args) };
  }
});
```

## 7. Skill 发现与去重

### 7.1 加载优先级

```typescript
async function loadAllSkills(): Promise<Skill[]> {
  // 并行加载三个来源
  const [bundled, plugins, user] = await Promise.all([
    loadBundledSkills(),
    loadPluginSkills(),
    loadUserSkills(),
  ]);

  // 用户Skill优先级最高（可覆盖内置）
  return deduplicateSkills([...user, ...plugins, ...bundled]);
}

function deduplicateSkills(skills: Skill[]): Skill[] {
  const seen = new Map<string, Skill>();

  for (const skill of skills) {
    // 第一个出现的同名Skill保留（用户 > 插件 > 内置）
    if (!seen.has(skill.name)) {
      seen.set(skill.name, skill);
    }
  }

  return Array.from(seen.values());
}
```

### 为什么用户 Skill 可以覆盖内置

用户可能需要自定义内置 Skill 的行为。例如，`/commit` 默认使用 conventional commits 格式，但用户的团队可能使用不同的格式。通过在 `~/.claude/skills/` 中创建同名的 `commit.md`，用户可以无缝覆盖。

### 7.2 Memoized by cwd

```typescript
const skillCache = new Map<string, { skills: Skill[]; mtime: number }>();

function getSkillsForCwd(cwd: string): Skill[] {
  const cached = skillCache.get(cwd);
  const currentMtime = getSkillsDirMtime(cwd);

  if (cached && cached.mtime >= currentMtime) {
    return cached.skills;
  }

  const skills = loadAllSkillsSync(cwd);
  skillCache.set(cwd, { skills, mtime: currentMtime });
  return skills;
}
```

按 cwd 缓存避免了每次命令调用都重新扫描 skills 目录。

## 8. 设计决策总结

| 决策 | 原因 |
|------|------|
| 惰性文件提取 | 大多数Skill未被使用，避免不必要的I/O |
| O_EXCL + O_NOFOLLOW | 防止TOCTOU竞态和符号链接攻击 |
| Per-process nonce | 防止预创建攻击目录 |
| 插件默认启用 | 零配置体验，降低功能发现门槛 |
| isAvailable vs isEnabled | 区分环境约束和用户选择 |
| 用户Skill覆盖内置 | 允许自定义内置行为 |
| 三来源并行加载 | 最小化启动延迟 |
| 按cwd缓存 | 避免重复扫描 |

相关章节：
- [[14 命令系统]]：Skill 如何注册为命令
- [[03 工具系统]]：SkillTool 在工具系统中的角色
- [[11 记忆系统]]：remember Skill 与记忆系统的交互
- [[15 Feature Flags与编译优化]]：构建时 DCE 对 Skill 的影响
