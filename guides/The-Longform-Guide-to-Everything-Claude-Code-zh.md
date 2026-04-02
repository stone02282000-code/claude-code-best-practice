# Claude Code 完全指南（长篇版）

> **原文地址**: [https://x.com/affaanmustafa/status/2014040193557471352](https://x.com/affaanmustafa/status/2014040193557471352)
>
> **相关文章**: [Claude Code 完全指南（简版）](./README.md)
>
> **作者配置仓库**: [https://github.com/affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)

---

在"Claude Code 完全指南（简版）"中，我介绍了基础配置：技能和命令、钩子、子代理、MCP、插件，以及构成有效 Claude Code 工作流程骨架的配置模式。那是一份配置指南和基础设施。

这篇长篇指南将深入探讨区分高效会话和低效会话的技术。如果你还没有阅读**[简版指南](https://x.com/affaanmustafa/status/2012378465664745795)**，请先回去完成你的配置。以下内容假设你已经配置好了技能、代理、钩子和 MCP。

本文的主题包括：Token 经济学、记忆持久化、验证模式、并行化策略，以及构建可复用工作流程的复合效应。这些是我在 10 多个月的日常使用中精炼出的模式，它们决定了你是在第一个小时就被上下文腐烂困扰，还是能够持续高效工作数小时。

简版和长篇文章中涵盖的所有内容都可以在 GitHub 上找到：**[everything-claude-code](https://github.com/affaan-m?tab=repositories)**

---

## 上下文和记忆管理

跨会话共享记忆的最佳方式是：创建一个技能或命令，用于总结和检查进度，然后保存到 `.claude` 文件夹中的 `.tmp` 文件，并在会话期间持续追加，直到会话结束。第二天可以使用该文件作为上下文，继续之前的工作。为每个会话创建新文件，这样就不会将旧上下文污染到新工作中。最终你会积累一大堆这样的会话日志——只需备份到有意义的地方或删除不需要的会话对话。

Claude 会创建一个总结当前状态的文件。查看它，如果需要可以要求修改，然后重新开始。对于新对话，只需提供文件路径。当你遇到上下文限制需要继续复杂工作时特别有用。这些文件应该包含：哪些方法有效（有证据验证）、哪些尝试过的方法无效、哪些方法尚未尝试以及还剩下什么要做。

![会话存储示例](images/longform/session-storage.png)

*会话存储示例 -> [https://github.com/affaan-m/everything-claude-code/tree/main/examples/sessions](https://github.com/affaan-m/everything-claude-code/tree/main/examples/sessions)*

**策略性清除上下文：**

一旦你设定好计划并清除了上下文（这现在是 Claude Code 计划模式中的默认选项），你就可以按照计划工作了。当你积累了大量与执行无关的探索性上下文时，这非常有用。对于策略性压缩，禁用自动压缩。在逻辑间隔处手动压缩，或创建一个技能为你执行此操作或在满足某些定义的条件时建议压缩。

---

**[策略性压缩技能](https://github.com/affaan-m/everything-claude-code/tree/main/skills/strategic-compact)（直接链接）：**

（嵌入供快速参考）

```bash
#!/bin/bash
# 策略性压缩建议器
# 在 PreToolUse 上运行，在逻辑间隔处建议手动压缩
#
# 为什么选择手动而非自动压缩：
# - 自动压缩在任意时间点发生，通常在任务进行中
# - 策略性压缩在逻辑阶段保持上下文
# - 在探索之后、执行之前压缩
# - 在完成里程碑之后、开始下一个之前压缩

COUNTER_FILE="/tmp/claude-tool-count-$"
THRESHOLD=${COMPACT_THRESHOLD:-50}

# 初始化或增加计数器
if [ -f "$COUNTER_FILE" ]; then
  count=$(cat "$COUNTER_FILE")
  count=$((count + 1))
  echo "$count" > "$COUNTER_FILE"
else
  echo "1" > "$COUNTER_FILE"
  count=1
fi

# 达到阈值工具调用后建议压缩
if [ "$count" -eq "$THRESHOLD" ]; then
  echo "[StrategicCompact] 已达到 $THRESHOLD 次工具调用 - 如果正在转换阶段，考虑使用 /compact" >&2
fi
```

将其挂钩到 Edit/Write 操作的 PreToolUse 上——当你积累了足够多可能需要压缩的上下文时，它会提醒你。

---

**高级：动态系统提示注入**

我正在试用的一个模式是：不仅仅将所有内容放在 CLAUDE.md（用户范围）或 `.claude/rules/`（项目范围）中（这些每次会话都会加载），而是使用 CLI 标志动态注入上下文。

```bash
claude --system-prompt "$(cat memory.md)"
```

这让你可以更精确地控制何时加载什么上下文。你可以根据正在进行的工作为每个会话注入不同的上下文。

**为什么这比 @ 文件引用更重要：**

当你使用 `@memory.md` 或将内容放在 `.claude/rules/` 中时，Claude 在对话期间通过 Read 工具读取它——它作为工具输出进来。当你使用 `--system-prompt` 时，内容在对话开始之前被注入到实际的系统提示中。

区别在于指令层级。系统提示内容的权威性高于用户消息，而用户消息的权威性高于工具结果。对于大多数日常工作，这种差异是边际的。但对于严格的行为规则、项目特定的约束，或者你绝对需要 Claude 优先考虑的上下文——系统提示注入确保它得到适当的权重。

**实际配置：**

一个有效的方法是将 `.claude/rules/` 用于基准项目规则，然后为场景特定的上下文设置 CLI 别名，可以在它们之间切换：

```bash
# 日常开发
alias claude-dev='claude --system-prompt "$(cat ~/.claude/contexts/dev.md)"'

# PR 审查模式
alias claude-review='claude --system-prompt "$(cat ~/.claude/contexts/review.md)"'

# 研究/探索模式
alias claude-research='claude --system-prompt "$(cat ~/.claude/contexts/research.md)"'
```

**[系统提示上下文示例文件](https://github.com/affaan-m/everything-claude-code/tree/main/contexts)（直接链接）：**

- dev.md 专注于实现
- review.md 专注于代码质量/安全
- research.md 专注于行动前的探索

同样，对于大多数事情，使用 `.claude/rules/context1.md` 和直接将 `context1.md` 追加到系统提示之间的区别是边际的。CLI 方法更快（无需工具调用）、更可靠（系统级权威），且略微更节省 Token。但这是一个小优化，对许多人来说，这比它值得的开销更大。

---

**高级：记忆持久化钩子**

有一些大多数人不知道或知道但不真正利用的钩子，它们有助于记忆：

```
会话 1                                   会话 2
─────────                              ─────────

[开始]                                  [开始]
   │                                      │
   ▼                                      ▼
┌──────────────┐                    ┌──────────────┐
│ SessionStart │ ◄─── 读取 ─────── │ SessionStart │◄── 加载之前的
│    钩子      │     暂无内容       │    钩子      │    上下文
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       ▼                                   ▼
   [工作中]                            [工作中]
       │                               (已了解)
       ▼                                   │
┌──────────────┐                           ▼
│  PreCompact  │──► 在总结前           [继续...]
│    钩子      │    保存状态
└──────┬───────┘
       │
       ▼
   [已压缩]
       │
       ▼
┌──────────────┐
│  Stop 钩子   │──► 持久化到 ──────────►
│ (会话结束)   │    ~/.claude/sessions/
└──────────────┘
```

- **PreCompact 钩子：** 在上下文压缩发生之前，将重要状态保存到文件
- **SessionComplete 钩子：** 在会话结束时，将学习成果持久化到文件
- **SessionStart 钩子：** 在新会话开始时，自动加载之前的上下文

**[记忆持久化钩子](https://github.com/affaan-m/everything-claude-code/tree/main/hooks/memory-persistence/)（直接链接）：**

（嵌入供快速参考）

```json
{
  "hooks": {
    "PreCompact": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/hooks/memory-persistence/pre-compact.sh"
      }]
    }],
    "SessionStart": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/hooks/memory-persistence/session-start.sh"
      }]
    }],
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/hooks/memory-persistence/session-end.sh"
      }]
    }]
  }
}
```

这些钩子的作用：

- **pre-compact.sh：** 记录压缩事件，用压缩时间戳更新活动会话文件
- **session-start.sh：** 检查最近的会话文件（最近 7 天），通知可用的上下文和已学习的技能
- **session-end.sh：** 创建/更新每日会话文件模板，跟踪开始/结束时间

将这些链接在一起，无需手动干预即可实现跨会话的持续记忆。这建立在第 1 篇文章中的钩子类型（PreToolUse、PostToolUse、Stop）之上，但专门针对会话生命周期。

---

## 持续学习 / 记忆

我们谈到了以更新代码地图形式的持续记忆更新，但这也适用于其他事情，比如从错误中学习。如果你不得不多次重复提示，而 Claude 遇到了同样的问题或给了你之前听过的回应，这就适用于你。

很可能你需要发送第二个提示来"重新引导"和校准 Claude 的方向。这适用于任何此类场景——这些模式必须追加到技能中。

现在你可以通过简单地告诉 Claude 记住它或将其添加到你的规则中来自动完成此操作，或者你可以拥有一个完全执行此操作的技能。

**问题：** 浪费的 Token、浪费的上下文、浪费的时间，当你沮丧地对 Claude 大喊不要做你在之前会话中已经告诉它不要做的事情时，你的皮质醇会飙升。

**解决方案：** 当 Claude Code 发现一些非平凡的东西——调试技术、变通方法、一些项目特定的模式——它将该知识保存为新技能。下次出现类似问题时，该技能会自动加载。

---

**[持续学习技能（直接链接）：](https://github.com/affaan-m/everything-claude-code/tree/main/skills/continuous-learning)**

为什么我使用 **Stop 钩子**而不是 **UserPromptSubmit**？**UserPromptSubmit** 在你发送的每条消息上运行——这是很大的开销，为每个提示增加延迟，坦率地说对于此目的来说过度了。Stop 在会话结束时只运行一次——轻量级，不会在会话期间拖慢你，并且评估完整的会话而不是零散的部分。

**安装：**

```bash
# 克隆到技能文件夹
git clone https://github.com/affaan-m/everything-claude-code.git ~/.claude/skills/everything-claude-code

# 或者只获取 continuous-learning 技能
mkdir -p ~/.claude/skills/continuous-learning
curl -sL https://raw.githubusercontent.com/affaan-m/everything-claude-code/main/skills/continuous-learning/evaluate-session.sh > ~/.claude/skills/continuous-learning/evaluate-session.sh
chmod +x ~/.claude/skills/continuous-learning/evaluate-session.sh
```

**[钩子配置](https://github.com/affaan-m/everything-claude-code/tree/main/hooks)（直接链接）：**

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/skills/continuous-learning/evaluate-session.sh"
          }
        ]
      }
    ]
  }
}
```

这使用 **Stop 钩子**在每个提示上运行激活器脚本，评估会话中值得提取的知识。该技能也可以通过语义匹配激活，但钩子确保一致的评估。

**Stop 钩子**在你的会话结束时触发——脚本分析会话中值得提取的模式（错误解决方案、调试技术、变通方法、项目特定模式等）并将它们保存为 `~/.claude/skills/learned/` 中的可复用技能。

**使用 /learn 手动提取：**

你不必等到会话结束。该仓库还包含一个 `/learn` 命令，你可以在刚刚解决了一些非平凡问题时在会话中运行。它会提示你立即提取模式，起草技能文件，并在保存前请求确认。参见**[这里](https://github.com/affaan-m/everything-claude-code/tree/main/commands/learn.md)**。

**会话日志模式：**

该技能期望 `.tmp` 文件中的会话日志。模式是：`~/.claude/sessions/YYYY-MM-DD-topic.tmp`——每个会话一个文件，包含当前状态、已完成项目、阻碍因素、关键决策和下一个会话的上下文。示例会话文件在仓库的 **[examples/sessions/](https://github.com/affaan-m/everything-claude-code/tree/main/examples/sessions)** 中。

---

**其他自我改进记忆模式：**

[@RLanceMartin](https://x.com/@RLanceMartin) 的一种方法涉及对会话日志进行反思以提炼用户偏好——本质上是建立一个关于什么有效什么无效的"日记"。每次会话后，反思代理提取什么进展顺利、什么失败了、你做了什么修正。这些学习成果更新一个在后续会话中加载的记忆文件。

[@alexhillman](https://x.com/@alexhillman) 的另一种方法是让系统每 15 分钟主动建议改进，而不是等你注意到模式。代理审查最近的交互，提议记忆更新，你批准或拒绝。随着时间推移，它从你的批准模式中学习。

---

## Token 优化

我收到了很多来自价格敏感消费者或作为高级用户经常遇到限制问题的人的问题。在 Token 优化方面，你可以做一些技巧。

**主要策略：子代理架构**

主要是优化你使用的工具和子代理架构，设计为将最便宜且足以完成任务的模型委派出去以减少浪费。你有几个选择——你可以尝试试错并随时调整。一旦你了解了什么是什么，你就可以将任务委派给 Haiku 还是 Sonnet 还是 Opus。

**基准测试方法（更复杂）：**

另一种更复杂的方法是让 Claude 设置一个基准测试，其中你有一个具有明确定义目标和任务以及明确定义计划的仓库。在每个 git worktree 中，让所有子代理都使用一个模型。当任务完成时记录——最好在你的计划和任务中。你必须至少使用每个子代理一次。

一旦你完成了一轮完整的测试并且任务已从你的 Claude 计划中勾选，停下来审计进度。你可以通过比较差异、创建在所有 worktree 中统一的单元、集成和 E2E 测试来做到这一点。这将根据通过的用例与失败的用例给你一个数字基准。如果所有测试都通过，你需要添加更多测试边缘用例或增加测试的复杂性。这可能值得也可能不值得，取决于这对你来说有多重要。

**模型选择快速参考：**

![模型选择表](images/longform/model-selection.jpg)

*各种常见任务上子代理的假设设置及选择背后的推理*

90% 的编码任务默认使用 Sonnet。当第一次尝试失败、任务跨越 5 个以上文件、架构决策或安全关键代码时升级到 Opus。当任务重复、指令非常明确或作为多代理设置中的"工人"时降级到 Haiku。坦率地说，Sonnet 4.5 目前处于一个尴尬的位置，每百万输入 Token 3 美元，每百万输出 Token 15 美元，与 Opus 相比成本节省约 66.7%，绝对来说这是很好的节省，但相对来说对大多数人来说或多或少是微不足道的。Haiku 和 Opus 组合最有意义，因为 Haiku vs Opus 是 5 倍的成本差异，而与 Sonnet 相比是 1.67 倍的价格差异。

![定价表](images/longform/pricing.jpg)

*来源：[https://platform.claude.com/docs/en/about-claude/pricing](https://platform.claude.com/docs/en/about-claude/pricing)*

在你的代理定义中，指定模型：

```yaml
---
name: quick-search
description: Fast file search
tools: Glob, Grep
model: haiku # 便宜且快速
---
```

**工具特定优化：**

考虑 Claude 最频繁调用的工具。例如，用 mgrep 替换 grep——在各种任务上，与传统 grep 或 ripgrep（Claude 默认使用的）相比，Token 减少平均约一半。

![mgrep 基准测试](images/longform/mgrep-benchmark.jpg)

*来源：[https://github.com/mixedbread-ai/mgrep/blob/main/README.md](https://github.com/mixedbread-ai/mgrep/blob/main/README.md)*

**后台进程：**

当适用时，如果你不需要 Claude 处理整个输出并直接实时流式传输，在 Claude 外部运行后台进程。这可以通过 tmux 轻松实现（参见**[简版指南](https://x.com/affaanmustafa/status/2012378465664745795)**和**[Tmux 命令参考（直接链接）](https://tmuxcheatsheet.com/)**）。获取终端输出，要么总结它，要么只复制你需要的部分。这将节省大量输入 Token，这是成本的主要来源——Opus 4.5 每百万 Token 5 美元，输出每百万 Token 25 美元。

**模块化代码库的好处：**

拥有一个更模块化的代码库，具有可复用的工具函数、函数、钩子等——主文件在几百行而不是几千行——这有助于 Token 优化成本，也有助于第一次就正确完成任务，这两者相关。如果你必须多次提示 Claude，你就在消耗 Token，尤其是当它反复读取非常长的文件时。你会注意到它必须进行很多工具调用才能完成读取文件。中间过程中，它会告诉你文件很长，它将继续读取。在此过程中的某个地方，Claude 可能会丢失一些信息。此外，停止和重新读取会消耗额外的 Token。通过拥有更模块化的代码库可以避免这种情况。示例如下 ->

```
root/
├── docs/                   # 全局文档
├── scripts/                # CI/CD 和构建脚本
├── src/
│   ├── apps/               # 入口点（API、CLI、Workers）
│   │   ├── api-gateway/    # 将请求路由到模块
│   │   └── cron-jobs/
│   │
│   ├── modules/            # 系统核心
│   │   ├── ordering/       # 自包含的"订单"模块
│   │   │   ├── api/        # 其他模块的公共接口
│   │   │   ├── domain/     # 业务逻辑和实体（纯净的）
│   │   │   ├── infrastructure/ # 数据库、外部客户端、仓库
│   │   │   ├── use-cases/  # 应用逻辑（编排）
│   │   │   └── tests/      # 单元和集成测试
│   │   │
│   │   ├── catalog/        # 自包含的"目录"模块
│   │   │   ├── domain/
│   │   │   └── ...
│   │   │
│   │   └── identity/       # 自包含的"认证/用户"模块
│   │       ├── domain/
│   │       └── ...
│   │
│   ├── shared/             # 每个模块使用的代码
│   │   ├── kernel/         # 基类（Entity、ValueObject）
│   │   ├── events/         # 全局事件总线定义
│   │   └── utils/          # 深度通用助手
│   │
│   └── main.ts             # 应用程序引导
├── tests/                  # 端到端（E2E）全局测试
├── package.json
└── README.md
```

**精简的代码库 = 更便宜的 Token：**

这可能很明显，但你的代码库越精简，Token 成本就越低。通过使用技能持续清理代码库来识别死代码至关重要，使用技能和命令进行重构。此外在某些时候，我喜欢浏览整个代码库，寻找突出或看起来重复的内容，手动拼凑上下文，然后将其与重构技能和死代码技能一起提供给 Claude。

---

**系统提示精简（高级）：**

对于真正注重成本的人：Claude Code 的系统提示占用约 18k Token（约 200k 上下文的 9%）。通过补丁可以将其减少到约 10k Token，节省约 7,300 Token（静态开销的 41%）。如果你想走这条路，参见 YK 的**[系统提示补丁](https://agenticcoding.substack.com/p/32-claude-code-tips-from-basics-to)**，但我个人不这样做。

---

## 验证循环和评估

评估和工具调优——根据项目，你会想要使用某种形式的可观察性和标准化。

**可观察性方法：**

一种方法是让 tmux 进程挂钩到跟踪思考流和技能触发时的输出。另一种方法是有一个 PostToolUse 钩子，记录 Claude 具体执行了什么以及确切的更改和输出是什么。

**基准测试工作流程：**

将其与不使用技能询问同样的事情并检查输出差异进行比较，以基准测试相对性能：

```
                    [相同任务]
                         │
            ┌────────────┴────────────┐
            ▼                         ▼
    ┌───────────────┐         ┌───────────────┐
    │  Worktree A   │         │  Worktree B   │
    │   有技能      │         │   无技能      │
    └───────┬───────┘         └───────┬───────┘
            │                         │
            ▼                         ▼
       [输出 A]                   [输出 B]
            │                         │
            └──────────┬──────────────┘
                       ▼
                  [git diff]
                       │
                       ▼
              ┌────────────────┐
              │ 比较日志、     │
              │ Token 使用量、 │
              │ 输出质量       │
              └────────────────┘
```

分叉对话，在其中一个中启动一个没有技能的新 worktree，最后拉出差异，查看记录了什么。这与持续学习和记忆部分相关。

**评估模式类型：**

更高级的评估和循环协议在这里介入。分为基于检查点的评估和基于 RL 任务的持续评估。

```
基于检查点                               持续
─────────────────                        ──────────

  [任务 1]                                 [工作]
     │                                        │
     ▼                                        ▼
  ┌─────────┐                            ┌─────────┐
  │检查点   │◄── 验证                    │ 定时器/  │
  │   #1    │    条件                    │  更改   │
  └────┬────┘                            └────┬────┘
       │ 通过？                               │
   ┌───┴───┐                                  ▼
   │       │                            ┌──────────┐
  是      否 ──► 修复 ──┐               │运行测试  │
   │              │    │                │  + Lint  │
   ▼              └────┘                └────┬─────┘
  [任务 2]                                   │
     │                                  ┌────┴────┐
     ▼                                  │         │
  ┌─────────┐                          通过      失败
  │检查点   │                          │         │
  │   #2    │                           ▼         ▼
  └────┬────┘                        [继续]    [停止并修复]
       │                                          │
      ...                                    └────┘

最适合：线性工作流程                    最适合：长时间会话
具有明确的里程碑                        探索性重构
```

**基于检查点的评估：**

- 在工作流程中设置明确的检查点
- 在每个检查点根据定义的条件进行验证
- 如果验证失败，Claude 必须在继续之前修复
- 适合具有明确阶段的线性工作流程

**持续评估：**

- 每 N 分钟或在重大更改后运行
- 完整的测试套件、构建状态、lint
- 立即报告回归
- 在继续之前停止并修复
- 适合长时间运行的会话

决定因素是你的工作性质。基于检查点的适用于具有明确阶段的功能实现。持续适用于探索性重构或维护，其中你没有明确的里程碑。

我会说，通过一些干预，验证方法足以避免大多数技术债务。让 Claude 在完成任务后通过运行技能和 PostToolUse 钩子进行验证有助于此。持续的代码地图更新也有帮助，因为它记录了更改以及代码地图随时间的演变，作为仓库本身之外的真相来源。通过严格的规则，Claude 将避免创建随机的 .md 文件弄乱一切，以及为类似代码创建重复文件并留下死代码的荒地。

**[评分器类型（来自 Anthropic - 直接链接）：](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)**

**基于代码的评分器：** 字符串匹配、二进制测试、静态分析、结果验证。快速、便宜、客观，但对有效变体脆弱。

**基于模型的评分器：** 评分标准打分、自然语言断言、成对比较。灵活且处理细微差别，但非确定性且更昂贵。

**人工评分器：** 主题专家审查、众包判断、抽样检查。黄金标准质量，但昂贵且缓慢。

**关键指标：**

```
pass@k：k 次尝试中至少有一次成功
        ┌─────────────────────────────────────┐
        │  k=1: 70%  k=3: 91%  k=5: 97%      │
        │  更高的 k = 更高的成功概率          │
        └─────────────────────────────────────┘

pass^k：所有 k 次尝试都必须成功
        ┌─────────────────────────────────────┐
        │  k=1: 70%  k=3: 34%  k=5: 17%      │
        │  更高的 k = 更难（一致性）          │
        └─────────────────────────────────────┘
```

当你只需要它工作并且任何验证反馈就足够时使用 **pass@k**。当一致性至关重要并且你需要接近确定性的输出一致性（在结果/质量/风格方面）时使用 **pass^k**。

**构建评估路线图（来自同一 Anthropic 指南）：**

1. 尽早开始——来自真实失败的 20-50 个简单任务
2. 将用户报告的失败转换为测试用例
3. 编写明确的任务——两位专家应该得出相同的结论
4. 构建平衡的问题集——测试行为应该发生和不应该发生的情况
5. 构建健壮的工具——每次试验从干净环境开始
6. 评分代理产生的内容，而不是它采取的路径
7. 阅读许多试验的记录
8. 监控饱和——100% 通过率意味着添加更多测试

---

## 并行化

在多 Claude 终端设置中分叉对话时，确保为分叉和原始对话中的操作明确定义范围。在代码更改方面力求最小重叠。选择彼此正交的任务以防止干扰的可能性。

**我偏好的模式：**

就我个人而言，我更喜欢主聊天专注于代码更改，而我做的分叉是用于关于代码库及其当前状态的问题，或用于研究外部服务，如拉取文档、在 GitHub 上搜索有助于任务的适用开源仓库，或其他有用的一般研究。

**关于任意终端数量：**

Boris [@bcherny](https://x.com/@bcherny)（创建 Claude Code 的传奇人物）有一些关于并行化的建议，我部分同意也部分不同意。他建议过类似在本地运行 5 个 Claude 实例和 5 个上游。我建议不要设置这样任意的终端数量。添加终端和添加实例应该出于真正的必要性和目的。如果你可以使用脚本完成该任务，就使用脚本。如果你可以留在主聊天中并让 Claude 在 tmux 中启动一个实例并以那种方式在单独的终端中流式传输，就那样做。

![Boris 的并行 Claude 设置](images/longform/boris-parallel.jpg)

你的目标真的应该是：用最小可行的并行化量完成多少工作。

对于大多数新手，我甚至建议在你掌握只运行单个实例并在其中管理一切的诀窍之前远离并行化。我不是在提倡给自己设障碍——我是说要小心。大多数时候，即使是我也只总共使用 4 个左右的终端。我发现我通常能够用 2 或 3 个 Claude 实例打开完成大多数事情。

**扩展实例时：**

如果你要开始扩展你的实例，并且你有多个 Claude 实例在处理彼此重叠的代码，那么使用 git worktrees 并为每个制定非常明确的计划至关重要。此外，为了在恢复会话时不会对哪个 git worktree 用于什么感到困惑或迷失（除了树的名称之外），使用 `/rename <名称>` 命名所有你的聊天。

**用于并行实例的 Git Worktrees：**

```bash
# 为并行工作创建 worktrees
git worktree add ../project-feature-a feature-a
git worktree add ../project-feature-b feature-b
git worktree add ../project-refactor refactor-branch

# 每个 worktree 获得自己的 Claude 实例
cd ../project-feature-a && claude
```

**好处：**

- 实例之间没有 git 冲突
- 每个都有干净的工作目录
- 易于比较输出
- 可以跨不同方法对同一任务进行基准测试

**级联方法：**

运行多个 Claude Code 实例时，使用"级联"模式组织：

- 在右侧的新标签页中打开新任务
- 从左到右、从旧到新扫描
- 保持一致的方向流
- 根据需要检查特定任务
- 一次最多关注 3-4 个任务——超过这个数量，心理开销增加得比生产力更快

---

## 基础工作

从头开始时，实际的基础非常重要。这应该很明显，但随着代码库复杂性和规模的增加，技术债务也会增加。管理它非常重要，如果你遵循一些规则就不会那么困难。除了为手头的项目有效地设置 Claude（参见简版指南）。

**双实例启动模式：**

对于我自己的工作流程管理（不是必需的但有帮助），我喜欢用 2 个打开的 Claude 实例启动一个空仓库。

**实例 1：脚手架代理**

- 将铺设脚手架和基础工作
- 创建项目结构
- 设置配置（CLAUDE.md、规则、代理——简版指南中的所有内容）
- 建立约定
- 准备好骨架

**实例 2：深度研究代理**

- 连接到所有你的服务、网络搜索等
- 创建详细的 PRD
- 创建架构 mermaid 图
- 使用实际文档中的实际片段编译参考资料

![双终端设置](images/longform/two-terminal-setup.jpg)

*启动设置：左侧终端用于编码，右侧终端用于问题 - 使用 /rename 和 /fork。*

你最少需要的就可以开始了——这比每次使用 Context7 或喂给它链接让它抓取或使用 Firecrawl MCP 网站更快。当你已经深入某事并且 Claude 明显语法错误或使用过时的函数或端点时，所有这些都有效。

**llms.txt 模式：**

如果可用，你可以在许多文档参考上通过在到达其文档页面后执行 `/llms.txt` 找到 llms.txt。这是一个例子：[https://www.helius.dev/docs/llms.txt](https://www.helius.dev/docs/llms.txt)

这给你一个干净的、LLM 优化的文档版本，你可以直接提供给 Claude。

**理念：构建可复用模式**

[@omarsar0](https://x.com/@omarsar0) 的一个见解我完全认同："早期，我花时间构建可复用的工作流程/模式。构建起来很繁琐，但随着模型和代理工具的改进，这产生了疯狂的复合效应。"

**值得投资的：**

- 子代理（简版指南）
- 技能（简版指南）
- 命令（简版指南）
- 规划模式
- MCP 工具（简版指南）
- 上下文工程模式

**为什么它会复合（[@omarsar0](https://x.com/@omarsar0)）：** "最好的部分是所有这些工作流程都可以转移到其他代理，如 Codex。"一旦构建，它们在模型升级中都有效。对模式的投资 > 对特定模型技巧的投资。

---

## 代理和子代理的最佳实践

在简版指南中，我列出了子代理结构——规划器、架构师、tdd-guide、code-reviewer 等。在这部分，我们关注编排和执行层。

**子代理上下文问题：**

子代理的存在是为了通过返回摘要而不是倾倒所有内容来节省上下文。但编排器拥有子代理缺乏的语义上下文。子代理只知道字面查询，而不知道请求背后的目的/推理。摘要经常遗漏关键细节。

[@PerceptualPeak](https://x.com/@PerceptualPeak) 的类比："你的老板派你去开会并要求一个摘要。你回来给他概述。十有八九，他会有后续问题。你的摘要不会包括他需要的一切，因为你没有他拥有的隐含上下文。"

**迭代检索模式：**

```
┌─────────────────┐
│   编排器        │
│  (有上下文)     │
└────────┬────────┘
         │ 带查询 + 目标派发
         ▼
┌─────────────────┐
│   子代理        │
│ (缺乏上下文)    │
└────────┬────────┘
         │ 返回摘要
         ▼
┌─────────────────┐      ┌─────────────┐
│   评估          │─否──►│  后续问题   │
│   足够？        │      │             │
└────────┬────────┘      └──────┬──────┘
         │ 是                   │
         ▼                      │ 子代理
    [接受]                获取答案
                                │
         ◄──────────────────────┘
              (最多 3 个周期)
```

为了解决这个问题，让编排器：

- 评估每个子代理的返回
- 在接受之前提出后续问题
- 子代理回到源头，获取答案，返回
- 循环直到足够（最多 3 个周期以防止无限循环）

**传递目标上下文，而不仅仅是查询。** 当派发子代理时，同时包括具体查询和更广泛的目标。这有助于子代理优先确定在其摘要中包含什么。

**模式：具有顺序阶段的编排器**

```markdown
阶段 1：研究（使用 Explore 代理）

- 收集上下文
- 识别模式
- 输出：research-summary.md

阶段 2：规划（使用 planner 代理）

- 读取 research-summary.md
- 创建实施计划
- 输出：plan.md

阶段 3：实施（使用 tdd-guide 代理）

- 读取 plan.md
- 先写测试
- 实现代码
- 输出：代码更改

阶段 4：审查（使用 code-reviewer 代理）

- 审查所有更改
- 输出：review-comments.md

阶段 5：验证（如果需要使用 build-error-resolver）

- 运行测试
- 修复问题
- 输出：完成或循环回去
```

**关键规则：**

1. 每个代理获得一个明确的输入并产生一个明确的输出
2. 输出成为下一阶段的输入
3. 永远不要跳过阶段——每个阶段都有价值
4. 在代理之间使用 `/clear` 以保持上下文新鲜
5. 将中间输出存储在文件中（不仅仅是内存中）

**代理抽象层级（来自 [@menhguin](https://x.com/@menhguin)）：**

**第 1 层：直接增益（易于使用）**

- **子代理** - 防止上下文腐烂和临时专业化的直接增益。只有多代理一半有用但复杂性少得多
- **元提示** - "我花 3 分钟提示一个 20 分钟的任务。"直接增益 - 提高稳定性并检查假设
- **开始时多问用户** - 通常是增益，尽管你必须在计划模式中回答问题

**第 2 层：高技能门槛（更难用好）**

- **长时间运行的代理** - 需要理解 15 分钟任务 vs 1.5 小时 vs 4 小时任务的形状和权衡。需要一些调整，显然是非常长的试错过程
- **并行多代理** - 非常高的方差，只在高度复杂或分段良好的任务上有用。"如果 2 个任务需要 10 分钟，而你花任意时间提示或天哪，合并更改，这是反生产力的"
- **基于角色的多代理** - "模型演变太快，硬编码启发式除非套利非常高。"难以测试
- **计算机使用代理** - 非常早期的范式，需要处理。"你让模型做一些它们一年前绝对不打算做的事情"

要点：从第 1 层模式开始。只有在你掌握了基础知识并有真正需要时才升级到第 2 层。

---

## 技巧和窍门

**某些 MCP 是可替代的，会释放你的上下文窗口**

这是如何做到的。

对于版本控制（GitHub）、数据库（Supabase）、部署（Vercel、Railway）等 MCP——大多数这些平台已经有健壮的 CLI，MCP 本质上只是包装它们。MCP 是一个不错的包装器，但它是有代价的。

要让 CLI 功能更像 MCP 而不实际使用 MCP（以及随之而来的减少的上下文窗口），考虑将功能捆绑到技能和命令中。剥离 MCP 暴露的使事情变得容易的工具，并将它们变成命令。

示例：不是让 GitHub MCP 一直加载，创建一个 `/gh-pr` 命令，用你偏好的选项包装 `gh pr create`。不是让 Supabase MCP 吃上下文，创建直接使用 Supabase CLI 的技能。功能相同，便利性相似，但你的上下文窗口被释放用于实际工作。

这与我收到的一些其他问题相关。自从我发布原始文章以来的过去几天里，Boris 和 Claude Code 团队在内存管理和优化方面取得了很大进展，主要是 MCP 的延迟加载，这样它们就不会从一开始就吃你的窗口。以前我会建议在可以的地方将 MCP 转换为技能，通过以下两种方式之一将功能卸载到执行 MCP：在那时启用它（不太理想，因为你需要离开并恢复会话）或者让技能使用 MCP 的 CLI 类似物（如果存在）并让技能成为它的包装器——本质上让它充当伪 MCP。

有了**延迟加载**，上下文窗口问题基本解决了。但 Token 使用量和成本并没有以同样的方式解决。CLI + 技能方法仍然是一种 Token 优化方法，可能与使用 MCP 的效果相当或接近。此外，你可以通过 CLI 而不是在上下文中运行 MCP 操作，这大大减少了 Token 使用量，对于繁重的 MCP 操作如数据库查询或部署特别有用。

---

## 视频？

正如你建议的，我认为这篇文章配合其他一些问题需要一个视频来配合这篇文章，涵盖这些内容。

**涵盖一个使用两篇文章策略的端到端项目：**

- 使用简版指南中的配置进行完整项目设置
- 这篇长篇指南中的高级技术实践
- 实时 Token 优化
- 验证循环实践
- 跨会话的记忆管理
- 双实例启动模式
- 使用 git worktrees 的并行工作流程
- 实际工作流程的截图和录制

我会看看能做什么。

---

## 参考资料

- [Anthropic：揭秘 AI 代理的评估](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)（2026 年 1 月）
- Anthropic："Claude Code 最佳实践"（2025 年 4 月）
- Fireworks AI："使用 Claude Code 的评估驱动开发"（2025 年 8 月）
- [YK：32 个 Claude Code 技巧](https://agenticcoding.substack.com/p/32-claude-code-tips-from-basics-to)（2025 年 12 月）
- Addy Osmani："我进入 2026 年的 LLM 编码工作流程"
- [@PerceptualPeak](https://x.com/@PerceptualPeak)：子代理上下文协商
- [@menhguin](https://x.com/@menhguin)：代理抽象层级
- [@omarsar0](https://x.com/@omarsar0)：复合效应哲学
- [RLanceMartin：会话反思模式](https://rlancemartin.github.io/2025/12/01/claude_diary/)
- [@alexhillman](https://x.com/@alexhillman)：自我改进记忆系统
