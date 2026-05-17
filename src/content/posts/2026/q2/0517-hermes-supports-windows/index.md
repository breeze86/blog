---
title: Windows 党狂喜？先别急着高兴，有几件事你得知道
published: 2026-05-17
pinned: false
description: "Hermes v0.14.0 原生支持 Windows（early beta），提供 PowerShell 安装脚本、PyPI 包、懒加载冷启动加速。WSL2 老用户建议暂留，观望用户可尝试，但需注意 Dashboard 终端不可用、子进程 rough edges 等已知限制。"
image: "./cover.avif"
tags: ["AI Agent", "windows", "安装部署"]
category: Hermes
draft: false
slug: 20260517-hermes-supports-windows
---

2026 年初，我在 WSL2 里装 Hermes 的时候，路径里带个空格都能报错。当时搜了一圈，发现 Windows 用户基本只有两条路：要么开 WSL2 当 Linux 用，要么干脆放弃。

那几个月，Windows 用户想跑 Hermes，标准流程是先装 WSL2，再配 Ubuntu，然后在 WSL 终端里执行安装命令。配置存在 `~/.hermes` 下，和 Windows 主系统隔着一层文件系统翻译。有时候在 Windows 浏览器里复制一段路径，贴到 WSL 终端里，斜杠方向不对，命令直接报错。这种小事堆多了，挺消耗耐心的。

所以 5 月 16 号看到 v0.14.0 发布，代号 "The Foundation Release"，官方说 "Native Windows ships in early beta" 的时候，我确实精神了一下。但用了几个小时，想泼点冷水——有几件事，装之前最好先知道。

---

## v0.14.0 到底带来了什么

最核心就一条：**原生 Windows 支持**。

不是 WSL 改进，是正经的 native 支持。有完整的 PowerShell 安装脚本，装完数据存在 `%LOCALAPPDATA%\hermes\`，和 WSL 的 `~/.hermes` 互不干扰，两边可以同时跑。官方原话是 "Both can run side-by-side"，这点对想迁移又不想删旧环境的用户挺友好。

另外几个实在的改进：

- **PyPI 包了**：`pip install hermes-agent` 直接装，wheel 里带了 Ink TUI，不用额外折腾。更新也简单，`pip install --upgrade hermes-agent` 就行。PyPI 版本是追踪 tagged releases 的，不是每次 main 分支提交都发版，稳定性相对有保障。
- **懒加载依赖**：通过 lazy imports 延迟导入，冷启动快了约 19 秒。最直观的感受是 `hermes tools` 这个命令，以前 All-Platforms 要等 14 秒，现在 1.5 秒以内出结果。延迟导入的模块包括 Feishu、QQ、Yuanbao、Teams、Google Chat 这些适配器，还有 `fal_client`、`google-cloud`、`httpx` 这些客户端。`models.dev` 还用了 disk-cache-first 策略，进一步省时间。
- **分层安装回退**：装 `.[all]` 失败会自动降级到 `[messaging,dashboard,ext]`，再不行就 `[messaging]`，最后裸装。网络不稳的时候不用反复手动改命令，这个设计挺务实的。

这些对 Windows 用户来说，确实是能感知的进步。尤其是懒加载，冷启动从十几秒缩到几秒，体验差距很明显。以前点开终端想快速查个工具列表，结果干等十几秒，那种烦躁感用过的人都懂。

---

## 两类用户的不同反应

### WSL 老用户

如果你已经在 WSL2 里跑顺了，我的建议是：**先别动**。

官方文档里 WSL2 仍然被标为 "most battle-tested setup"。v0.14.0 的 native Windows 是 early beta，而 WSL2 路线已经磨合了将近三个月。Dashboard 里的 `/chat` 终端面板在 native 下不可用（需要 POSIX PTY），但在 WSL2 里正常。为了一点安装便利切过去，结果常用功能少一个，不划算。

还有一个现实问题：你的历史数据在 WSL 的 `~/.hermes` 里，native 用的是 `%LOCALAPPDATA%\hermes\`，两边不互通。想迁移得手动搬配置，beta 阶段犯不着折腾这一趟。而且你 WSL 里配好的网关、模型、工具链，native 里要重新来一遍，时间成本不低。

当然，如果你就是好奇想试试，可以同时装，两边互不干扰。只是别指望 native 现阶段能完全替代 WSL2 的体验。

### 观望用户

如果你之前因为 "要装 WSL" 一直没碰 Hermes，现在可以试试了。

`pip install hermes-agent` 装完，跑 `hermes setup` 走向导，选模型、配工具、设网关，一步一步来。配置存在 `%LOCALAPPDATA%\hermes\`，secrets 写 `.env`，其他设置写 `config.yaml`，路径比 WSL 里直观一些。至少你不用再查 "WSL 怎么共享 Windows 的文件系统" 这种基础问题了。

安装门槛确实低了，不用先学 WSL 那套东西。但要有心理准备：beta 就是 beta，下面这几个坑是官方自己标出来的。

---

## 简要部署步骤

如果你决定尝试，确认过的步骤如下：

```powershell
# 安装
pip install hermes-agent

