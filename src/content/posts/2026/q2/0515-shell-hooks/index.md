---
title: Hermes Shell Hooks：给 AI 工具调用加一道"安检门"
published: 2026-05-15
pinned: false
description: "必改的几个配置，让你有Hermes最佳的使用体验"
image: ""
tags: ["Hermes Agent", "Shell-Hooks", "AI Agent", "安全配置", "自动化"]
category: Agent Harness
draft: false
slug: 20260515-shell-hooks
---

## 先说我为什么需要它

我很早就怕 AI 读 `.env`，所以早在去年cursor流行的时候，我都不敢用，都是直接把代码贴到Chatgpt来写代码，直到今年的项目中没啥敏感的环境变量，都是本地的配置，所以才开始使用codex让AI完全托管，后面就没有太关注这个事情了。

直到最近开始养“马”，在制定SOUL.md的时候，我就把"不允许读取 .env，并且不允许修改.env"写进去了，一开始执行得很好，AI 还会友好提示："规则不让读，我就不读了，我告诉你怎么改，你自己去弄。弄好了告诉我"。

直到我当时在研究 Hindsight 时（官方还没有在插件中内置），所以折腾的挺时间，也因为这样，所以经常过了很多轮的沟通，直到AI 说：“我发现问题了，你没在 .env 里配置正确的 Hindsight 需要使用LLM APIKEY，apikey显示为: xxxx ” 。我当时就懵了，我明明就配置了，所以只能去看一下，这不看不要紧，一看吓一跳，配置文件中现在就只躺着4条配置，还是带星号的，我其它几十行的环境变量都没了。还好当时在弄 Hindsight 之前就对整个 hermes 备份了。

原来 SOUL.md 的规则在上下文压缩后会被稀释，记忆也不是 100% 可靠。实际上就算是接入了 Hindsight 也是一样的，它仍然会有时候不遵守规则, 只有 Shell Hooks 这种**机制层面的拦截**才能真正防住。

但 `.env` 只是起点。既然能拦 `.env`，那 `.ssh/id_rsa`、`.aws/credentials`、`.kube/config` 呢？思路一打开，干脆把敏感文件统一管起来，再顺手把 `eval` 也拦了——它能直接执行字符串，绕过文件读取限制来读环境变量。一个脚本，三层保护：读、写、eval。

---

## 它到底能干啥

简单说，就是在 Hermes 执行工具调用之前或之后，自动跑一段你写的 shell 脚本。

按价值分层：

| 场景 | 价值 | 优先级 |
|---|---|---|
| **阻止读取 `.env`、`.ssh/*`、AWS 凭证等** | 防止密钥泄露 | **最高** |
| **阻止写入 `.env`、覆盖配置文件** | 防止恶意/误操作 | **最高** |
| **阻止 `eval` 执行动态字符串** | 绕过文件读取限制 | **高** |
| 写完 Python 自动 `black` 格式化 | 代码风格 | 低 |
| 每次对话前注入 `git status` | 信息辅助 | 低 |

**我的原则：安全场景必须上，其他看心情。**

---

## 我为啥不折腾 Python 插件

知道了 Shell hooks 能做什么，你可能会问：这和 Hermes 的 Plugin hooks 有什么区别？我该用哪个？

因为 Hermes 提供了不止一种扩展机制。除了 Shell hooks，还有 Plugin hooks 也能实现工具拦截、注入上下文。它们本质上是同一套事件系统的两种实现方式。

我看了眼源码，发现它俩走的是同一个 `invoke_hook()` 函数。Plugin hooks 先注册，冲突时优先；Shell hooks 后注册，作为补充。

那我为什么选 Shell hooks？不是因为 Plugin hooks 不好，而是我的场景用 shell 就够用了：

| 我的需求            | Shell hooks 能否满足      |
| --------------- | --------------------- |
| 字符串匹配拦截 `.env`  | ✅ 能                   |
| 调用 `black` 格式化  | ✅ 能                   |
| 执行 `git status` | ✅ 能                   |
| 修改 AST、分析代码结构   | ❌ 不能（得用 Plugin hooks） |

我的想法是：**能用 shell 解决的，绝不写插件。** 不是 Plugin hooks 不好，是没必要为了几行字符串处理引入一整套 Python 工程。

但如果你需要修改 AST、做复杂代码分析、或者深度集成 Hermes 内部状态，Plugin hooks 是唯一选择。

---

## 怎么配置

在 `config.yaml` 里声明：

```yaml
hooks:
  pre_tool_call:
    - matcher: "terminal|read_file|write_file|patch"
      command: "~/.hermes/agent-hooks/block-sensitive-ops.sh"
      timeout: 5
  post_tool_call:
    - matcher: "write_file|patch"
      command: "~/.hermes/agent-hooks/auto-format.sh"
      timeout: 10
  pre_llm_call:
    - command: "~/.hermes/agent-hooks/inject-cwd-context.sh"
      timeout: 5
```

