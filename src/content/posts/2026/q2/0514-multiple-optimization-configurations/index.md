---
title: Hermes多项调优配置
published: 2026-05-14
pinned: false
description: "必改的几个配置，让你有Hermes最佳的使用体验"
image: ""
tags: ["AI Agent", "Config"]
category: Hermes
draft: false
slug: 20260514-multiple-optimization-configurations
---

## 先说我踩过的坑

Hermes Agent 装完就能用？是能跑，但跑起来费钱、费时间、还时不时死循环。

我调了 8 个配置之后，日均消耗从 15 块降到 6-7 块，再也没遇到过工具死循环。关键是这几个改动都不复杂，改完立马见效。

---

## 一图看懂：改前 vs 改后

| 配置项 | 改前 | 改后 | 效果 |
| --- | --- | --- | --- |
| 忙时输入模式 | interrupt（打断式） | steer（追加式） | 执行中可随时追加指令 |
| Prompt 缓存时间 | 5 分钟 | 1 小时 | 长对话省钱省时间 |
| 工具断路器 | 硬停止关闭 | 硬停止开启 | 防止死循环烧钱 |
| 子任务深度 | 1 层 | 2 层 | 复杂任务拆得更细 |
| 子代理工具集 | 3 项基础 | +web+search+浏览器 | 子代理能力更强 |
| 隐私脱敏 | 关闭 | 开启 | 日志不泄露隐私 |
| Skill 安全扫描 | 关闭 | 建议保持关闭 | 避免误杀自用 skill |
| 智能模型路由 | 关闭 | 开启 | 简单问题用便宜模型 |

配置文件位置：`~/.hermes/config.yaml`

---

## 逐个说

### 1. 忙时输入模式：interrupt → steer

**作用**：中途插话不会打断 AI 当前任务，而是作为追加指令排队处理。

```yaml
display:
  busy_input_mode: steer  # 默认是 interrupt
```

**我为什么改**：

之前用 `interrupt`，任务跑到一半我说"等一下"，AI 直接停了，结果是个半成品。后来改成 `steer`，我说"输出格式改成表格"，AI 记在心里，当前步骤跑完自动调整，任务不会断。

---

### 2. Prompt 缓存时间：5m → 1h

**作用**：避免重复发送几千字的系统提示词，减少 token 消耗。

```yaml
prompt_caching:
  cache_ttl: 1h  # 默认是 5m
```

**算账**：

5 分钟缓存什么概念？一个复杂任务跑 10 分钟，中途缓存失效，系统提示词重新上传，多烧一次钱。改成 1 小时之后，绝大多数任务都能覆盖，系统提示词只传一次。

---

### 3. 工具循环断路器：开启硬停止

**作用**：防止 AI 在同一个失败工具上无限重试，烧光预算。

```yaml
tool_loop_guardrails:
  warnings_enabled: true      # 默认已开启，无需改动
  hard_stop_enabled: true     # ⚠️ 默认 false，这是真正要改的
  warn_after:
    same_tool_failure: 3      # 同一工具失败 3 次后警告
  hard_stop_after:
    same_tool_failure: 8      # 同一工具失败 8 次后强制停止
```

**注意**：原文说"没开"有误导。`warnings_enabled` 默认就是 `true`，真正缺的是 `hard_stop_enabled`。我开之前遇到过 AI 在一个报错上死磕十几轮，token 烧了一堆啥也没干成。

---

### 4. 子任务嵌套深度：1 → 2

**作用**：允许子代理继续 spawn 子代理，复杂任务拆解更细。

```yaml
delegation:
  max_spawn_depth: 2  # 默认是 1
```

**我为什么改**：

默认 1 层意味着子代理不能再拆任务。我用 Kanban 编排多步骤任务时，主控 → orchestrator → leaf worker 需要三层结构，1 层根本跑不通。改成 2 层之后，复杂任务才能真正拆细。

---

### 5. 子代理工具集：增加 web、search、browser

