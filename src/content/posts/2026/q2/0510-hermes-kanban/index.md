---
title: Hermes Kanban 任务调度实践记录：从配置到可用
published: 2026-05-10
pinned: false
description: "记录将 Hermes Kanban 任务编排跑通全过程。以89平装修预算为例，展示如何用自然语言触发任务链：research查材料价格→default汇总总表。踩坑包括profile环境隔离、通知订阅、中文编码等。"
image: "./cover.avif"
tags: ["Hermes Agent", "Kanban", "AI Agent"]
category: Hermes
draft: false
slug: 20260510-hermes-kanban
---

> **免责声明**：本文仅为个人技术实践记录，不构成任何工具推荐或教程。文中涉及的配置方法为个人使用场景定制，不保证适用于其他环境。所有自动化配置均需人工审核确认，不替代真人决策。使用任何工具请遵守相关平台规则。

## 开头

我这两天主要在折腾 Hermes 的 Kanban。

不是那种"看起来很高级"的多角色展示，而是很朴素的一个目标：我在聊天软件里发一句话，后台能把任务拆开，交给不同 profile 去做，做完以后还能通知我，并且把结果保存成文件。

一开始其实只跑通了"能创建任务"。

但这离真正可用还差很远。

因为日常使用里，我关心的不是任务卡片本身，而是这几个问题：

- 谁在做？
- 做到哪一步了？
- 上游结果有没有交给下游？
- 长结果有没有保存成文件？
- 最后能不能回到聊天软件提醒我？

这篇就是一份个人实践记录。不是教程包装，也不是功能宣传。更像是我把 Kanban 从"能跑"调到"能通知、能交付"的过程记录。

## 一、为什么我需要 Kanban

我已经有多个 Hermes profile。

它们各自都能单独工作，但以前缺少一个稳定的任务协作层。

我现在的分工大概是这样：

```text
default：技术系统主控，负责工具链、配置、排障、落盘和最终验收
research：资料查询、官方文档、事实核查、版本差异梳理
dev：脚本、配置、debug、自动化
life：计划、复盘、日程、生活类方案整理
```

以前普通聊天也能完成很多事，但一旦任务变长，就开始混乱。

比如做一套装修预算方案：

- `research` 先查材料价格、人工费、施工顺序和市场行情。
- `default` 根据资料汇总总预算表，控制预算在 11 万以内。

如果全塞在一个聊天里，问题很明显：上下文很长，中间结论容易散，最后到底哪个版本才是可用结果也不清楚。

Kanban 解决的就是这层问题。

它不是简单的待办清单，而是 Hermes 里的任务协作层：任务可以分配给 profile，可以设置依赖，可以记录 comment、日志和状态，也可以由 gateway 后台拉起执行。

我想要的闭环是：

```text
主控拆任务
→ 专家 profile 执行
→ 中间 comment 交接
→ 长产出保存成文件
→ final task 汇总
→ gateway 通知完成
→ 我只看最终结果
```

## 二、先讲结果：这套配置跑通后是什么体验

配置完成后，最明显的变化是：我不需要一直盯着 dashboard。

我可以在聊天软件里发一句自然语言：

```text
用 Kanban 帮我做一套装修预算方案。房子 89 平两室一厅，毛坯房，预算 11 万以内，简约风格。需要查材料价格、排施工顺序、汇总总预算表。需要保存文档。
```

理想情况下，后面应该发生这些事：

```text
主控创建 Kanban 工作流
→ research 查询材料价格、施工顺序、人工费
→ default 汇总总预算表，控制预算在 11 万以内
→ gateway 在任务结束后通知我
```

注意，这里最重要的不是"多 Agent 同时干活"。

恰恰相反，我后来发现，多数任务应该串行。

装修预算这种任务，如果一上来并发拆给多个 profile，很容易出现信息不一致：一个人按 A 品牌算价格，另一个人按 B 品牌算价格，最后还要主控擦屁股。

更稳的方式是：

```text
research → default
```

上游把结论写进 comment 或文件，下游基于上游结果继续做。最后 default 只做一件事：把用户真正需要看的结果收口。

## 三、Kanban 的优点和缺点

先讲优点。

第一，长任务可以后台跑。

普通聊天里跑长任务，当前对话基本就被占住了。Kanban 创建任务后，可以由 dispatcher 后台执行。你不用一直等在原地。

第二，多 profile 分工更清楚。

不是所有 profile 都叫"助手"。每个 profile 应该有明确边界：查资料的就查资料，做方案的就做方案，判断取舍的就判断取舍，主控最后验收。

第三，状态可追踪。

任务不是一句"我去做了"就没下文。你可以看到它是 ready、running、blocked、done，还是 gave_up。

