---
title: OpenRouter 一个错误，让我的 Hermes 罢工了一天
published: 2026-05-12
pinned: false
description: "上游一个数字写错，下游一天白干。AI 时代，连'模型能用'这件事，都不是理所当然的。"
image: ""
tags: ["AI Agent", "OpenRouter"]
category: Hermes
draft: false
slug: 20260512-ai-strike
---

昨晚十一点，我还在用 kimi-k2.6 跑任务，一切正常。



今早出门，手机上想查个东西，给 Hermes 发了条消息。



它回了我一行报错：

> Model kimi-k2.6 has a context window of 32,768 tokens, which is below the minimum 64,000 required by Hermes Agent.



我：想骂人～～～



昨晚还用得好好的，模型没变，配置我也没动过，怎么突然就说模型上下文不满足Hermes的最低要求了？



更气的是，我人在外面，只能看着，啥也改变不了。改配置要开电脑，手机上一点办法没有。



等忙完回家上Hermes的代码仓库查Issues，好家伙，原来我不是个例，好多人都中招了，最后定位到源头：OpenRouter 对 kimi-k2.6 的上下文长度标注错了。官方规格是 256K，OpenRouter 返回 32K。Hermes 每次处理消息前会查询模型元数据，拿到这个错误值，直接拒绝服务。



更坑的是 Hermes 的解析逻辑：它优先信任 OpenRouter 的动态数据，也没有自己硬编码的已知正确值。结果OpenRouter一错，下游全线崩溃，没有任何 fallback 机制。



我真服了。又再一次验证了“世界就是一个巨大的草台班子”



临时解法有两个：

- 手动覆盖 context_length（hermes config set model.context_length 262144）  这个我验证过了，无效

- 切回 kimi-k2.5（OpenRouter 对 k2.5 的标注是正确的）



我选了后者（ 切回 kimi-k2.5 ），秒恢复。



：

这件事

给我提了个醒



我们对 AI 工具链的依赖，远比想象中脆弱。 模型层、路由层、代理层，任何一个环节的数据出错，整个工作流就断了。而且你根本不知道错在哪一层，只能一层层剥。



OpenRouter 已经是个相对成熟的聚合层了，依然会犯这种低级错误。更关键的是，Hermes 作为下游应用，没有对这种错误做防御——它选择了"信任上游"而不是"质疑并自保"。

这不是技术问题，是设计哲学问题：在分布式系统里，"不信任"才是默认姿态。



如果你也遇到同样报错：

- 最快恢复：切回 kimi-k2.5

- 等官方 PR #24156 / #24066 合并修复



GitHub Issue #24268 已标记 P1，应该很快。

 一句话总结 

上游一个数字写错，下游一天白干。AI 时代，连"模型能用"这件事，都不是理所当然的。