# 初始化向导
hermes setup

# 查看已安装工具
hermes tools

# 配置模型提供商
hermes model

# 配置消息网关（可选）
hermes gateway setup
```

注：`hermes setup` 是向导式命令，会引导你完成基础配置。官方文档中未检索到 `hermes init` 命令，初始化就用 `setup` 即可。

装完之后，数据在 `%LOCALAPPDATA%\hermes\` 下，代码安装位置是 `%LOCALAPPDATA%\hermes\hermes-agent`，`hermes` 二进制会自动加到 User PATH。开机自启可以用 `schtasks` 配，和 WSL 里的 systemd 对应。

---

## 几个需要知道的坑

| 限制项 | 具体情况 | 影响范围 |
|--------|----------|----------|
| Dashboard `/chat` 终端 | 不可用，需 POSIX PTY | 想用浏览器里嵌终端的 |
| 子进程处理 | rough edges | 调外部工具、脚本时 |
| 路径解析 | quirks | 带空格、特殊字符的路径 |
| 非 ASCII 输出 | 可能乱码 | 中文日志、文件名等 |
| 32 位 Windows | 回退 MinGit，禁用 terminal-tool / agent-browser | 极少数旧设备 |

关于非 ASCII 输出，官方在 `hermes_cli/stdio.py` 里做了 `configure_windows_stdio()`，会自动切控制台到 CP_UTF8，但能不能覆盖所有场景，我还不敢打包票。毕竟 Windows 控制台的编码问题是个历史包袱，不是一行代码能彻底解决的。

另外，v0.14.0 里塞了大量 Windows 专属修复，覆盖 CLI、gateway、TUI、curator、tools 各个模块。修得多，说明之前碎得多——这是好事，但也说明还在早期。你大概率不会遇到严重 bug，但小毛病可能时不时冒出来一个。

---

## 个人建议

如果你纯 Windows、不想碰 WSL，现在可以入，但别当主力生产环境用。子进程和路径的问题，写代码时迟早会撞上。`hermes setup` 跑起来很顺，但真到调外部脚本、处理带中文的文件名时，rough edges 该来还是会来。

如果你已经 WSL2 跑得好好的，等一两个小版本再说。native 支持是趋势，但 "battle-tested" 这四个字，官方目前只给了 WSL2。等那 40 个专属修复再沉淀一轮，等 `/chat` 终端面板在 Windows 下也能用了，再迁不迟。

装完 `hermes setup` 走完向导，突然觉得这个时代变化太快了。三个月前还在 WSL 里改 `.bashrc` 配路径，现在直接装到 Windows 本地了。只是 beta 阶段，慢点换船比较稳。