**作用**：让子代理具备完整的网络能力，不只是文件操作。

```yaml
delegation:
  default_toolsets:
    - terminal
    - file
    - web
    - search      # 深度搜索：多轮聚合、跨站点调研、资料穷尽
    - browser     # 浏览器自动化：登录、点击、填表、截图
```

**三项网络工具的分工**：

| 工具 | 能做什么 | 不能做什么 |
| --- | --- | --- |
| web | 基础搜索、页面内容提取 | 登录态页面、动态交互 |
| search | 深度搜索、多源聚合、结构化报告 | 网页操作 |
| browser | 点击、填表、滚动、截图、登录 | 快速信息检索 |

**实际怎么用**：

查资料的时候，`web_search` 先定位目标页面，`web_extract` 静态提取内容。如果页面需要登录或者提取失败，切 `browser` 兜底操作。深度调研就用 `search` 多轮聚合。

---

### 6. 隐私脱敏：false → true

**作用**：自动脱敏手机号、身份证号、邮箱等 PII，日志不再明文存储。

```yaml
privacy:
  redact_pii: true  # 默认是 false
```

**我为什么改**：

之前在 Telegram 里发了身份证号让 AI 填表，后来看日志发现完整号码明文躺在那里。如果日志备份到云端就泄露了。开 `true` 之后，日志里显示 `[REDACTED]` 或 `[PHONE]`，安心很多。

---

### 7. Skill 安全扫描：建议保持 false ⚠️

**配置**：

```yaml
skills:
  guard_agent_created: false  # 建议保持默认 false
```

**我为什么不开**：

| 场景 | 扫描行为 | 结果 |
| --- | --- | --- |
| 从 Hub 安装第三方 skill | 始终扫描（不受此配置影响） | 安全 |
| Agent 自己创建 skill | false → 不扫描 | 正常 skill 顺利通过 |
| Agent 自己创建 skill | true → 扫描 | 误杀率高，正常 skill 里的 rm、curl、password 等关键词会触发拦截 |

外部 skill 的安检已内置，不受这个开关影响。开 `true` 只会增加你自己写 skill 的摩擦，无实质安全增益。Agent 本就能通过 `terminal()` 执行任意命令，扫描 skill 文件是"防君子不防小人"。

---

### 8. 智能模型路由：false → true

**作用**：按问题复杂度自动选择模型，简单问题用便宜模型。

```yaml
smart_model_routing:
  enabled: true           # 默认 false
  max_simple_chars: 160   # 超过 160 字不走便宜模型
  max_simple_words: 28    # 超过 28 个词不走便宜模型
  cheap_model:
    provider: openrouter   # 填你的便宜模型提供商
    model: google/gemini-flash-1.5
```

**分诊逻辑**：
- "今天几号" → 便宜模型（~¥0.01）
- "帮我写封邮件" → 中等模型（~¥0.05）
- "帮我重构这段代码" → 强力模型（~¥2.00）

**注意**：此项可能需要配合源码修改或自定义 hooks 才能完全生效，配置 alone 可能只覆盖部分场景。

---

## 效果验证

| 指标 | 改前 | 改后 |
| --- | --- | --- |
| 日均消耗 | ~15 元 | ~6-7 元 |
| 死循环次数 | 偶发 | 0（断路器生效后） |
| 隐私泄露风险 | 中 | 低 |
| 复杂任务拆解能力 | 单层 | 双层嵌套 |

---

## 一句话总结

> 大多数人装完就跑，然后觉得"Hermes 也就那样"——不是 Hermes 不行，是你没调好。

**8 项配置里，真正必改的只有 5 项**：
1. `busy_input_mode: steer`
2. `prompt_caching.cache_ttl: 1h`
3. `tool_loop_guardrails.hard_stop_enabled: true`
4. `delegation.max_spawn_depth: 2`
5. `privacy.redact_pii: true`

其余 3 项（子代理工具集、skill 扫描、智能路由）按你的实际使用场景决定。