上面同时展示了三种触发时机：
- `pre_tool_call`：工具执行前拦截（安全场景）
- `post_tool_call`：工具执行后善后（格式化场景）
- `pre_llm_call`：LLM 回答前注入上下文（git 状态场景）

注：`timeout: 5` 是脚本最大执行时间（秒）。如果脚本 5 秒内没返回，Hermes 会强制终止并视为超时失败。5 秒对于示例脚本非常宽松，通常几毫秒就完成。设置 timeout 是为了防止脚本卡住阻塞整个 agent 循环。

**关于 `matcher` 的建议**：

| 方式 | 示例 | 适用场景 |
| --- | --- | --- |
| **显式写上** | `matcher: "terminal\|read_file\|write_file\|patch"` | 推荐。配置即文档，一眼看出这个 hook 管哪些工具 |
| **删除 matcher** | 不写 | 脚本对所有工具生效。简单但不透明，容易误触发 |

**最佳实践**：显式写 `matcher` + 脚本内部二次校验。

- `matcher`：配置层面控制哪些工具触发 hook（粗筛）
- 脚本内部 `grep`/`jq`：精确判断具体操作是否该拦截（精筛）

比如 `block-sensitive-ops.sh` 同时拦截 `terminal` 和 `read_file`，因为 AI 可能用 `cat .env` 也可能用 `read_file(path=".env")` 读取。

---

## 怎么写脚本

Hermes 会把当前上下文以 JSON 格式通过 **stdin** 传给脚本，脚本通过 **stdout** 返回结果。

stdin 收到的数据：

```json
{
  "hook_event_name": "pre_tool_call",
  "tool_name": "terminal",
  "tool_input": {"command": "cat .env"},
  "session_id": "sess_abc123",
  "cwd": "/home/user/project"
}
```

stdout 返回的格式：

```jsonc
// 阻止工具执行
{"decision": "block", "reason": "Forbidden: reading .env files"}

// 注入上下文给 LLM
{"context": "Current git status: 3 files modified"}

// 什么都不做，放行
{}
```

---

## 核心脚本：安全拦截

这是我最推荐的脚本，覆盖读取敏感文件、写入敏感文件、拦截 eval 三个场景。

```bash
#!/usr/bin/env bash
# ~/.hermes/agent-hooks/block-sensitive-ops.sh

payload="$(cat -)"
cmd=$(echo "$payload" | jq -r '.tool_input.command // empty')
path=$(echo "$payload" | jq -r '.tool_input.path // empty')
tool_name=$(echo "$payload" | jq -r '.tool_name // empty')

# ── 敏感文件清单 ──
# 写裸路径即可，不需要手动转义
SENSITIVE_PATTERNS=(
  '.env'                    # .env 及其变体
  '.env.local'
  '.env.production'
  '.aws/credentials'
  '.ssh/id_rsa'
  '.ssh/id_ed25519'
  '.ssh/known_hosts'
  '.docker/config.json'
  '.netrc'
  '.npmrc'
  '.pypirc'
  'git-credentials'
  '.git-credentials'
  '.config/gh/hosts.yml'
  '.kube/config'
)

# 构建正则：匹配作为独立路径成分出现的敏感文件
# 自动为每个模式添加转义（. → \.）
build_sensitive_regex() {
  local parts=()
  for p in "${SENSITIVE_PATTERNS[@]}"; do
    # 转义正则特殊字符 . 和 /
    local escaped="${p//./\.}"
    parts+=("$escaped")
  done
  local joined=$(IFS='|'; echo "${parts[*]}")
  echo "(^|[[:space:]/\"'\"'\"'])("$joined")([[:space:]/\"'\"'\"']|$)"
}

SENSITIVE=$(build_sensitive_regex)

# ── 1. 阻止读取敏感文件 ──
# 读取类工具：read_file、browser
if [[ "$tool_name" == "read_file" ]] || [[ "$tool_name" == "browser" ]]; then
  if echo "$path" | grep -qE "$SENSITIVE"; then
    printf '{"decision": "block", "reason": "blocked: reading sensitive files is not permitted"}\n'
    exit 0
  fi
fi

# terminal 命令里出现敏感路径
if [[ "$tool_name" == "terminal" ]]; then
  if echo "$cmd" | grep -qE "$SENSITIVE"; then
    printf '{"decision": "block", "reason": "blocked: reading sensitive files is not permitted"}\n'
    exit 0
  fi
fi

# ── 2. 阻止写入/覆盖敏感文件 ──
if [[ "$tool_name" == "write_file" ]] || [[ "$tool_name" == "patch" ]]; then
  if echo "$path" | grep -qE "$SENSITIVE"; then
    printf '{"decision": "block", "reason": "blocked: writing to sensitive files is not permitted"}\n'
    exit 0
  fi
fi

# ── 3. 阻止 eval 执行动态字符串 ──
# eval 能直接执行字符串，可能绕过文件读取限制来读环境变量
if echo "$cmd" | grep -qE '\beval\s*\$|\beval\s*"'; then
  printf '{"decision": "block", "reason": "blocked: eval is not permitted"}\n'
  exit 0
fi

# 放行
printf '{}\n'
```

