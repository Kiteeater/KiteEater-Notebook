# 如何构建优质的 Agent Skill (How to build a good SKILL)

本质上，Skill 还是一种注入到系统上下文的 prompt。`SKILL.md` 的基础概念如脚本、模板和渐进式披露大家都已经耳熟能详了。一个带有 YAML Frontmatter 的 `SKILL.md`，加上可选的脚本、参考文档和模板，在会话启动时会像 Tool 一样注册到 `AGENT.md` 或上下文中。注入的信息主要包含 Skill 的名字和描述 (Description)。

> **💡 深入理解：为什么 Skill 不是靠向量召回的？**
> 
> 为什么 Skills 不是通过语义相似度或 BM25 召回的，而是将 Skills 的 Frontmatter 信息常驻在上下文里呢？
> 这是因为我们希望 Skill 是**原子的、可二次组合的**。我们希望由大模型来决定为了完成用户提出的任务，究竟是使用单一 Skill 还是多个 Skills 组合。
> 如果 Skills 过多，占用太多上下文空间怎么办呢？单独用一个 Flash / Lite 版本的小尺寸大语言模型来做 Skills 挑选即可。

对于 Skill 的编写，值得注意的有以下 5 点：

1. Skill 的名字和 Description
2. Skill 是给 AI 的知识和流程规范、说明书
3. Skill 编写时的渐进式披露很重要
4. Skill 不是一次完成的，需要反复 Eval 迭代
5. 编写 Skill 的实用小妙招

---

## 1. Skill 的 Description 编写

Description 需要清晰地包含以下三点：
- **何时使用这个 Skill**
- **这个 Skill 是用来干什么的**
- **什么时候不要用这个 Skill**（偶尔需要指明）

因为 Skill 的启动完全取决于以上几点，所以当你发现你自己写的 Skill 使用频率不高，就需要多考量一下如何写好 Description。

另外，Skill 的名字重要性也往往会被忽略。不要随便起一个名字，恰当的名字可以在一定程度上为这个 Skill 的使用范围和边界添砖加瓦。

## 2. Skill 是给 AI 看的说明书

Skill 是指导 AI 该如何执行特定任务的，所以 **Skill 的内容一定得由 AI 来编写**，因为 AI 才能更好地理解 AI 的逻辑。而且一定要使用能力强的模型来编写。

## 3. Skill 的渐进式披露 (Progressive-Disclosure)

**首先，什么是渐进式披露？**
实际上这是一种 Context 管理策略。如果 AI 决定调用这个 Skill，首先阅读的就是 `SKILL.md`。如果这个 Skill 流程很长、要控制的细节多、流程分支复杂，那么利用渐进式披露来管理 Context 就显得尤为重要。

**那么如何设计渐进式披露？**
假设你发现 `SKILL.md` 过长，可以直接告诉帮助你写 Skill 的 AI：

> *"当前 `SKILL.md` 太大了，缺乏 progressive-disclosure 机制，但也请不要滥用。"*

比如，你的 Skill 里有类似 `switch...case` 的逻辑分支，并且每一个分支的逻辑都很庞大。AI 就会帮你把这些逻辑拆分成独立的 Markdown 文件，在运行时只有命中条件的一个或多个逻辑分支才会被加载，避免占用冗余上下文。

## 4. 创建完 Skill 不能丢在那不管了

一个好用的 Skill 注定是要经过多次迭代的。这就很像 Agent 版本的 RL（强化学习），**每一轮迭代都是一次人工 RLHF（人类反馈强化学习）**。

如果使用的是 Claude Code，在使用 `/skill-creator` 工具的时候，我们可以要求它进行 Eval (评估)。然后就会发生：

1. Claude 开始自动生成测试用例。
2. 派发多个 Subagent 对当前 Skill 进行 Eval。
3. 在 Eval 完成后，它会生成一个 `Eval Reviewer` 网站用于查看评测结果，让你直观比对使用了这个 Skill 和没有使用的区别。

剩下我们要做的就是在这个网站上进行 Review 和 Comment，以指导进行下一步的优化！

## 5. 编写 Skill 的小妙招

- **引入 Human-in-loop (人机协作)**
  一个好的 Skill 应该适当引入人的参与。你可以在 Skill 的 Prompt 里明确要求它使用 `AskUserQuestion` 工具来与用户进行多轮交互。该工具支持单选、多选、预览单选等交互方式，同时也支持 Step-by-step 式的向导。

- **利用脚本支持工具调用**
  如果想要 Skill 具备特定的系统操作能力，我们完全可以用少量非核心代码作为 Python 或 Node.js 程序写在 Skill 的 `scripts` 目录中，并且支持命令行参数作为输入。

- **歧义越大的场景，反例越重要**
  参考 [`claude-api`](https://github.com/anthropics/skills/blob/main/skills/claude-api/SKILL.md) 这个 Skill 的写法。在容易产生误解的地方，提供错误示范或反例非常有价值。

- **先写“一个能跑的例子”，再抽象成 Skill**
  这是一个反直觉但极为好用的方法：先在对话中让 AI 手动完成一次任务，把过程成功跑下来，然后向 Skill 测试工具（如 skill-creator）说：*"把刚才这段流程固化成 Skill"*。从具体案例抽象成规则，比凭空手写抽象逻辑效果好得多。

- **把大块知识拆分为多文件，`SKILL.md` 只放索引**
  在主 `SKILL.md` 里只写：*"详细 API 参考见 `reference/api.md`，模板见 `templates/`"*。AI 在需要时会自己顺藤摸瓜去阅读。主 `SKILL.md` 最好控制在 100 行以内，因为它是每次环境加载都要通读一遍的入口文件。

- **确定性逻辑：脚本优先于 Prompt**
  如果某个子步骤是完全确定性的操作（如复杂的格式转换、严格校验、调 API 等），把它写成 `scripts/xxx.py` 让 Skill 去调取执行，往往比用自然语言让模型生成要可靠一个数量级。因为模型默认偏好使用 Prompt 解决问题，你需要主动提醒它：*"这一步请用 Python 脚本实现"*。

- **利用 `allowed-tools` Frontmatter 收窄权限**
  在 `SKILL.md` 开头的 Frontmatter 里，可以通过声明 `allowed-tools: Read, Grep, Bash(git log:*)` 来严格限定工具的使用范围，避免模型在不该动手的场景下乱改文件。自动生成的 Skill 通常不会主动包含权限限制，需要人工设计或提示。

- **测试驱动：先写 Eval，再写 Skill**
  让 Skill-Creator 先生成 5 到 10 个测试 Prompt（包含预期行为描述），再让它撰写具体的 `SKILL.md` 逻辑，最后运行 Eval 以对齐最终效果。这其实就是前面提到的 RL（强化学习）循环的自动化实现版本。