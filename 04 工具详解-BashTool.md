# 第4章 工具详解 - BashTool

> **作者**: [Wang Yanshu](https://catyans.github.io)

> BashTool 是 Claude Code 中最复杂的工具实现，由 20 个文件组成。它拥有三层安全架构：语法安全检查（23项）、权限规则匹配和只读模式强制。本章深入分析每一层的设计。

## 1. 三层安全架构概述

```
用户输入 / 模型生成的命令
        │
        ▼
┌─────────────────────────────┐
│  第1层: bashSecurity.ts     │  语法级安全检查（23项）
│  (102KB, 检测危险模式)       │  → 拒绝注入、混淆、提权
└──────────┬──────────────────┘
           │ 通过
           ▼
┌─────────────────────────────┐
│  第2层: bashPermissions.ts  │  权限规则匹配（98KB）
│  (命令+路径+操作符匹配)      │  → 检查用户/企业配置的规则
└──────────┬──────────────────┘
           │ 通过
           ▼
┌─────────────────────────────┐
│  第3层: readOnlyValidation  │  只读模式强制（68KB）
│  (写入操作检测)              │  → 在只读模式下阻止写入
└──────────┬──────────────────┘
           │ 通过
           ▼
        执行命令
```

三层各自独立运作，职责分明：
- **第1层**：阻止攻击（注入、混淆、提权）——安全问题
- **第2层**：检查授权（用户是否允许此命令+路径）——策略问题
- **第3层**：强制只读（在受限模式下阻止写操作）——模式问题

## 2. 23 项安全检查

### 2.1 ValidationContext

每个验证器都接收一个统一的上下文对象：

```typescript
interface ValidationContext {
  originalCommand: string;         // 原始命令字符串
  baseCommand: string;             // 解析出的基础命令（如 "git"）
  unquotedContent: string;         // 去除引号后的内容
  fullyUnquotedContent: string;    // 完全去除所有引号层的内容
  fullyUnquotedPreStrip: string;   // 去引号但保留空白的内容
  unquotedKeepQuoteChars: string;  // 去引号但保留引号字符本身
  treeSitter: TreeSitterAnalysis;  // Tree-sitter AST分析结果
}
```

`treeSitter` 提供了命令的 AST 级别理解，比纯正则表达式更准确。例如，它能区分命令参数中的 `$()` 和实际的命令替换。

### 2.2 完整检查列表

以下按执行顺序列出全部 23 项安全检查：

#### 1. INCOMPLETE_COMMANDS（不完整命令）

```typescript
// 检测未闭合的引号、括号、管道等
// 防止命令注入通过不完整的语法
"echo 'hello"      // 拒绝：未闭合的单引号
"cat file |"        // 拒绝：悬挂的管道
```

**威胁**：不完整命令可能在 shell 中等待额外输入，被后续命令注入利用。

#### 2. JQ_SYSTEM_FUNCTION（jq 系统函数）

```typescript
// 检测 jq 中的 @base64d, @html, @json, env, $ENV 等
"jq 'env.HOME'"    // 拒绝：访问环境变量
```

**威胁**：jq 的内置函数可以读取环境变量和执行编码/解码，绕过其他检查。

#### 3. JQ_FILE_ARGUMENTS（jq 文件参数）

```typescript
// 检测 jq 的 --rawfile, --slurpfile 等文件读取参数
"jq --rawfile x /etc/passwd"  // 拒绝：通过jq读取敏感文件
```

**威胁**：jq 可以通过参数读取任意文件内容。

#### 4. OBFUSCATED_FLAGS（混淆标志）

```typescript
// 检测使用Unicode或控制字符伪装的命令行标志
"rm -\u200Brf /"   // 拒绝：零宽空格隐藏在标志中
```

**威胁**：混淆标志可以让危险命令看起来无害。

#### 5. SHELL_METACHARACTERS（Shell 元字符）

```typescript
// 检测危险的 shell 元字符组合
"cat file; rm -rf /"    // 拒绝：分号连接危险命令
"cat $(malicious)"      // 拒绝：命令替换
```

**威胁**：元字符可以将安全命令转变为危险操作的载体。

#### 6. DANGEROUS_VARIABLES（危险变量）

```typescript
// 检测修改关键 shell 变量
"PATH=/tmp:$PATH cmd"   // 拒绝：PATH篡改
"LD_PRELOAD=evil.so"    // 拒绝：库注入
```

**威胁**：修改 `PATH`、`LD_PRELOAD`、`LD_LIBRARY_PATH` 可以实现代码注入。

#### 7. NEWLINES（换行符）

```typescript
// 检测命令中的换行符
"echo hello\nrm -rf /"  // 拒绝：隐藏的第二条命令
```

**威胁**：换行符可以在看似单行命令中隐藏额外命令。

#### 8. COMMAND_SUBSTITUTION（命令替换）

```typescript
// 检测 $() 和 `` 命令替换
"echo $(cat /etc/shadow)"  // 拒绝：通过替换泄露信息
```

**威胁**：命令替换允许在参数位置执行任意命令。有安全例外：git commit 消息中的常见模式被允许。

#### 9. INPUT_REDIRECTION（输入重定向）

```typescript
// 检测 < 和 <<< 输入重定向
"mysql < drop_all.sql"  // 拒绝：危险的输入重定向
```

**威胁**：输入重定向可以向程序注入任意内容。有例外：heredoc（`<< 'EOF'`）在某些安全场景下被允许。

#### 10. OUTPUT_REDIRECTION（输出重定向）

```typescript
// 检测 > 和 >> 输出重定向
"echo malicious > ~/.bashrc"  // 拒绝：覆盖配置文件
```

**威胁**：输出重定向可以覆盖或追加任意文件内容。

#### 11. IFS_INJECTION（IFS 注入）

```typescript
// 检测 IFS 变量的修改
"IFS=/ cmd arg"  // 拒绝：改变字段分隔符
```

**威胁**：修改 `IFS`（Internal Field Separator）可以改变 shell 解析参数的方式，导致命令被意外拆分。

#### 12. GIT_COMMIT_SUBSTITUTION（Git 提交替换）

```typescript
// 检测 git commit 消息中的命令替换
"git commit -m '$(whoami)'"  // 拒绝：提交消息中的命令替换
```

**威胁**：某些 git 钩子可能会处理提交消息中的特殊字符。

#### 13. PROC_ENVIRON_ACCESS（/proc 环境访问）

```typescript
// 检测对 /proc/self/environ 等路径的访问
"cat /proc/self/environ"  // 拒绝：泄露环境变量（含密钥）
```

**威胁**：`/proc/self/environ` 包含所有环境变量，可能含有 API 密钥等敏感信息。

#### 14. MALFORMED_TOKEN_INJECTION（畸形Token注入）

```typescript
// 检测在token边界注入的特殊字符
"cmd arg\x00hidden"  // 拒绝：null字节截断
```

**威胁**：null 字节和其他控制字符可以截断或修改命令解析。

#### 15. BACKSLASH_ESCAPED_WHITESPACE（反斜杠转义空白）

```typescript
// 检测反斜杠转义的空白字符
"rm\ -rf\ /"  // 拒绝：通过转义隐藏参数分隔
```

**威胁**：转义空白可以改变参数边界，使危险参数被合并到安全参数中。

#### 16. BRACE_EXPANSION（花括号展开）

```typescript
// 检测危险的花括号展开
"cat /etc/{passwd,shadow}"  // 拒绝：通过展开访问多个敏感文件
```

**威胁**：花括号展开可以在单个命令中访问多个路径，绕过单路径权限检查。

#### 17. CONTROL_CHARACTERS（控制字符）

```typescript
// 检测 ASCII 控制字符（除了\t\n\r）
"cmd \x1b[2J"  // 拒绝：终端逃逸序列
```

**威胁**：控制字符可以操纵终端显示，隐藏实际执行的命令。

#### 18. UNICODE_WHITESPACE（Unicode 空白）

```typescript
// 检测非ASCII空白字符
"cmd\u2003arg"  // 拒绝：em space等Unicode空白
```

**威胁**：Unicode 空白字符在终端中不可见，可以改变命令解析。

#### 19. MID_WORD_HASH（词中井号）

```typescript
// 检测在单词中间出现的 # 字符
"cmd#comment rm -rf"  // 拒绝：可能的注释注入
```

**威胁**：在某些 shell 配置中，`#` 可以开始注释，导致前面的内容被忽略。

#### 20. ZSH_DANGEROUS_COMMANDS（zsh 危险命令）

```typescript
// 检测 zsh 特有的危险内建命令
"zmodload zsh/system"  // 拒绝：加载系统模块
"sched +5 malicious"   // 拒绝：定时执行
```

**威胁**：zsh 有一些 bash 没有的内建命令，可以进行系统级操作。

#### 21. BACKSLASH_ESCAPED_OPERATORS（反斜杠转义运算符）

```typescript
// 检测反斜杠转义的 shell 运算符
"cmd \&\& rm -rf"  // 拒绝：隐藏的命令链接
```

**威胁**：转义运算符可能在某些解析阶段被还原，导致意外的命令链接。

#### 22. COMMENT_QUOTE_DESYNC（注释引号不同步）

```typescript
// 检测引号和注释的不一致状态
"echo 'hello' #' rm -rf /"  // 拒绝：引号/注释混淆
```

**威胁**：不同的解析器（人眼、shell、安全检查）可能对引号/注释边界有不同理解。

#### 23. QUOTED_NEWLINE（引号内换行）

```typescript
// 检测引号字符串内的换行符
"echo 'line1\nrm -rf /'"  // 检查上下文决定是否安全
```

**威胁**：引号内的换行在某些场景下会被 shell 展开为实际换行。

## 3. 验证器链执行顺序

23 个验证器按严格顺序执行，且有**早期允许（early-allow）**机制：

```typescript
function runValidators(ctx: ValidationContext): SecurityResult {
  // 安全模式快速通过
  if (isSafeGitCommit(ctx)) return { allowed: true };
  if (isSafeHeredoc(ctx)) return { allowed: true };
  if (isSafeKnownCommand(ctx)) return { allowed: true };

  // 依次执行23个验证器
  for (const validator of VALIDATORS) {
    const result = validator(ctx);
    if (result.denied) {
      return result; // 第一个拒绝即停止
    }
  }

  return { allowed: true };
}
```

早期允许的原因：
- **git commit**：提交消息中经常包含特殊字符，这些在 git 上下文中是安全的
- **heredoc**：heredoc 语法（`<< 'EOF'`）需要换行和特殊字符，有引号保护
- **已知安全命令**：如 `ls`、`pwd`、`date` 等纯读取命令

## 4. 权限规则匹配

### 4.1 bashToolHasPermission 流程

```
命令字符串
    │
    ▼
解析（Parse）
    │ 提取基础命令、参数、路径
    ▼
语义检查（Semantics Check）
    │ 理解命令意图
    ▼
安全检查（Security Checks）
    │ 23项验证器
    ▼
路径约束（Path Constraints）
    │ 检查命令涉及的文件路径
    ▼
命令操作符权限（Command Operator Permissions）
    │ 检查管道、重定向等
    ▼
权限模式（Permission Mode）
    │ ask / auto / bypass
    ▼
最终决策
```

### 4.2 路径约束检查

```typescript
function checkPathConstraints(
  command: ParsedCommand,
  rules: PermissionRule[]
): PermissionDecision {
  for (const path of command.involvedPaths) {
    const resolved = resolvePath(path, command.cwd);

    for (const rule of rules) {
      if (rule.pathPattern && matchPath(resolved, rule.pathPattern)) {
        if (rule.decision === 'deny') {
          return { decision: 'deny', reason: `路径 ${resolved} 被规则禁止` };
        }
      }
    }
  }

  return { decision: 'continue' }; // 继续下一步检查
}
```

### 4.3 命令操作符权限

管道（`|`）、后台（`&`）、命令链（`&&`、`||`）等操作符会改变命令的执行语义。权限系统对这些操作符有独立的检查：

```typescript
// 管道中的每个命令都单独检查
"cat file | grep pattern"
→ checkPermission("cat file")    // 只读：允许
→ checkPermission("grep pattern") // 只读：允许
→ 整体：允许

"cat file | rm -rf /"
→ checkPermission("cat file")    // 只读：允许
→ checkPermission("rm -rf /")    // 危险：拒绝
→ 整体：拒绝
```

## 5. 只读模式强制

### 5.1 写入操作检测

```typescript
function isWriteOperation(command: ParsedCommand): boolean {
  // 检查已知的破坏性命令
  if (DESTRUCTIVE_COMMANDS.includes(command.baseCommand)) return true;

  // 检查 sed/awk 的写入模式
  if (command.baseCommand === 'sed' && command.args.includes('-i')) return true;
  if (command.baseCommand === 'awk' && hasInplaceFlag(command)) return true;

  // 检查安全命令白名单
  if (SAFE_COMMANDS.includes(command.baseCommand)) return false;

  // 默认保守：未知命令视为可能的写入
  return true;
}
```

### 5.2 破坏性命令列表

```typescript
const DESTRUCTIVE_COMMANDS = [
  'rm', 'rmdir',           // 删除
  'mv',                     // 移动/重命名
  'cp',                     // 复制（可能覆盖）
  'chmod', 'chown', 'chgrp', // 权限修改
  'touch',                  // 创建/修改时间戳
  'mkdir',                  // 创建目录
  'ln',                     // 创建链接
  'dd',                     // 块设备操作
  'truncate',               // 截断文件
  'shred',                  // 安全删除
  // ...更多
];
```

### 5.3 安全命令白名单

```typescript
const SAFE_COMMANDS = [
  'ls', 'dir', 'find',     // 列表/搜索
  'cat', 'head', 'tail',   // 读取
  'grep', 'rg', 'ag',      // 搜索
  'wc', 'sort', 'uniq',    // 文本处理（只输出不修改）
  'file', 'stat', 'du',    // 信息查询
  'git log', 'git status', // Git只读操作
  'git diff', 'git show',
  'pwd', 'which', 'type',  // 环境查询
  'date', 'uname', 'id',   // 系统信息
  'echo', 'printf',        // 输出
  // ...更多
];
```

## 6. 自动分类器（Auto-Classifier）

### 6.1 投机分类检查

```typescript
async function speculativeClassifierCheck(
  command: string,
  context: ClassifierContext
): Promise<ClassificationResult> {
  // 快速模式匹配
  const quickResult = quickPatternMatch(command);
  if (quickResult !== 'unknown') return quickResult;

  // 调用小模型进行分类
  return await classifyWithModel(command, context);
}
```

### 6.2 待定分类检查

```typescript
// 在权限检查的宽限期内运行
const pendingClassifierCheck = createPendingClassifier({
  gracePeridMs: 200,  // 200ms宽限期
  onResult: (result) => {
    if (result === 'allow') {
      resolvePermission({ decision: 'allow' });
    }
  },
});
```

投机分类器在权限对话框显示给用户之前运行。如果在 200ms 宽限期内分类为安全，自动批准而不打扰用户。

### 为什么需要宽限期

用户输入速度很快时，工具调用可能在用户还在打字时触发。200ms 的宽限期让自动分类器有时间判断，避免在用户打字过程中弹出权限对话框。

## 7. 设计决策总结

### 为什么需要三层而非一层

单层安全检查的问题：
- 安全检查和策略检查混在一起，难以维护
- 只读模式的逻辑与安全检查逻辑纠缠
- 不同部署场景需要不同的策略配置，但安全检查应该统一

三层分离使得：
- 安全检查（第1层）对所有部署统一应用
- 权限规则（第2层）可由企业管理员自定义
- 只读模式（第3层）可在运行时切换

### 为什么23个检查而不是更少

每个检查针对一种特定的攻击向量。合并检查会导致：
- 难以禁用单个检查（调试或特殊场景）
- 错误报告不精确（用户不知道哪个规则被触发）
- 早期允许逻辑复杂化

### Tree-sitter 的角色

纯正则表达式的限制：
- 无法区分引号内外的特殊字符
- 无法理解嵌套结构
- 容易被构造性的绕过欺骗

Tree-sitter 提供了 shell 脚本的 AST，使安全检查可以在语法层面而非文本层面运行。例如，它能准确识别 `$(...)` 是在双引号内（安全）还是在命令位置（危险）。

### 投机分类器的设计

投机分类器是一个"乐观锁"——假设大多数命令是安全的，先尝试自动批准，失败时才升级为交互式审批。这减少了权限对话框的出现频率，提升了用户体验，同时不牺牲安全性（最终总会由某种机制做出决策）。

相关章节：
- [[07 权限系统]]：权限决策管道的完整流程
- [[03 工具系统]]：BashTool 在工具系统中的注册
- [[02 查询引擎]]：BashTool 执行结果的处理
- [[15 Feature Flags与编译优化]]：某些安全检查受 feature flag 控制