**别忘了加执行权限**：

```bash
chmod +x ~/.hermes/agent-hooks/block-sensitive-ops.sh
```

这个脚本覆盖了两个层面：
- **读保护**：`.env`、SSH 密钥、AWS 凭证、kubeconfig 等
- **写保护**：同上，防止 AI "帮忙配置" 时覆盖敏感文件
- **eval 拦截**：`eval` 能直接执行字符串，可能绕过文件读取限制来读环境变量

**为什么不做其他命令拦截？**

`curl | bash` 这些确实危险，但 Hermes 自身的 approval 机制已经能管控。而且只要敏感文件读不到，AI 就拿不到密钥，自然发不出去。源头控住就够了。但 `eval` 是例外——它能直接执行动态字符串，必须单独拦截。

---

## 可选参考：格式化与上下文注入

这两个属于锦上添花，不是必须。

### 写完 Python 自动格式化

```bash
#!/usr/bin/env bash
# ~/.hermes/agent-hooks/auto-format.sh

payload="$(cat -)"
path=$(echo "$payload" | jq -r '.tool_input.path // empty')

[[ "$path" == *.py ]] && command -v black > /dev/null && black "$path" 2>/dev/null

printf '{}\n'
```

`post_tool_call` 触发 → 工具已执行完毕，这里只是"善后"。返回 `{}` 表示不拦截，只是静默处理。

### 每次对话前注入 git 状态

```bash
#!/usr/bin/env bash
# ~/.hermes/agent-hooks/inject-cwd-context.sh

cat - > /dev/null  # 丢弃 stdin，我们只需要当前目录

if status=$(git status --porcelain 2> /dev/null) && [[ -n "$status" ]]; then
  jq --null-input --arg s "$status" \
     '{context: ("Uncommitted changes in cwd:\n" + $s)}'
else
  printf '{}\n'
fi
```

`pre_llm_call` 触发 → 每次 AI 准备回答前执行。有变更才注入，避免空内容干扰上下文。

---

## 安全机制

**首次批准**：每个脚本第一次运行时，Hermes 会提示你批准：

```
Allow shell hook pre_tool_call → ~/.hermes/agent-hooks/block-sensitive-ops.sh? [y/N]
```

批准后写入 `~/.hermes/shell-hooks-allowlist.json`，以后不再问。

**脚本改了怎么办**：批准只认路径，不认内容。脚本被改了不会触发重新批准。用 `hermes hooks doctor` 检查是否被修改过。

**非交互环境**：Gateway、cron 没有 TTY，需要加 `--accept-hooks` 或 `HERMES_ACCEPT_HOOKS=1`。

---

## 管理命令

```bash
# 查看所有配置的 hooks
hermes hooks list
# 指定工具类型测试（模拟 terminal 工具被调用）
hermes hooks test pre_tool_call --for-tool terminal --payload-file <( echo '{"args":{"command":"cat .env"}}' )
# 健康检查
hermes hooks doctor
# 撤销授权
hermes hooks revoke "block-sensitive-ops.sh"
```

**⚠️ `hermes hooks test` 的正确用法**：

测试时必须用 **`args`** 作为 key 来覆盖工具参数，因为 `--payload-file` 的 JSON 是直接合并到 payload 根上的。

**错误的测试命令**（传 `command` 到根上，会被放进 `extra`，脚本读不到）：
```bash
hermes hooks test pre_tool_call --for-tool terminal --payload-file <( echo '{"command":"cat .env"}' )
```

**正确的测试命令**（传 `args` 覆盖默认参数）：
```bash
hermes hooks test pre_tool_call --for-tool terminal --payload-file <( echo '{"args":{"command":"cat .env"}}' )
```

**脚本写法**（只需读 `tool_input`，无需兼容 `extra`）：
```bash
payload="$(cat -)"
cmd=$(echo "$payload" | jq -r '.tool_input.command // empty')
path=$(echo "$payload" | jq -r '.tool_input.path // empty')
```

**注意区分两个 hooks 目录**：

| 目录 | 用途 |
| --- | --- |
| `~/.hermes/agent-hooks/` | **Shell hooks** 脚本（本文讲的） |
| `~/.hermes/hooks/` | **Gateway Event Hooks**（Python 插件，不是一回事） |

---

## 我现在配了什么

核心就一个：

1. `pre_tool_call` + `block-sensitive-ops.sh` — 阻止读取/写入敏感文件、拦截 eval

这个够了。其他两个（格式化、git 注入）看个人需求，不是必须。

如果你刷到了这篇文章，还是强烈建议你配置一下，不然指不定哪天你的 apikey等敏感信息，就是公开可见了，你也去试试吧。