第四，中间交接可复盘。

Kanban comment 很适合做上下游交接。比如 research 查完资料，不需要把所有内容塞给用户看，只要写清楚：关键结论、证据来源、保存路径、下游该怎么用。

第五，最终交付可以标准化。

我不希望每个 worker 都随便回复一段话。最终应该有 final task，把结果整理成用户真正能用的东西。

再讲缺点。

第一，它不是开箱即用的"自动交付系统"。

Kanban 负责调度、状态、依赖和日志。但最终文件放哪、comment 怎么写、什么时候通知、谁来验收，这些需要自己加规则。

第二，通知有细节坑。

聊天平台里 `/kanban create ...` 可以触发当前聊天的终态订阅，但 CLI 直接执行的任务不会自动通知。这个坑我后面单独写。

第三，comment 不适合放长产出。

comment 是交接和索引，不是文章、攻略、代码、报告本体。长结果建议保存成文件，然后在 comment 里写路径。

第四，多 profile 不等于并发乱撒任务。

大多数任务只需要一个专家。真要多 profile，也建议默认串行。能少拆就少拆。

## 四、如何配置才能让 Kanban 跑起来

这里只写主流程，细节和踩坑放在后面章节。

### 4.1 初始化 Kanban

```bash
hermes kanban init
hermes kanban boards current
```

默认会有一个 `default` board，后面所有测试都用它。

### 4.2 启动 gateway

Kanban 的 dispatcher 由 gateway 托管。

```bash
hermes -p default gateway start
```

### 4.3 验证 worker 能被拉起

```bash
hermes kanban create "smoke test" --assignee research --body "写一句 comment 然后 complete。"
```

观察事件流：

```bash
hermes kanban watch
```

正常会经历 `created → claimed → spawned → commented → completed`。

### 4.4 保留官方 skills

```bash
hermes skills list | grep kanban
```

应该看到 `kanban-orchestrator` 和 `kanban-worker` 是 enabled 状态。建议保留，不要用自己的 skill 替代它们。

### 4.5 用自然语言触发完整闭环（额外配置）

前面 4.1-4.4 是 Kanban 基础配置，跑通后已经能用 CLI 创建任务了。

但日常更方便的用法是：在聊天软件里发一句自然语言，主控简化操作流程，完成"创建任务 → 分配 profile → 建立依赖 → 订阅通知 → 产出落盘"整个流程。

这需要额外配置主控的 system_prompt，让它知道如何处理 Kanban 请求。

**配置方法：**

在 `~/.hermes/config.yaml` 的 `agent.system_prompt` 中加入：

```yaml
system_prompt: |
  ## 交付压缩规则
  - 单任务回执：ID + Assignee + 状态 + 路径 ≤80字
  - 批量回执：合并为单条，每条只保留 ID + Assignee + 状态
  - 完成通知：结论 + 路径 ≤80字。详细内容写入文件，不在通知展开

  ## Kanban 协作规则
  - 复杂任务（3+步骤或多 profile）建议使用 Hermes Kanban 编排
  - 先加载 workspace-management skill 确定落盘路径
  - 创建任务后自动 assign 给对应 profile
  - 只订阅 final task 通知，不逐个订阅中间任务
  - 长产出建议保存到 workspace，不在聊天里展开
```

**使用效果：**

配置完成后，在 Telegram 里发：

```text
用 Kanban 帮我做一套装修预算方案。房子 89 平两室一厅，毛坯房，预算 11 万以内，简约风格。需要查材料价格、排施工顺序、汇总总预算表。需要保存文档。
```

主控会简化操作流程：

```text
1. 加载 workspace-management skill，确定落盘路径
2. 创建项目目录：[你的workspace路径]/para/10-Projects/装修预算-89平毛坯简约风/
3. 创建 research 任务：查材料价格、施工顺序、人工费
4. 创建 default 任务：汇总总预算表
5. 建立依赖：research → default
6. 订阅 final task 通知到当前聊天
7. 返回压缩回执（task id + assignee + 状态）
```

**注意：** 这部分是可选配置。不配置的话，Kanban 仍然可用，只是需要手动 CLI 操作。

### 4.6 产出落文件

所有长产出建议保存到 workspace，不在 comment 里展开。

```text
research-材料价格与施工顺序.md
总预算表.md
```

（路径仅为示例，请替换为你自己的 workspace 目录）

## 五、走通这套流程遇到的问题

### 5.1 任务创建后一直是 ready，不执行

原因：没有 assignee。

`hermes kanban create` 的 `--profile` 参数不等于 `--assignee`。前者只是指定用哪个 profile 的 `.env`，后者才是告诉调度器谁来执行。

