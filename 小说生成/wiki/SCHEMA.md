---
title: Wiki Schema
created: 2026-04-14
updated: 2026-05-01
type: meta
tags: [schema, wiki, conventions]
---
# Wiki Schema — 网络文学写作知识库

## Domain

网文写作方法论与AI辅助创作系统：流派分析、情节工程、人物塑造、AI辅助写作、投稿运营、多Agent协作技能体系。

## Directory Structure

```
wiki/
├── concepts/       # 写作概念与方法论
├── comparisons/    # 平台/流派对比分析
├── entities/       # 流派/模组/人物原型
│   ├── genres/
│   ├── character-archetypes/
│   └── plot-types/
├── agents/         # AI Agent角色定义 (9个)
├── rules/          # 项目规则与强制流程 (13个)
│   └── project_rules/
├── skills/         # 分层多Agent技能树 (80个SKILL.md)
│   ├── 网文架构师/
│   ├── 网文原创作家/
│   ├── 网文创意总监/
│   ├── 网文审核官/
│   ├── 上下文管理器/
│   └── 拆书分析师/
├── meta/           # 元信息（调用规则/类型定义）
├── references/     # 参考资料
├── raw/            # 未处理源文件 (待清洗后删除)
├── raw_articles/   # 原始文章
├── index.md
├── SCHEMA.md
└── log.md
```

## Conventions

- 文件名：中文小写，英文用连字符，如 `退婚打脸模组.md`、`ai章节指令模板.md`
- 每个 wiki 页面以 YAML frontmatter 开头（见下方）
- 使用 `[[页面名]]` 双向链接，新建页面必须至少链接到 2 个已有页面
- 更新页面时同步更新 `updated` 日期
- 所有页面须登记到 `index.md`
- 所有操作须追加到 `log.md`

## Frontmatter

```yaml
---
title: 页面标题
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary | template | framework | agent | rule | skill | meta
tags: [from taxonomy below]
sources: [raw/articles/源文件.md]
---
```

## Tag Taxonomy

**流派与基调**（type-tags）
- 流派：升级流、穿越流、系统流、无敌流、悬疑流、恋爱流、职场所
- 基调：热血、逆袭、轻松、虐心、搞笑、治愈
- 人设：废柴逆袭、天才崛起、隐藏实力、扮猪吃虎

**情节模组**（plot-tags）
- 爽感模组：退婚打脸、成长逆袭、扮猪吃虎、装逼打脸、绝处逢生
- 情节元素：系统面板、功法传承、副本冒险、秘境探宝、势力争斗
- 节奏：起承转合、高潮设计、悬念铺设、伏笔回收

**人物**（char-tags）
- 主角：废柴型、天才型、腹黑型、热血型
- 配角：老爷爷、师姐/师兄、对手/反派、工具人
- 人物关系：师徒、兄弟/姐妹、敌对、羁绊

**写作工具**（tool-tags）
- AI提示词、结构模板、拆书方法、拉片技巧
- 章节指令、开篇公式、卡文急救

**运营与投稿**（biz-tags）
- 投稿平台、签约策略、全勤攻略、读者互动

## Page Thresholds

- **创建 entity**（流派/模组/模板）：出现在 2+ 来源中，或在一个来源中占核心位置
- **创建 concept**（写作概念）：定义清晰、有多个应用场景
- **创建 comparison**：对比维度有价值（如"升级流 vs 无敌流"）
- **添加进已有页面**：当某内容在多个来源中反复提及
- **不创建**：一次性提及、无上下文的新词

## Entity Pages（流派/模组/模板）

- 定义与核心机制
- 经典案例（引用源文件）
- AI 写作提示词要点
- 适用场景与禁忌

## Concept Pages（写作概念）

- 定义与解释
- 在网文中的具体应用
- 相关流派/模组（[[wikilinks]]）
- 来源

## Agent Pages（AI角色）

- 角色定位与职责范围
- 输入/输出规范
- 协作接口（与其他Agent的交互协议）
- 禁止行为（OOC约束）

## Rule Pages（项目规则）

- 规则层级（L1-L5）
- 强制执行条件
- 违规处理机制
- 关联Agent角色

## Skill Pages（Agent技能）

- 技能定位（所属Agent/层级）
- 前置条件与触发条件
- 执行步骤（Step-by-step）
- 输出格式与验证标准
- 已知坑点与注意事项

## Comparison Pages

- 对比对象与原因
- 维度对比（表格）
- 各自优劣
- 选择建议

## Update Policy

新信息与已有内容冲突时：
1. 以较新来源为准
2. 真正矛盾时注明双方立场与来源日期
3. 在 frontmatter 标记 `contradictions: [page-name]`
4. 在 lint 报告中标红

## Log Policy

- 每次 ingest/update/query/lint 追加一条到 log.md
- 格式：`## [YYYY-MM-DD] action | subject`
- 超过 200 条时rotate为 `log-YYYY.md`
