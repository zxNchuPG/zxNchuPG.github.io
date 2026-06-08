---
layout:      post
title:       "Karpathy LLM Wiki 学习笔记"
subtitle:    "从 RAG 对比、三层架构、目录设计、原始资料获取到一次完整 ingest 示例"
author:      "Ekko"
header-img:  "img/bg/bg-scattered.png"
catalog:     true
tags:
  - 学习笔记
  - AI
  - LLM
  - 知识库
  - Obsidian
---

> 这篇笔记的目标，不是把 `LLM Wiki` 当成一个新名词去转述，而是把它拆成可以落地的工作流：它和传统 `RAG` 到底差在哪，为什么 Karpathy 会强调 “persistent wiki”，以及个人或小团队该如何把这套模式真正搭起来。

> 这篇内容以 Karpathy 的原始 gist 为主线，也补充了 Codex / Claude Code 的规则文件文档、Obsidian Web Clipper、Cloud Document Converter 等更偏实操的资料；因此它既解释方法论，也尽量回答“原始资料到底怎么进来、目录到底怎么搭、一次 ingest 到底怎么跑”。

> 参考资料：
>
> [Karpathy - llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
>
> [Karpathy 的 LLM Wiki 深度拆解（知乎）](https://zhuanlan.zhihu.com/p/2024921762337398826)
>
> [Karpathy 的 LLM Wiki 到底是什么（掘金）](https://juejin.cn/post/7625301482213130291)
>
> [OpenAI Codex - Custom instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
>
> [Claude Code - How Claude remembers your project](https://code.claude.com/docs/en/memory)
>
> [obsidian获取文档插件](https://help.obsidian.md/web-clipper)
>
> [Obsidian Help - Clip web pages](https://help.obsidian.md/web-clipper/capture)
>
> [网页获取文档插件](https://github.com/whale4113/cloud-document-converter)

[TOC]

---

## 一、先把结论说清楚：LLM Wiki 不是“另一个 RAG”

如果先用一句话概括 Karpathy 的核心想法，我会写成：

> **不要每次提问时都从原始资料里临时“现检索、现拼答案”，而是让 LLM 持续维护一份可演化的 Markdown Wiki，作为你和原始资料之间的中间层。**

这就是它和常见 RAG 的根本区别。

很多人接触文档问答时，默认模式大致是：

- 上传一批文件
- 查询时检索若干 chunk
- LLM 临时综合并生成回答

这当然能工作，但问题也很明显：

- 知识不会自然沉淀
- 同一类复杂问题会被反复“从头拼一次”
- 跨文档的综合结论容易只存在聊天记录里
- 之前发现的矛盾、补充和上下文不一定能长期保留

Karpathy 想做的是另一件事：

- 原始资料依然保留
- 但 LLM 会把它们逐步“编译”成结构化 wiki
- 新资料进入后，不只是新增摘要，还会回头更新旧页面
- 查询不再主要依赖原始文档，而是优先依赖这份已经被整理过的 wiki

所以更准确的理解不是：

- `LLM Wiki = RAG 替代品`

而是：

- `LLM Wiki = 持久化知识编译层`
- `RAG = 查询时检索层`

两者可以并存，但关注点不同。

## 二、它真正想解决什么问题

Karpathy 在 gist 里反复强调的，其实不是某个工具，而是一个长期知识管理的痛点：

> 人类并不是真的不会写笔记，真正难的是“持续维护一份会不断长大的知识库”。

普通笔记系统最大的问题不是“写不出来”，而是后续维护成本很高：

- 新资料来了，旧页面要不要更新
- 某个结论被推翻了，哪里需要回改
- 一个概念已经在 5 个地方出现过，是否应该抽成独立页面
- 某篇页面长期没有入链，是不是已经成了孤岛
- 某次高质量问答其实很有价值，要不要沉淀回知识库

这些事都像“记账”和“整理台账”，信息量不一定大，但非常耗精力。

Karpathy 的判断是：

- 人类更适合做资料筛选、方向判断、问题提出
- LLM 更适合做摘要、归档、更新交叉链接、统一格式、补索引

所以这套模式的本质分工是：

- 人负责 `curate`
- LLM 负责 `maintain`

这个分工一旦成立，wiki 才有可能持续增长，而不是写几篇后就荒废。

## 三、核心架构：Raw Sources -> Wiki -> Schema

Karpathy 在原文里把系统拆成 3 层，这一层次关系非常关键。

### 3.1 Raw Sources：原始资料层

这是你的事实来源层，特点是：

- 文章
- 论文
- PDF
- 图片
- 会议记录
- 数据文件

这一层的核心约束是：

> **只读，不改。**

因为它承担的是“可追溯的源头事实”。

如果原始资料层也被 LLM 随手改写，后面就很难判断：

- 某个结论到底来自原文，还是来自二次加工
- 某条信息是否被误改
- 某次争议到底该回到哪里核对

所以 Raw Sources 的作用不是“方便检索”，而是“作为不可变输入”。

### 3.2 Wiki：结构化知识层

这一层才是 LLM Wiki 的主角。

它不是原始资料的简单摘要堆积，而是一个可持续维护的 Markdown 知识库，常见内容包括：

- 主题总览页
- 概念页
- 实体页
- 对比页
- 来源摘要页
- 综合分析页

Karpathy 对这一层的定位很明确：

> **你主要负责阅读和提问，LLM 负责写入和维护。**

也就是说，这一层默认应该由 LLM 主导生成和更新，而不是人手工维护所有页面。

### 3.3 Schema：规则与工作流层

这一层最容易被忽略，但实际上最关键。

Schema 可以理解为一个约束文档，比如：

- 目录怎么组织
- 页面如何命名
- 每类页面应该长什么样
- ingest 时先更新旧页还是优先新建
- query 的结果什么时候需要写回 wiki
- lint 时应该检查哪些问题

如果没有这一层，LLM 做的事情往往会越来越像“聊天机器人临时输出”，而不是“稳定维护一个知识库”。

所以可以把三层关系压缩成一句话：

```text
Raw Sources 提供事实输入
Wiki 承接整理后的知识输出
Schema 约束 LLM 如何持续维护这份知识输出
```

## 四、三类核心操作：Ingest、Query、Lint

Karpathy 把日常使用动作收束成 3 类，这个划分非常实用。

### 4.1 Ingest：摄入新资料

这一步不是“把文件放进去就结束”，而是：

1. 把新资料加入 raw collection
2. 让 LLM 阅读该资料
3. 生成一页或多页摘要/主题更新
4. 更新相关概念页、实体页、索引页
5. 记录这次变更

Karpathy 提到他更喜欢一次 ingest 一个来源，并且自己参与其中。这个习惯很值得借鉴，因为它意味着：

- 每次更新范围可控
- 更容易及时纠偏
- 你能逐步校准 schema

对个人学习来说，这通常比“一次批量喂几十篇文章”更稳。

### 4.2 Query：基于 Wiki 提问

Query 阶段的重点不是直接回到原始资料里全文搜，而是优先：

1. 看 wiki 的索引
2. 找到相关页面
3. 读取这些页面
4. 再做综合回答

这意味着回答的基础已经不是“原始碎片”，而是“之前整理过的知识结构”。

更重要的是，Karpathy 还强调了一点：

> **高价值回答本身也可以写回 wiki，成为新页面。**

这一点特别像代码世界里的“产出可复用资产”：

- 一次有价值的比较
- 一次系统性的分析
- 一次跨资料的综合结论

不应该只停留在聊天记录里。

### 4.3 Lint：给 Wiki 做健康检查

随着 wiki 增长，真正危险的不是没有内容，而是内容开始失控。

Lint 阶段要检查的典型问题包括：

- 旧结论是否已经被新资料推翻
- 有没有孤立页面
- 有没有只被提到但尚未独立成页的重要概念
- 交叉引用是否不足
- 有哪些数据缺口值得继续补资料

这个动作非常像代码库里的质量治理：

- 不只是新增内容
- 还要定期做结构维护

## 五、为什么它在小到中等规模下可以先不靠向量 RAG

这是很多人第一次看到 gist 时最意外的一点。

Karpathy 的建议不是一上来就接 embedding、向量库和复杂检索层，而是先用两个特殊文件把 wiki 跑起来：

- `index.md`
- `log.md`

### 5.1 `index.md`：内容索引

`index.md` 是内容导向的目录页，通常会记录：

- 每个页面的链接
- 一句话摘要
- 分类信息
- 可能还有日期、来源数等元信息

它的价值在于：

- LLM 查询时先读索引
- 快速确定哪些页面值得展开
- 避免每次全库盲读

换句话说，`index.md` 先承担了一个“人工可读、LLM 也可读”的轻量检索入口。

### 5.2 `log.md`：时间线日志

`log.md` 是按时间记录的变更历史，比如：

```text
## [2026-04-02] ingest | 某篇文章标题
## [2026-04-03] query  | 对某个主题的比较分析
## [2026-04-04] lint   | 发现两处结论冲突
```

它的作用主要不是内容导航，而是帮助回答这些问题：

- 最近处理过什么
- 这个 wiki 是怎么长起来的
- 上一轮 ingest 或 lint 做了哪些事

这类时间维度信息对 LLM 也很有帮助，因为它能补足“上下文最近发生了什么”。

### 5.3 为什么这在早期是够用的

Karpathy 的判断很务实：

- 当资料规模还不算夸张时
- 一个好的索引文件 + 有结构的页面体系
- 往往就已经足够让 LLM 导航整个 wiki

所以更合理的工程顺序不是：

1. 先上最复杂的 RAG 基建

而是：

1. 先把页面结构、索引、日志、schema 跑顺
2. 再在规模增大后补搜索工具

这个顺序能避免“基础内容组织还没稳，就先把基础设施堆很重”。

## 六、Karpathy 给出的几个实用技巧

原始 gist 虽然很短，但里面有几条很接地气的建议。

### 6.1 Obsidian 是 IDE，LLM 是程序员，Wiki 是代码库

这是全文最出圈的类比，但它不只是一个比喻，背后其实对应了职责分配：

- Obsidian：负责浏览、链接、图谱查看
- LLM：负责写入和维护
- Wiki：负责承载持续沉淀下来的知识资产

这个类比成立后，很多动作就顺了：

- 你不必强迫自己亲手维护所有页面
- 你主要是“看”和“问”
- LLM 主要是“写”和“改”

### 6.2 Web Clipper 很重要

Karpathy 明确提到 Obsidian Web Clipper，因为很多网页资料如果不先转成 Markdown，本地知识流就不完整。

它解决的是：

- 如何把网络文章稳定地落到本地
- 如何让资料成为 raw sources 的一部分

也就是说，LLM Wiki 要能长期运转，首先得有顺手的“采集入口”。

### 6.3 图片下载到本地是加分项，但不是硬前提

Karpathy 的建议是：如果资料里有重要图片，最好下载到本地。

原因很简单：

- 外链可能失效
- 本地图像更稳定
- LLM 可以在需要时额外读取图片内容

但他也说得很清楚，这不是绝对必需项。

所以工程上更合理的理解是：

- 文字是主干
- 图片是增强上下文

只有当图片本身承载关键信息时，本地化才真正有必要。

## 七、和常见知识系统的关系：我会怎么理解它

把这套模式放到常见工具图谱里，大概可以这样看：

| 方案 | 更像在做什么 | 强项 | 弱项 |
| --- | --- | --- | --- |
| 普通笔记 | 人自己写和整理 | 灵活、低门槛 | 维护成本高，交叉更新难 |
| RAG | 查询时检索原始资料 | 即时问答强 | 知识不沉淀，回答不复用 |
| LLM Wiki | 让 LLM 持续维护中间知识层 | 可积累、可演化、可阅读 | 需要 schema 和维护纪律 |

所以更重要的不是“谁替代谁”，而是：

- 如果你只是偶尔问几个问题，RAG 就够
- 如果你要长期深挖一个主题，LLM Wiki 更有价值
- 如果你既想保留原始资料，又想有可读的知识层，二者可以组合

## 八、一个最小可用落地方式

如果想把这套思路先做成个人学习流，我更推荐下面这种简化结构：

```text
llm-wiki/
├── raw/
│   ├── articles/
│   ├── papers/
│   ├── notes/
│   └── assets/
├── wiki/
│   ├── index.md
│   ├── log.md
│   ├── overview.md
│   ├── concepts/
│   ├── entities/
│   └── sources/
└── AGENTS.md
```

其中每一层的职责可以先尽量收紧：

- `raw/`：只放原始资料，不改写
- `wiki/`：只放 LLM 生成或维护的结构化页面
- `AGENTS.md`：写清楚目录规则、页面模板、更新原则

第一阶段不需要追求很复杂，只要先把下面 4 件事做稳：

1. 有固定目录
2. 有固定页面类型
3. 有固定索引文件
4. 有固定 ingest / query / lint 规则

只要这 4 件事稳定了，后面再补搜索、脚本、自动化都不晚。

## 九、这套模式最适合什么，不适合什么

### 9.1 适合的场景

它更适合“需要持续累积、持续比较、持续回看”的主题：

- 长期研究一个技术方向
- 系统化读书
- 课程学习
- 团队内部知识库
- 竞品分析
- 个人项目沉淀

这些场景的共同点是：

- 知识不是一次性消费
- 旧结论会被新资料修正
- 需要跨资料形成长期结构

### 9.2 不太适合的场景

如果只是下面这些需求，LLM Wiki 反而可能太重：

- 临时看一两篇文章
- 只做短期问答
- 资料非常少且不会继续增长
- 只需要搜得到，不需要长期维护知识结构

这时直接：

- 收藏夹
- 普通笔记
- 简单 RAG

可能就够了。

## 十、一个真实可用的目录结构应该长什么样

如果只是想先跑起来，我更推荐直接从下面这套“标准版目录”起步，而不是只建 `raw/` 和 `wiki/` 两个空目录。

```text
llm-wiki/
├── AGENTS.md
├── CLAUDE.md
├── README.md
├── .obsidian/                  # 可选，Obsidian Vault 配置
├── inbox/                      # 临时投递区
│   └── .processed/             # 已处理过的临时资料
├── raw/                        # 原始资料层，只读不改
│   ├── _index.md
│   ├── articles/               # 网页文章、博客、新闻、长文
│   ├── papers/                 # PDF、论文、白皮书
│   ├── repos/                  # 仓库 README、源码导读、issue 摘要
│   ├── notes/                  # 会议纪要、手工摘录、飞书文档转存
│   ├── data/                   # CSV/JSON/表格类资料
│   └── assets/                 # 图片、附件、架构图
├── wiki/                       # LLM 维护的结构化知识层
│   ├── _index.md
│   ├── overview.md
│   ├── glossary.md
│   ├── concepts/
│   ├── topics/
│   ├── entities/
│   ├── references/
│   ├── comparisons/
│   ├── timelines/
│   └── theses/
├── output/                     # 查询、报告、演讲稿等派生产物
│   ├── _index.md
│   ├── answers/
│   ├── reports/
│   └── slides/
├── logs/
│   ├── ingest-log.md
│   ├── query-log.md
│   └── lint-log.md
└── templates/
    ├── raw-source-template.md
    ├── concept-template.md
    ├── topic-template.md
    └── query-output-template.md
```

这一套结构的核心思路是：

- `inbox/` 解决“先收进来”
- `raw/` 解决“保留原文和原件”
- `wiki/` 解决“沉淀成可读知识结构”
- `output/` 解决“承接临时问答和派生产物”
- `logs/` 解决“知道最近改了什么”
- `templates/` 解决“别让每次页面结构都靠临场发挥”

### 10.1 每个目录到底放什么

#### `inbox/`

它是投递区，不是知识库。

适合放：

- 刚用浏览器插件剪藏下来的网页
- 刚下载的 PDF
- 刚导出的飞书 Markdown
- 还没来得及归类的截图或附件

不适合放：

- 长期沉淀的正式资料
- 已经写过 frontmatter 的结构化页面
- 需要被反复引用的知识页

#### `raw/`

`raw/` 是事实源头层，核心原则是：

> 保留原文、保留原件、尽量不改。

例如：

- `raw/articles/` 放网页文章转存的 Markdown
- `raw/papers/` 放 PDF 或 PDF 提取后的文本
- `raw/notes/` 放会议纪要、飞书文档导出的 Markdown
- `raw/assets/` 放图片、附件、示意图

这一层要尽量能回答“这条结论最初来自哪里”。

#### `wiki/`

`wiki/` 是结构化知识层，不是原文备份区。

我更建议拆成：

- `concepts/`：原子概念，例如 `rag.md`、`embedding.md`
- `topics/`：主题总览，例如 `llm-wiki-system-design.md`
- `entities/`：人物、组织、产品、框架
- `references/`：书单、资料单、清单页
- `comparisons/`：对比页
- `timelines/`：时间线页
- `theses/`：带判断色彩的结论页

这样做的好处是，agent 查询时不会把所有 Markdown 当成一锅粥，而能先知道哪些页在回答“是什么”，哪些页在回答“怎么拆”，哪些页在回答“怎么比较”。

#### `output/`

`output/` 是派生产物区，专门承接：

- 某次问答
- 某次阶段性报告
- 某次分享提纲
- 某次临时对比表

这些内容有价值，但不一定立刻适合并回 `wiki/`。先放 `output/`，再判断是否值得回写到 `wiki/`，会更稳。

#### `logs/`

日志建议至少拆成：

- `ingest-log.md`
- `query-log.md`
- `lint-log.md`

因为日志不只是流水账，它还承担：

- 追踪最近 ingest 了什么
- 追踪某页何时被更新
- 追踪 lint 发现过哪些结构问题

## 十一、原始资料到底怎么获取

如果不把“资料怎么进入系统”讲清楚，很多人看完 `LLM Wiki` 还是不知道第一步怎么做。

我更建议把原始资料入口分成 3 条线：

1. 网页文章
2. PDF / 论文
3. 飞书 / Lark 云文档

### 11.1 网页文章：`Obsidian Web Clipper`

这是最顺手、也最接近 Karpathy 原始建议的入口。

根据 Obsidian 官方文档，`Obsidian Web Clipper` 是官方浏览器扩展，支持把网页内容直接保存到你的 vault，并且支持：

- 从浏览器工具栏或右键菜单启动
- 默认提取正文主体，而不是整页噪音
- 使用 template、variables、filters 自定义输出
- 在页面上先高亮，再只保存高亮内容
- 用 Reader 模式先净化页面再捕获

它特别适合这类场景：

- 技术博客
- 官方文档页面
- 长文章
- 访谈和教程

一个很实用的点是，Web Clipper 可以把抓到的 metadata 一起写进页面，所以你很容易把网页落成类似下面的原始资料格式：

```markdown
---
title: "LLM Wiki"
source: "https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f"
type: "article"
captured: 2026-06-06
tags: [llm, wiki]
summary: "Karpathy 对 LLM Wiki 的整体设想。"
---
```

但要特别注意一件事：根据 Obsidian 官方说明，图片默认不会自动下载，只会保留网页 URL。也就是说：

- 如果只是普通配图，先保留外链也没问题
- 如果图片本身承载关键信息，后续要在 Obsidian 里执行 `Download attachments for current file`
- 如果你希望资料长期离线可读，最好把关键图片再落到本地

### 11.2 飞书 / Lark 云文档：`Cloud Document Converter`

有些团队知识根本不在网页和 PDF，而在飞书文档里。

这时候 `Cloud Document Converter` 就非常有用。根据它的官方仓库说明，它是一个浏览器扩展，支持：

- 把 Lark Doc 下载为 Markdown
- 把 Lark Doc 复制为 Markdown
- 在 Chrome / Edge / Firefox 安装使用

它适合这类原始资料：

- 团队内部方案文档
- 飞书会议纪要
- 调研文档
- 产品方案页

但它也有很重要的边界：

- 图片 URL 的可访问时间只有约两小时
- 流程图、UML 等复杂图形不能稳定转成纯 Markdown 语义
- 某些复杂块最终更像图片或下载附件，而不是可编辑正文

所以更稳的做法是：

- 文本主体用 `Cloud Document Converter` 变成 Markdown
- 关键图片或附件另存到 `raw/assets/`
- 如果某张图特别关键，再补一张本地截图或导出的附件

### 11.3 PDF / 论文：保留原件，再决定是否抽文本

PDF 路径和网页路径不一样。

更合理的做法通常是：

1. 先把 PDF 原件下载到 `inbox/`
2. 再移动到 `raw/papers/`
3. 如果 agent 对 PDF 直读不稳定，再补一份提取文本或手工摘要

这里不要一上来就把 PDF 内容“改写成自己的笔记”，因为那会把原始证据层和结构化知识层混在一起。

一句话总结原始资料入口：

```text
网页 -> Obsidian Web Clipper
飞书文档 -> Cloud Document Converter
PDF / 论文 -> 浏览器或官方站点下载原件
```

## 十二、`AGENTS.md` / schema 到底应该怎么设计

很多人第一次搭 LLM Wiki 会漏掉最关键的一层：不是“页面怎么写”，而是“agent 应该按什么协议维护这些页面”。

### 12.1 `AGENTS.md` 和 `CLAUDE.md` 不是一个文件

如果只从作用看，它们都像“规则文件”。

但如果从具体工具看：

- `Codex` 主要读取 `AGENTS.md`
- `Claude Code` 主要读取 `CLAUDE.md`

所以如果你希望兼容多 agent，我更推荐这种结构：

```text
AGENTS.md   -> 放通用 wiki 协议
CLAUDE.md   -> 用 `@AGENTS.md` 导入，再补少量 Claude 专属说明
```

这样可以避免你把所有规则只写在 `AGENTS.md`，结果在 Claude Code 里没被稳定采纳。

### 12.2 一个实用的 schema 至少要覆盖什么

真正有用的 schema，至少要写清楚这 6 类东西：

1. 这份 wiki 是干什么的
2. 目录结构和每层职责
3. ingest / query / lint 的操作规则
4. 页面模板和 frontmatter
5. 命名规范、链接规范、索引更新规范
6. 什么不能做

最容易失败的写法反而是：

- 大段讲理念
- 不写目录职责
- 不写更新顺序
- 不写“禁止修改 raw”

### 12.3 一份够用的 `AGENTS.md` 骨架

```markdown
# LLM Wiki Schema

## Purpose

This repository is a topic-focused LLM wiki for long-term knowledge accumulation.
The user curates sources. The agent ingests, organizes, links, updates, and lint-checks the wiki.

## Directory Contract

- `inbox/`: temporary drop zone
- `raw/`: immutable source material
- `wiki/`: compiled knowledge maintained by the agent
- `output/`: generated answers and reports
- `logs/`: append-only operation logs

## Ingest Rules

1. Classify new items into the right `raw/` subfolder
2. Preserve the original source path and URL
3. Prefer updating existing wiki pages over creating duplicates
4. Update indexes and append a log entry

## Query Rules

1. Read `_index.md` first
2. Answer from `wiki/` first, then inspect `raw/` if needed
3. Save long-form answers into `output/`

## Lint Rules

Check for orphan pages, broken links, stale claims, and missing source references.

## Non-Negotiables

- Do not overwrite source files in `raw/`
- Do not delete logs
- Do not present speculation as confirmed fact
```

对个人学习来说，这份骨架已经足够起步。后面目录变复杂了，再把它拆成更细的局部规则也不晚。

## 十三、用一个真实主题跑一次完整 ingest

光有目录和规则还不够，真正能帮你建立手感的，是看一遍“资料怎么流动”。

这里我用一个真实主题做示范：`RAG`

为什么选它：

- 有网页文章
- 有论文 PDF
- 非常适合做“概念页 + 主题页 + 对比页”

我用两类原始资料来演示：

- 网页：Hugging Face 的 `Retrieval Augmented Generation` 介绍文章
- PDF：arXiv 的 `Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks`

如果你还有团队内的飞书方案文档，也可以走同样流程，只是入口从 `Web Clipper` 换成 `Cloud Document Converter`。

### 13.1 第一步：资料先进入 `inbox/`

假设此时目录里是这样的：

```text
llm-wiki/
└── inbox/
    ├── 2026-06-06-hf-rag-blog.md
    ├── 2020-rag-paper.pdf
    └── 2026-06-06-rag-design-from-lark.md
```

它们的来源分别是：

- `2026-06-06-hf-rag-blog.md`：用 `Obsidian Web Clipper` 从网页剪藏
- `2020-rag-paper.pdf`：从 arXiv 下载原始 PDF
- `2026-06-06-rag-design-from-lark.md`：用 `Cloud Document Converter` 从飞书文档导出

这一步最重要的原则是：

- 先收进来
- 不急着命名成最终结构
- 不急着直接写成知识页

### 13.2 第二步：分类并移动到 `raw/`

当你确认主题是 `RAG` 后，agent 再把这些资料归位：

```text
llm-wiki/
├── raw/
│   ├── articles/
│   │   └── 2026-06-06-hf-rag-blog.md
│   ├── papers/
│   │   └── 2020-rag-paper.pdf
│   ├── notes/
│   │   └── 2026-06-06-rag-design-from-lark.md
│   └── assets/
└── inbox/
    └── .processed/
```

此时要顺手更新 `raw/_index.md`，让后续 agent 不用每次盲扫整个 `raw/`。

例如：

```markdown
# Raw Index

## Articles

- [2026-06-06-hf-rag-blog.md](articles/2026-06-06-hf-rag-blog.md)
  - type: article
  - summary: Hugging Face 对 RAG 的通俗介绍

## Papers

- [2020-rag-paper.pdf](papers/2020-rag-paper.pdf)
  - type: paper
  - summary: RAG 论文原始 PDF

## Notes

- [2026-06-06-rag-design-from-lark.md](notes/2026-06-06-rag-design-from-lark.md)
  - type: meeting-note
  - summary: 团队内部关于 RAG 架构的飞书设计稿
```

### 13.3 第三步：先看 `wiki/_index.md`，再决定更新旧页还是建新页

这一步是很多人最容易跳过的。

假设此时 `wiki/` 里已经有：

```text
wiki/
├── _index.md
├── concepts/
│   └── rag.md
└── comparisons/
    └── rag-vs-fine-tuning.md
```

那 agent 的优先策略应该是：

1. 先更新已有的 `rag.md`
2. 再判断这批新资料是否值得新建主题页
3. 如果只是补充证据，就不要重复再建一个“rag-intro.md”

这一步能显著减少知识库里“同义页面越长越多”的问题。

### 13.4 第四步：更新 `wiki/` 页面，而不是直接复制原文

例如，`wiki/concepts/rag.md` 可能会被更新成下面这种结构：

```markdown
---
title: "RAG"
category: "concept"
sources:
  - "../../raw/articles/2026-06-06-hf-rag-blog.md"
  - "../../raw/papers/2020-rag-paper.pdf"
  - "../../raw/notes/2026-06-06-rag-design-from-lark.md"
updated: 2026-06-06
summary: "RAG 通过在生成阶段引入外部检索，缓解参数记忆和时效性问题。"
---

# RAG

## Summary

RAG 不是单一模型结构名词，而是一类“检索 + 生成”的系统模式。

## Key Points

- 论文解释了基本范式
- 博客提供了更易懂的工程化表述
- 团队飞书方案补充了落地约束

## Sources

- [HF blog](../../raw/articles/2026-06-06-hf-rag-blog.md)
- [RAG paper](../../raw/papers/2020-rag-paper.pdf)
- [Internal design note](../../raw/notes/2026-06-06-rag-design-from-lark.md)
```

这里最关键的是：

- `wiki/` 不复制原始资料
- `wiki/` 负责综合、组织、链接
- `wiki/` 必须能追溯回 `raw/`

### 13.5 第五步：如果有必要，再生成主题页或对比页

当新资料已经不只是“补概念解释”，而是在回答更大的问题时，就值得新建主题页或对比页。

例如：

- `wiki/topics/rag-system-design.md`
- `wiki/comparisons/rag-vs-long-context.md`

这一步对应的判断标准不是“有没有新资料”，而是“有没有形成新的阅读入口或新的稳定结论”。

### 13.6 第六步：追加日志，而不是只改页面不留痕

例如 `logs/ingest-log.md` 可以写成：

```markdown
## [2026-06-06] ingest | RAG sources bundle

- Added:
  - raw/articles/2026-06-06-hf-rag-blog.md
  - raw/papers/2020-rag-paper.pdf
  - raw/notes/2026-06-06-rag-design-from-lark.md
- Updated:
  - wiki/concepts/rag.md
- Created:
  - wiki/topics/rag-system-design.md
```

这样后续你再回头看，就知道这次 ingest 到底改了什么，而不是只看到最终页面长成了什么样。

### 13.7 这次 ingest 结束后，目录应该变成什么样

```text
llm-wiki/
├── raw/
│   ├── _index.md
│   ├── articles/
│   │   └── 2026-06-06-hf-rag-blog.md
│   ├── papers/
│   │   └── 2020-rag-paper.pdf
│   └── notes/
│       └── 2026-06-06-rag-design-from-lark.md
├── wiki/
│   ├── _index.md
│   ├── concepts/
│   │   └── rag.md
│   ├── topics/
│   │   └── rag-system-design.md
│   └── comparisons/
│       └── rag-vs-fine-tuning.md
├── logs/
│   └── ingest-log.md
└── inbox/
    └── .processed/
```

这就是一轮最小但完整的 ingest。

### 13.8 把这条流程再压缩成一张图

```text
网页 -----------> Obsidian Web Clipper ----\
                                             \
PDF ------------> 下载原件 ------------------> inbox/ -> raw/ -> wiki/ -> logs/
                                             /
飞书文档 -------> Cloud Document Converter --/
```

对个人知识库来说，这张图其实比很多抽象定义都更重要，因为它真正回答了“第一步到底怎么走”。

## 十四、实操里最容易踩的坑

### 14.1 把 `raw/` 当成可编辑笔记区

一旦你开始经常性改写 `raw/`，它就不再是源头证据层了。

### 14.2 没有 `output/`，把所有问答都直接塞进 `wiki/`

这样会让 `wiki/` 很快混入大量一次性回答，页面角色会越来越模糊。

### 14.3 没有 `_index.md`，靠 agent 盲扫整个目录

小规模时还能工作，规模一大就会慢、乱、漏。

### 14.4 只写理念，不写 schema

“这个知识库希望长期演化”这种话没有错，但如果不把目录职责、页面类型、日志规则和禁止事项写出来，agent 每次都只能临场发挥。

### 14.5 只知道网页剪藏，不知道飞书和 PDF 怎么进系统

这也是很多人读完 `LLM Wiki` 之后仍然落不了地的原因。真正的原始资料从来不只是一种格式，所以入口必须提前设计。

## 十五、这篇笔记里我最想记住的几句话

最后做一个收束，把最关键的认知压缩成几条。

- `LLM Wiki` 的重点不是“让 LLM 回答问题”，而是“让 LLM 持续维护一份中间知识层”
- 它和传统 `RAG` 的差别，不在于能不能回答，而在于知识会不会沉淀成持久资产
- 三层结构里，`Raw Sources` 负责保真，`Wiki` 负责承载整理后的知识，`Schema` 负责约束维护方式
- 日常动作可以收束成 `Ingest`、`Query`、`Lint` 三件事
- 在小到中等规模下，先把 `_index.md` 和日志做好，往往比急着上复杂向量检索更重要
- 想让系统真正跑起来，必须提前设计“原始资料入口”，网页、PDF、飞书文档至少要各有一条进入 `inbox/` 的路径
- `Obsidian Web Clipper` 适合网页，`Cloud Document Converter` 适合飞书文档，PDF 则要优先保留原件
- 真正难的不是让 LLM 写几篇 Markdown，而是把 `目录职责`、`规则文件`、`索引机制`、`日志机制`、`回写机制` 一开始就定清楚
