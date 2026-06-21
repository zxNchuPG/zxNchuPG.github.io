---
layout:      post
title:       "Superpowers"
subtitle:    "技能驱动的 AI 工程方法、与 OpenSpec 的边界、典型疑问，以及混合使用完整案例"
author:      "Ekko"
header-img:  "img/bg/bg-scattered.png"
catalog:     true
tags:
  - 学习笔记
  - AI
  - 工具
  - Superpowers
  - OpenSpec
---

> 这篇笔记的目标是把 `Superpowers` 放回一个更准确的位置里看清楚：它不是另一个“写 spec 的工具”，而是一套约束 AI 编码助手工作方式的技能体系。重点不在“产出哪几个文档”，而在“让代理按什么工程纪律工作”。

> 文章会把 `Superpowers` 和 `OpenSpec` 放在同一张图里比较，并给出一套更实用的搭配思路：由 `OpenSpec` 负责定义变更意图和长期规范，由 `Superpowers` 负责规划执行、测试、调试、评审与收尾。重点会单独解释一个容易卡住的问题：如果混合流程里没有执行 `/opsx:apply`，为什么仍然会走到 `/opsx:archive`，以及这在语义上到底归档了什么。

> 参考资料：
>
> [obra/superpowers](https://github.com/obra/superpowers)
>
> [superpowers 官方说明页](https://langlabs.io/obra/superpowers)
>
> [OpenSpec Workflow](https://openspec.pro/workflow/)
>
> [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)
>
> [OpenSpec Issue #780: Distribute OpenSpec as a Superpowers skill pack](https://github.com/Fission-AI/OpenSpec/issues/780)
>
> [站内前文：OpenSpec]({% post_url 2026-05-12-openspec %})

[TOC]

---

## 一、先说结论

`Superpowers` 和 `OpenSpec` 看起来都在解决“AI 写代码容易跑偏”这个问题，但切入层次并不一样。

- `Superpowers` 更像 **AI 编码代理的工程方法论**
- `OpenSpec` 更像 **规范驱动开发的变更治理框架**

如果只用一句话概括：

> `OpenSpec` 约束“要做什么、为什么做、系统行为怎么变化”，`Superpowers` 约束“代理在实现这件事时必须怎么做”。

所以二者并不是天然替代关系，更接近两层不同的约束：

- 上层是 **变更治理和规范沉淀**
- 下层是 **执行纪律和工程手法**

真正容易冲突的地方，不是它们不能一起用，而是如果不划分边界，就会在“讨论需求”“写计划”“保存设计”这几个环节重复产出文档。

### 1.1 这篇放在系列里的位置

如果把这组主题当成一条连续的学习链路，这篇更适合放在 [OpenSpec]({% post_url 2026-05-12-openspec %}) 之后看。

- 前一篇更偏 `OpenSpec` 自身机制，包括 `proposal/spec/design/tasks`、Delta Spec、`apply/archive`
- 这一篇更偏关系辨析，重点回答 `Superpowers` 到底补在哪一层，以及它和 `OpenSpec` 怎么分工
- 如果后续还继续写，同系列最自然的下一篇会是“混合模式下的 verify 机制、质量门设计与落地检查项”

可以概括为：

- `OpenSpec` 那篇负责建立“规范层”的基本理解
- 这一篇负责建立“规范层 + 执行层”之间的关系视图

### 1.2 推荐阅读路径

这篇笔记的信息量比单纯工具介绍更大，阅读时可以按自己的问题进入：

- 如果核心问题是“`Superpowers` 到底是什么”，优先看第二、三节
- 如果核心问题是“它和 `OpenSpec` 到底差在哪”，优先看第五、六节
- 如果核心问题是“混合模式怎么落地”，优先看第九节
- 如果核心问题是“为什么没跑 `/opsx:apply` 还能归档”，优先看 `9.4`
- 如果核心问题是“质量门到底卡在哪”，优先看 `9.7`

---

## 二、Superpowers 是什么

按照官方描述，`Superpowers` 是一套可组合的 skills library，用来让 coding agent 按照更像真实工程团队的方法工作，而不是接到需求就直接开始改代码。

它的核心形式不是一个 CLI 命令，也不是一个独立运行时，而是一组 `SKILL.md` 文件。代理在合适的上下文中读取这些技能文件，把其中的流程约束当成执行规则。

### 2.1 它解决的不是“不会写代码”，而是“不会按工程方法写代码”

在没有额外约束时，很多 AI 编码助手的默认行为通常是：

1. 接到一句需求
2. 很快开始写代码
3. 中途补一点解释
4. 最后说“已经完成”

问题在于，这个过程经常缺失几个工程上真正重要的动作：

- 没有先澄清需求和边界
- 没有先形成设计或计划
- 没有严格测试驱动
- 没有系统化调试
- 没有显式评审和收尾

`Superpowers` 的重点，就是把这些动作从“最好做一下”变成“代理应该遵守的默认流程”。

### 2.2 它的核心不是一个技能，而是一组工作流技能

根据官方说明，`Superpowers` 里比较核心的技能包括：

- `brainstorming`：在编码前先澄清问题、收敛设计
- `using-git-worktrees`：为每个开发任务建立隔离工作区
- `writing-plans`：把设计拆成清晰、可执行、可验证的任务
- `subagent-driven-development`：按任务分发执行，并做多轮检查
- `test-driven-development`：强调 `RED -> GREEN -> REFACTOR`
- `requesting-code-review` / `receiving-code-review`：把评审纳入流程
- `systematic-debugging`：按更系统的步骤定位根因
- `verification-before-completion`：完成前要求验证证据
- `finishing-a-development-branch`：任务完成后的测试、合并、清理

可以看到，`Superpowers` 的关注点主要是：

- **怎么开始干活**
- **怎么拆任务**
- **怎么改代码**
- **怎么测**
- **怎么调**
- **怎么审**
- **怎么收尾**

这是一条完整的工程执行链，但它本身并不天然提供像 `OpenSpec` 那样清晰的“主规范 / 变更规范 / 归档演进”模型。

---

## 三、怎么理解 Superpowers 的工作方式

### 3.1 它本质上是“通过提示词结构去约束代理行为”

`Superpowers` 最值得注意的一点是：它不是靠复杂运行时去强制执行流程，而是通过 skill 的文本规则，把工程方法嵌入代理的上下文。

换句话说，它更像是在给代理注入一套“你应该如何工作”的操作系统。

这套操作系统的特点是：

- 轻量
- 可读
- 可组合
- 可随平台迁移

但它也意味着，`Superpowers` 更擅长的是 **行为约束**，不是 **系统规范建模**。

### 3.2 它特别适合补齐 AI 编码中的“执行纪律”

如果一个团队已经大致知道需求是什么，但总是遇到下面这些问题：

- AI 上来就写，结果方向错了
- 任务拆得太大，改动不可控
- 没写测试就宣布完成
- 遇到 bug 只会反复试错
- 代码改完没有清晰的验证证据

那么 `Superpowers` 往往会非常有价值。

因为它补的不是“需求文档模板”，而是“工程动作顺序”。

---

## 四、OpenSpec 是什么

如果说 `Superpowers` 关注的是“代理怎么干活”，那 `OpenSpec` 关注的是“这次变更到底是什么”。

站在 `OpenSpec` 的视角里，核心问题不是：

> 代理是否做了 TDD？

而是：

> 这次变更的目标、范围、行为变化、技术方案和任务拆解，是否形成了可审查、可追踪、可归档的结构化产物？

它的典型产物是：

- `proposal.md`
- `specs/`
- `design.md`
- `tasks.md`

它的典型流程是：

```text
propose -> apply -> archive
```

或者扩展成：

```text
new -> ff/continue -> apply -> verify -> archive
```

其中最关键的点，在于 `OpenSpec` 把“当前系统规范”和“本次变更规范”分开了：

- `openspec/specs/` 表示当前系统的 source of truth
- `openspec/changes/<change-id>/specs/` 表示这次变更的 Delta Spec

这一层能力非常重要，因为它处理的是：

- 需求不会只存在于聊天记录里
- 变更范围有独立边界
- 规范能沉淀为长期知识
- 多次迭代之后仍然能回看系统为什么变成现在这样

这正是 `Superpowers` 不擅长、也不是它主要目标的部分。

---

## 五、Superpowers 和 OpenSpec 的差异到底在哪里

很多人第一次接触两者时，会觉得它们在“brainstorming”“planning”“执行前审查”上有重叠。这个观察没有错，但真正的区别不在“都有计划环节”，而在“计划服务于什么目标”。

### 5.1 一张表看差异

| 维度 | Superpowers | OpenSpec |
| --- | --- | --- |
| 核心定位 | 代理执行方法论 | 规范驱动开发框架 |
| 主要约束对象 | AI 代理的行为 | 变更本身的规范与产物 |
| 关注重点 | 澄清、规划、TDD、调试、评审、收尾 | proposal、spec、design、tasks、archive |
| 主要价值 | 防止代理跳步骤、少验证、少评审 | 防止需求漂移、变更失忆、规范丢失 |
| 产物形态 | skills 驱动，可生成计划/设计文档 | 明确的规范目录与变更目录 |
| 长期知识沉淀 | 有一定文档沉淀，但不是主打 archive 模型 | 很强，尤其是 Delta Spec + Archive |
| 适合问题 | “AI 工作方式不专业” | “变更治理和系统规范不清晰” |
| 更像什么 | 工程纪律层 | 规范治理层 |

### 5.2 两者分别在补哪种短板

如果把 AI 编码里的常见问题拆开看，大致有三类：

1. **需求没讲清楚**
2. **执行过程不专业**
3. **历史决策无法沉淀**

那么可以粗略对应为：

- `OpenSpec` 主要解决第 1 和第 3 类
- `Superpowers` 主要解决第 2 类

所以二者最合理的关系，不是竞争，而是分工。

---

## 六、两者哪里会重叠

虽然分工不同，但它们确实有交叉区域，主要集中在下面三处。

### 6.1 需求讨论

- `Superpowers` 的 `brainstorming` 会先把问题说清楚
- `OpenSpec` 的 `propose` 也会生成 proposal 并明确范围

两者都在做“先想清楚再写代码”，但：

- `Superpowers` 偏 **方法动作**
- `OpenSpec` 偏 **结构化产物**

### 6.2 计划生成

- `Superpowers` 有 `writing-plans`
- `OpenSpec` 有 `design.md` 和 `tasks.md`

表面看都在拆任务，差异在于：

- `Superpowers` 更强调“给代理一个足够清晰的执行计划”
- `OpenSpec` 更强调“把本次变更的设计与任务纳入规范流转”

### 6.3 文档落盘

两者都会把一部分信息写到磁盘上，但写入的目标不同：

- `Superpowers` 更偏执行辅助文档
- `OpenSpec` 更偏项目内长期治理文档

如果不设边界，最容易出现的问题就是：

- 一份设计写两遍
- 一套任务清单维护两份
- 一边更新了，另一边没更新

这也是为什么搭配使用时，必须先定“谁是主文档”。

---

## 七、什么时候更适合用 Superpowers

如果只能选一个工具，`Superpowers` 更适合下面这些场景：

### 7.1 你最痛的是“AI 干活方式不稳定”

比如：

- 代理经常跳过澄清直接写
- 任务拆解很粗
- 很少主动补测试
- 改完就说好了
- 调试时没有方法论

这类问题的根子不在“缺少一个规范目录”，而在“执行纪律缺失”，这时优先补 `Superpowers` 更直接。

### 7.2 你想把 AI 当工程师，而不是生成器

`Superpowers` 的价值很大程度上来自一个思路转变：

> 不要只让 AI 产出代码，要让 AI 遵守工程流程。

如果团队对这件事有共识，它带来的收益通常会比单纯增加更多 prompt 更明显。

### 7.3 你更关心实现质量，而不是规范归档

个人项目、快速实验、小范围特性开发，很多时候不一定需要很完整的变更治理，但很需要：

- 别乱改
- 别漏测
- 别草率结束

这时 `Superpowers` 往往足够有效。

---

## 八、什么时候更适合用 OpenSpec

相对地，`OpenSpec` 更适合下面几类场景：

### 8.1 变更本身需要被治理

比如：

- 功能会跨多个模块
- 涉及多人协作
- 需要明确 in scope / out of scope
- 需求会多轮迭代

这时重点已经不是“代理会不会写测试”，而是“这次变更到底算什么、边界在哪、以后怎么追踪”。

### 8.2 项目是存量系统

`OpenSpec` 对 brownfield 项目的价值很突出，因为它允许用 Delta Spec 描述变化，而不是每次都重写整份规范。

对于已有系统，这种“增量描述变化”的方式比重新写一套完整说明更现实。

### 8.3 你希望把系统知识沉淀下来

很多团队用 AI 一段时间后，真正痛苦的并不是某次实现写差了，而是几个月后没人知道：

- 当初为什么这么设计
- 这个行为是后来加的还是一开始就有
- 哪个改动改变了系统契约

这类问题更像知识治理问题，`OpenSpec` 的 archive 机制就很有价值。

---

## 九、Superpowers 和 OpenSpec 怎么搭配

我更推荐把二者理解成：

- `OpenSpec` 负责 **定义和沉淀**
- `Superpowers` 负责 **执行和验证**

### 9.1 最稳妥的分工方式

实践里最不容易打架的办法是：

1. 用 `OpenSpec` 产出本次变更的正式文档
2. 用 `Superpowers` 读取这些文档并执行工程流程
3. 变更完成后仍然回到 `OpenSpec` 做验证与归档

也就是说，把角色定成：

- `proposal/spec/design/tasks` 的主来源是 `OpenSpec`
- `worktree/TDD/debug/review/verification` 的主来源是 `Superpowers`

这样各自负责自己擅长的层。

### 9.2 一套比较顺的协作流程

可以把整个过程拆成下面几步：

#### Step 1：用 OpenSpec 固定变更边界

先通过 `OpenSpec` 建立这次变更：

```bash
/opsx:propose add-user-login
```

或者：

```bash
/opsx:new add-user-login
/opsx:ff
```

这一阶段的目标不是马上编码，而是先把下面几件事钉住：

- 为什么做
- 做什么
- 不做什么
- 系统行为会怎么变化
- 技术方案是什么
- 任务怎样拆

#### Step 2：把 OpenSpec 产物当成 Superpowers 的输入

当 `proposal.md`、`design.md`、`tasks.md` 已经足够稳定后，再让 `Superpowers` 接管执行纪律：

- 用它的 planning / execution 思路检查任务是否可执行
- 用 `using-git-worktrees` 做隔离分支
- 用 `test-driven-development` 驱动编码
- 用 `requesting-code-review`、`verification-before-completion` 控制质量

这里的关键原则是：

> `Superpowers` 不再重新定义“做什么”，而是围绕 `OpenSpec` 已经确定的内容去执行。

#### Step 3：实现完成后回到 OpenSpec

代码、测试和验证都完成后，再执行：

```bash
/opsx:verify
/opsx:archive
```

把这次变更合并回主规范，形成长期知识沉淀。

### 9.3 一句话版本

最简化的搭配口诀可以写成：

```text
OpenSpec 定义变化
Superpowers 约束执行
OpenSpec 归档结果
```

### 9.4 一个最容易卡住的问题：没有执行 `/opsx:apply`，为什么还能 `/opsx:archive`

这个问题不澄清，混合流程就会一直显得别扭。

很多人看到 `OpenSpec` 官方默认流程：

```text
propose -> apply -> archive
```

会自然地得出一个推论：

> 只有执行了 `/opsx:apply`，才有资格执行 `/opsx:archive`。

但更准确的理解应该是：

> `/opsx:apply` 是 OpenSpec 默认工作流里的“实现动作”，`/opsx:archive` 是“把已经完成并确认过的变更，合并回 source of truth”的动作。

两者经常连在一起出现，是因为在纯 `OpenSpec` 流程里，通常就是由 `apply` 负责实现代码；但 `archive` 的真正先决条件，并不是“你是否敲过这个命令”，而是下面几件事是否成立：

- 代码实现已经完成
- `tasks.md` 对应任务已经落实
- 变更后的实现仍然符合 `proposal.md`、`specs/`、`design.md`
- 测试和验证已经通过
- 当前变更已经具备合并回主规范的条件

官方工作流说明里，`archive` 的描述更接近“在 implementation complete and tested 之后归档”，而不是“只有执行 apply 命令的会话才能归档”。这两个语义并不相同。

换句话说，`archive` 归档的不是“命令调用历史”，而是：

- 这次 change 的 Delta Spec
- 这次 change 对应的正式产物
- 这次 change 最终落地后的系统行为

所以在混合模式下有两种完全不同但都合理的路径：

#### 路径 A：纯 OpenSpec 实现

```text
/opsx:propose -> /opsx:apply -> /opsx:verify -> /opsx:archive
```

这里 `apply` 是实现层，`archive` 是规范归档层。

#### 路径 B：OpenSpec 负责规范，Superpowers 负责实现

```text
/opsx:new -> /opsx:ff -> Superpowers 执行实现 -> /opsx:verify -> /opsx:archive
```

这里没有执行 `/opsx:apply`，但并不等于“没有 implementation”。实现只是由 `Superpowers` 或人工编码完成，而不是由 `OpenSpec` 的 apply 步骤完成。

这一点可以概括成一句话：

> 在混合流程里，`apply` 可以被替换，`archive` 不应该被跳过。

### 9.5 混合使用时，`/opsx:apply` 到底该怎么处理

真正实操时，`/opsx:apply` 大致有三种处理方式。

#### 方式一：保留 `apply`，让 Superpowers 只管辅助质量

这种方式最接近标准 `OpenSpec`：

- `OpenSpec` 负责 proposal、spec、design、tasks
- `OpenSpec` 也负责 `apply`
- `Superpowers` 主要用于 worktree、调试、补测试、代码评审、完成前验证

适合下面两类场景：

- 团队已经习惯了 `OpenSpec` 的默认工作流
- 你只想借用 `Superpowers` 的局部能力，而不想改实现主链路

#### 方式二：跳过 `apply`，让 Superpowers 成为实现主链路

这是更典型的“混合模式”：

- `OpenSpec` 负责把 change 文档固定下来
- `Superpowers` 读取这些文档并真正改代码
- 完成后回到 `OpenSpec` 做 `verify` 和 `archive`

这种方式适合下面几类情况：

- 你更信任 `Superpowers` 的 TDD、code review、debugging、branch finishing
- 你希望实现阶段使用 worktree、子代理、强约束测试流程
- 你把 `OpenSpec` 明确定位成“规范层”而不是“执行层”

#### 方式三：`apply` 只处理简单实现，复杂问题切给 Superpowers

还有一种折中方案：

- 正常情况下用 `/opsx:apply`
- 如果实现中遇到复杂调试、重构、测试缺失、执行跑偏，再切换到 `Superpowers`

这种方式的优点是迁移成本低，但缺点也很明显：

- 边界容易漂
- 责任归属不够清楚
- 过程文档容易不一致

所以从长期维护角度看，更推荐前两种，而不是把实现权在两套流程之间来回切换。

### 9.6 一个完整混合案例

上面的抽象讨论还不够，真正好理解的方式是看一遍完整流程。

这里假设有一个已经在线上的后台系统，现在要新增一个能力：

> 在用户登录成功后，如果命中高风险规则，就在管理后台的用户详情页显示“近期登录存在异常风险”的提醒卡片；本次只做风险提示，不做登录拦截，不改鉴权主流程。

这个需求适合拿来演示混合流，有三个原因：

- 它是 brownfield 场景，不是从零开始
- 它既需要明确边界，又需要谨慎实现
- 它很适合体现“规范”和“执行纪律”是两层不同的事

#### Step 0：先明确问题点

这个需求表面上像是“加一个提示 UI”，但其实有几个容易出错的点：

1. 需求边界容易漂
2. AI 容易顺手把“风险提示”做成“风险拦截”
3. 变更涉及鉴权、风控和后台展示，跨域信息容易散
4. 如果没有测试约束，最容易在登录流程里引入回归

所以这里最适合的分工就是：

- 用 `OpenSpec` 锁定边界
- 用 `Superpowers` 约束实现

#### Step 1：先用 OpenSpec 建 change

```bash
/opsx:new add-login-risk-banner
/opsx:ff
```

这一阶段的目标不是写代码，而是把 change folder 建完整。

理想情况下，这一轮应该产出：

- `proposal.md`：为什么做、做什么、不做什么
- `specs/auth/spec.md`：增加“风险提示”这一行为，而不是“拒绝登录”
- `design.md`：风险信号来源、聚合点、展示位置、数据流
- `tasks.md`：拆成可实现的小任务

这一步最关键的不是“文档齐不齐”，而是把范围钉死，例如：

- `in scope`：展示风险提示卡片、补充风险标签文案、增加后台查询接口字段
- `out of scope`：登录拦截、短信二次验证、风控规则引擎重写

如果这里不写清楚，后面的 AI 很容易“好心做多”。

#### Step 2：人工 review 一次 artifacts

混合流程里，这一步不能省。

要重点检查三个问题：

1. `specs/` 写的是“行为变化”，不是实现细节
2. `design.md` 是否明确指出数据从哪里来、在哪一层汇总、在哪一层展示
3. `tasks.md` 是否已经拆到可以让代理稳定执行，而不是一条任务写成“完成整个风险提示功能”

如果 review 后发现设计有缺口，就继续改 `proposal/spec/design/tasks`，而不是急着进入实现。

#### Step 3：这里故意不执行 `/opsx:apply`

这是这个案例最关键的一步。

这里假设团队决定：

> 不用 `OpenSpec` 的 apply 负责实现，而是把执行阶段交给 `Superpowers`。

原因是这个需求虽然不算特别大，但它有几个实现层风险：

- 登录相关代码比较敏感
- 需要测试先行
- 可能涉及调试已有风控信号链路
- 希望放到 worktree 里隔离完成

所以这里更合理的执行方式是：

1. 让 `Superpowers` 读取 `openspec/changes/add-login-risk-banner/` 下的文档
2. 以这些文档为 source of intent
3. 用 `using-git-worktrees`、`test-driven-development`、`requesting-code-review` 这一套执行

注意这里没有 `/opsx:apply`，但不等于“跳过实现”。实现动作只是换了执行引擎。

#### Step 4：由 Superpowers 负责真正实现

这一阶段可以理解成：

```text
OpenSpec 提供任务真相
Superpowers 负责工程执行
```

一个更像真实工程的执行链路通常会长这样：

1. 建 worktree 和独立分支
2. 先写失败测试，验证“高风险登录会显示提醒卡片”
3. 补充风险标签的数据读取和组装逻辑
4. 实现后台用户详情页的提醒组件
5. 继续补边界测试，例如“普通登录不展示风险卡片”
6. 自查是否出现越界实现，例如误改登录拦截逻辑
7. 做一次 code review 和完成前验证

如果中间发现原始设计不够准确，例如风险信号不是登录服务直接提供，而是审计表聚合得出，就应该回头更新 `design.md`，而不是让实现偷偷偏离文档。

#### Step 5：把 OpenSpec 产物和真实实现重新对齐

混合流程最容易漏掉的，不是写代码，而是实现完后忘记回填 artifacts。

这一阶段至少要做四件事：

1. 勾掉 `tasks.md` 中已完成的任务
2. 如果实际方案和 `design.md` 有偏差，更新设计文档
3. 如果行为边界发生变化，更新 Delta Spec
4. 确认 `proposal.md` 的 in scope / out of scope 仍然成立

这一轮对齐做完后，OpenSpec change folder 才重新成为“这次变更的真实描述”。

#### Step 6：执行 `/opsx:verify`

```bash
/opsx:verify add-login-risk-banner
```

这里的意义是把“已经实现了”变成“已经和规范对齐了”。

如果验证阶段发现下面这些问题，就不应该归档：

- `tasks.md` 完成了，但 spec 场景没覆盖
- 代码能跑，但行为超出了 proposal 范围
- 设计上临时改过，但 `design.md` 没同步
- 测试只覆盖 happy path，没有覆盖关键边界

这一步可以理解成混合流程里的“最后一致性检查”。

#### Step 7：最后再执行 `/opsx:archive`

```bash
/opsx:archive add-login-risk-banner
```

这时归档才是语义完整的，因为：

- 代码已经由 `Superpowers` 实现完成
- artifacts 已经和实现重新对齐
- `verify` 已经证明当前实现与变更文档基本一致
- 这次变更已经具备合并回 source of truth 的条件

归档完成后，发生的是两件事：

1. Delta Spec 合并回主 `openspec/specs/`
2. change folder 移入 `openspec/changes/archive/`

所以这里归档的核心不是“OpenSpec 自己写了代码”，而是：

> 这次 change 已经从提案态，变成了系统正式知识的一部分。

#### Step 8：这个案例真正回答了什么问题

这个完整案例其实回答了四个常见疑问：

1. 混合使用时，`OpenSpec` 和 `Superpowers` 是否会重复实现？
2. 如果不跑 `/opsx:apply`，是否还能 `/opsx:archive`？
3. `archive` 归档的到底是代码执行过程，还是变更知识？
4. 在混合模式下，谁来对最终一致性负责？

答案可以压缩成下面四句话：

- `OpenSpec` 负责 change 的定义、校验和归档
- `Superpowers` 负责实现阶段的工程纪律
- `/opsx:apply` 不是 archive 的唯一前提，实现完成才是
- `/opsx:archive` 归档的是被验证过的 change，而不是某次命令调用记录

### 9.7 混合模式下，`verify` 具体检查什么，为什么它比 `archive` 更像真正的质量门

如果把混合流程拆开看，`verify` 和 `archive` 经常会被误以为只是前后相邻的两个命令，但它们的职责并不对称。

更准确地说：

- `verify` 负责回答“这次 change 现在到底对不对”
- `archive` 负责回答“这次 change 现在能不能沉淀进主规范”

所以从工程语义上看，`verify` 更像质量门，`archive` 更像知识入库动作。

#### `verify` 在混合模式下到底检查什么

在纯 `OpenSpec` 场景里，`verify` 会检查实现是否匹配 change artifacts；在混合模式里，这个角色反而更重要，因为实现动作已经交给了 `Superpowers` 或人工编码，流程天然更容易出现“代码改完了，但文档没对齐”的情况。

可以把 `verify` 的检查点拆成四层。

#### 第一层：任务完成度检查

这一层回答的问题是：

> `tasks.md` 里写的事情，是否真的完成了？

常见检查点包括：

- 任务项是否已经落实到代码，而不只是口头完成
- 是否有任务被跳过
- 是否有任务虽然完成，但实现方式与设计前提冲突

如果这一层没过，就说明实现连最基础的交付完整性都没满足。

#### 第二层：规格一致性检查

这一层回答的问题是：

> 代码行为是否真的符合 `specs/` 里的 requirement 和 scenario？

这是混合模式里最容易出问题的一层，因为 `Superpowers` 更强调工程纪律，不会天然替你维护 Delta Spec 的正确性。

典型问题包括：

- 代码实现了一个“看起来合理”的功能，但超出了 spec
- 原本只要求“提示风险”，实现却顺手做成了“阻断登录”
- happy path 能跑，但 scenario 里的边界条件没覆盖

这一层没过时，问题不在代码能不能跑，而在它是否仍然是“这次 change 想要的那个行为”。

#### 第三层：设计一致性检查

这一层回答的问题是：

> 真实实现是否仍然符合 `design.md` 的核心决策？

很多混合流程里的偏差，不是功能错误，而是设计漂移。例如：

- 文档里写的是“从审计事件聚合风险信号”，实现时却直接在登录链路里硬编码判断
- 文档里要求“后台查询接口补字段”，实现时却临时拼装多个接口
- 原本设计为低侵入式扩展，实现时却偷偷修改了核心鉴权流程

这些问题短期可能也能工作，但它们会导致：

- 后续维护者读 `design.md` 时理解错系统
- 下一次 change 建在错误认知之上
- `archive` 后沉淀进主规范的知识不再可信

#### 第四层：验证证据检查

这一层回答的问题是：

> 你有什么证据证明前面三层不是“自我感觉良好”？

在混合模式下，这一层尤其重要，因为 `Superpowers` 往往会引入更多工程动作，例如 TDD、自检、code review、debugging 记录。理想状态下，`verify` 应该要求能拿得出下面这些东西中的一部分：

- 失败测试到通过测试的链路
- 关键场景测试结果
- code review 后的修正记录
- 对高风险路径的人工验证结论
- artifacts 更新后的最终一致性

没有证据的“已完成”，本质上只是声明，不是验证。

#### 为什么说 `verify` 比 `archive` 更像真正的质量门

质量门的本质，不是“把状态往下推进”，而是“判断当前状态是否允许推进”。

按这个定义看：

- `verify` 会拦住不完整、不一致、无证据的实现
- `archive` 一旦开始，前提通常是“这个 change 已经被认为可以收口”

也就是说：

- `verify` 是 **判定动作**
- `archive` 是 **沉淀动作**

前者更像门，后者更像入库。

#### 一个更容易记住的区分方式

可以用下面这组问句来区分：

- `verify` 问的是：这次 change 真的完成并且对齐了吗？
- `archive` 问的是：既然已经对齐，是否可以把它并入主规范了？

如果 `verify` 没做扎实，`archive` 做得越顺，反而越危险，因为它会把未经充分确认的理解写进 source of truth。

#### 混合模式下，我更推荐的顺序

如果采用 `OpenSpec + Superpowers`，更稳妥的末端流程通常是：

```text
Superpowers 完成实现
-> 回填 OpenSpec artifacts
-> /opsx:verify
-> 修正剩余偏差
-> /opsx:archive
```

这一顺序背后的核心逻辑是：

- `Superpowers` 保证“实现过程尽量专业”
- `verify` 保证“最终结果确实和变更文档一致”
- `archive` 保证“确认过的 change 进入长期知识”

所以如果只看“最后一道门”在哪里，答案通常不是 `archive`，而是 `verify`。

---

## 十、搭配使用时要避免什么

如果想让两套体系真正协同，最重要的不是“都装上”，而是避免重复治理。

### 10.1 不要让两边同时当主计划

最常见的问题是：

- `OpenSpec` 里有 `tasks.md`
- `Superpowers` 里又产出一份 plan
- 最后不知道该信哪一份

更稳妥的做法是：

- `OpenSpec tasks.md` 作为 **任务主清单**
- `Superpowers` 只负责把它转成 **执行过程中的纪律动作**

如果确实需要 `Superpowers` 细化计划，也应该把它视为执行层展开，而不是替代主任务单。

### 10.2 不要让两边同时当主设计

如果 `OpenSpec design.md` 已经明确了架构决策，就不要再让 `Superpowers` 重新发明一份平行设计文档。

否则最容易出现：

- 架构假设不一致
- 命名不一致
- 一个更新了另一个没更新

### 10.3 不要在非常小的改动上同时走两套重流程

比如改一个 typo、修一个简单样式、调一个常量，这时同时跑 `OpenSpec + Superpowers` 很可能只会增加成本。

一个简单经验是：

- **小改动**：优先轻量流程
- **跨模块或高风险改动**：再考虑双栈

---

## 十一、我更推荐的两种用法

### 11.1 用法 A：OpenSpec 为主，Superpowers 为辅

这是我最推荐的模式，尤其适合：

- 存量项目
- 团队协作
- 多轮迭代
- 需要留痕和归档

其思路是：

- 用 `OpenSpec` 管理所有正式变更
- 用 `Superpowers` 保证实现质量

这时的关系最清晰，也最不容易出现产物打架。

### 11.2 用法 B：Superpowers 先探索，OpenSpec 再固化

如果一开始需求还非常模糊，也可以先借助 `Superpowers` 的 brainstorming 把问题想清楚，再把最终结论收敛到 `OpenSpec` 的 proposal/spec/design/tasks 中。

但这里有一个前提：

> 一旦准备正式实现，就要回到 `OpenSpec` 建立唯一的正式变更产物。

否则后面还是会遇到“到底哪个文档代表最终版本”的问题。

---

## 十二、一个实际的脑图式理解

可以把两者放在同一条链路上看：

```text
需求想法
  -> OpenSpec proposal/spec/design/tasks
  -> Superpowers worktree/planning/TDD/debug/review/verify
  -> OpenSpec archive
  -> 更新系统 source of truth
```

从这个角度看：

- `OpenSpec` 负责让“意图”可追踪
- `Superpowers` 负责让“执行”可约束
- `OpenSpec` 再负责让“结果”可沉淀

这比单独依赖任意一边都更完整。

---

## 十三、总结

`Superpowers` 的价值，不在于它是不是又发明了一套 spec 文件，而在于它试图把 AI 编码从“即时问答式生成”拉回“工程化执行”。

它最强的地方是：

- 把澄清、规划、测试、调试、评审、收尾这些动作前置
- 把“代理应该如何工作”变成一套可复用技能
- 补上很多 AI 编码助手默认缺失的工程纪律

而 `OpenSpec` 最强的地方则是：

- 让变更拥有结构化产物
- 让行为变化能以 Delta Spec 形式表达
- 让规范成为长期 source of truth，而不是聊天记录副产物

所以如果一定要问两者关系，更准确的回答不是“谁替代谁”，而是：

> `Superpowers` 更像执行层的方法论，`OpenSpec` 更像治理层的规范框架。

对个人或轻量实验，单用 `Superpowers` 往往已经很有帮助；对存量系统、多人协作和需要持续沉淀的项目，`OpenSpec + Superpowers` 的组合会更完整。

最后把结论再压缩成三句话：

- 只想让 AI 干活更像工程师，用 `Superpowers`
- 只想让变更治理更清晰、规范可归档，用 `OpenSpec`
- 想同时解决“做什么”和“怎么做”，就让 `OpenSpec` 管规范，让 `Superpowers` 管执行