解决：创建后手动 assign，或者封装脚本简化操作。

```bash
hermes kanban assign <task_id> research
```

### 5.2 profile 的 `.env` 不继承主目录

每个 profile 有独立的 `.env`，不会自动继承 `~/.hermes/.env`。

比如主目录配置了 `TAVILY_API_KEY`，但 research profile 的 `.env` 里没有，导致 web_search 失败。

解决：按需同步 Key 到各 profile。

```bash
# 检查各 profile 的 .env
cat ~/.hermes/profiles/research/.env
cat ~/.hermes/profiles/life/.env
```

（路径仅为示例，请替换为你自己的 Hermes 配置目录）

### 5.3 CLI 执行的任务不会触发聊天通知

直接在终端里 `hermes kanban create ...` 创建的任务，不会自动向 Telegram/Discord 推送完成通知。

原因：CLI 绕过 gateway，没有聊天上下文。

解决：显式订阅通知目标。

```bash
hermes kanban notify-subscribe <task_id> --platform telegram --chat-id <your_chat_id>
```

（chat-id 仅为示例，请替换为你自己的聊天 ID）

### 5.4 web.extract_backend 为空时走 Tavily，但额度用完会失败

`config.yaml` 里：

```yaml
web:
  backend: tavily
  extract_backend: ''
```

`extract_backend: ''` 表示继承 `backend`，所以提取也走 Tavily。

但 Tavily 和 Firecrawl 都有免费额度限制，用完就会失败。

解决：监控额度，或换号。

### 5.5 中文编码问题

`config.yaml` 的 `system_prompt` 如果写中文，可能被存成 Unicode 转义序列（`\u6267\u884C...`），人很难读。

解决：用 YAML 管道符 `|` 写多行文本，保持中文可读。

```yaml
system_prompt: |
  ## 交付压缩规则
  - 单任务回执：ID + Assignee + 状态 + 路径 ≤80字
```

### 5.6 批量创建任务时回执太长

每创建一个任务就输出一大段 JSON 和说明，信息过载。

解决：在 `system_prompt` 里加约束：

```yaml
system_prompt: |
  ## 交付压缩规则
  - 单任务回执：ID + Assignee + 状态 + 路径 ≤80字
  - 批量回执：合并为单条，每条只保留 ID + Assignee + 状态
  - 完成通知：结论 + 路径 ≤80字。详细内容写入文件，不在通知展开
```

### 5.7 worker crashed 不一定是 Kanban 坏了

可能是 profile 的模型/provider/API key 配置有问题。修好 profile 后，任务就能正常执行。

排查顺序：

```bash
hermes kanban show <task_id>
hermes kanban runs <task_id>
hermes kanban log <task_id>
```

## 六、命令速查

初始化：

```bash
hermes kanban init
hermes kanban boards current
```

启动和诊断：

```bash
hermes -p default gateway start
hermes kanban diagnostics
hermes kanban stats
hermes kanban assignees
```

创建和观察：

```bash
hermes kanban create "标题" --assignee research --body "..."
hermes kanban watch
hermes kanban show <task_id>
hermes kanban runs <task_id>
```

通知：

```bash
hermes kanban notify-subscribe <task_id> --platform telegram --chat-id <your_chat_id>
hermes kanban notify-list <task_id>
```

依赖：

```bash
hermes kanban link <parent> <child>
```

## 总结

这次折腾完以后，Kanban 在我的 Hermes 里不再只是一个任务板。

它更像后台任务协作层。

最终可用的模式是：

```text
一句自然语言请求
→ 主控创建 Kanban 工作流
→ 专家 profile 串行执行
→ comment 做交接
→ 长结果保存成文件
→ final task 汇总
→ gateway 通知完成
```

我这次真正确认下来的结论有几个：

1. `default` board 初期够用，不建议过早拆 board。
2. `default` 负责技术主控和最终验收。
3. profile 按能力分，不按项目分。
4. 官方 `kanban-orchestrator` / `kanban-worker` 保持 enabled。
5. 自己的规则只写偏好和约束，不替代官方生命周期。
6. CLI 创建的任务不会自动通知，要显式订阅。
7. 长产出建议落文件，comment 只写索引。
8. 多 profile 默认串行，最后用 final task 收口。
9. profile 的 `.env` 独立，需要按需同步 Key。
10. "归档任务"不等于"保存产物"。

---

**写在最后**

本文记录的是个人使用 Hermes 进行任务管理的配置过程，属于技术探索笔记。
**不构成教程**：每个人的使用场景不同，配置方法仅供参考。
**不承诺效果**：工具配置因人而异，不保证按本文操作一定能达到相同效果。