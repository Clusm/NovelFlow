---
title: AI多Agent写作流程管控
created: 2026-04-30
updated: 2026-04-30
type: concept
tags: [概念, AI创作, 工作流, 多Agent, 流程管控, AIGC]
sources: [raw/待处理_2026-4-30/rules/L1_规则总览.md, raw/待处理_2026-4-30/rules/L2.5_强制执行机制.md, raw/待处理_2026-4-30/rules/L2.6_强制自检清单.md]
---

# AI多Agent写作流程管控

> 来源：Trae AI 多Agent写作系统的 CHECKPOINT + 违规代码管控体系（2026-04-28）
> 用途：AI辅助写作时的流程强制管控，防止跳步、漏审、不同步

## 核心问题

多Agent协作写网文时，常见失败模式：
- **跳步**：没读大纲就创作，没审核就提交
- **漏审**：章节写完不检查，直接进入下一章
- **不同步**：memory/index 不同步，导致后续Agent上下文错乱
- **复用失败**：失败的项目设定/文风被继续沿用

## CHECKPOINT 体系

CHECKPOINT 是强制检查点，每个节点必须通过才能继续。

### 新项目创建（0-5）

|| 检查点 | 必须通过 | 违规代码 |
||--------|---------|---------|
|| CP-0 | 触发词识别（写小说/创建项目） | VIOLATION_STEP_JUMP |
|| CP-1 | 参数确认（章节数/字数/平台/风格） | VIOLATION_STEP_JUMP |
|| CP-2 | 上下文管理器调用 | VIOLATION_SKIP_CONTEXT |
|| CP-3 | 网文架构师调用（设定大纲/人设/世界观） | VIOLATION_SKIP_SKILL |
|| CP-4 | 网文创意总监调用（风格/节奏/爽点） | VIOLATION_SKIP_SKILL |
|| CP-5 | INDEX.md 状态同步 | VIOLATION_SKIP_SYNC |

### 章节创作（A-F）

|| 检查点 | 必须通过 | 违规代码 |
||--------|---------|---------|
|| CP-A | 上下文恢复（读outline/characters/foreshadow） | VIOLATION_SKIP_CONTEXT |
|| CP-B | 创作前验证（大纲锚定/人物一致性） | VIOLATION_STEP_JUMP |
|| CP-C | 原创作家调用 | VIOLATION_SKIP_SKILL |
|| CP-D | 审核官调用（逐章审核） | VIOLATION_SKIP_AUDIT |
|| CP-E | 审核结果处理（修改/patch/通过） | VIOLATION_STEP_JUMP |
|| CP-F | 保存与 INDEX/memory 同步 | VIOLATION_SKIP_SYNC |

### 审核环节（C1-C8）

每章审核必须逐项检查：
- 大纲锚定 → 人设一致性 → 逻辑自洽 → 爽点布局 → 禁区词扫描 → 字数验证 → 情绪节奏 → 收尾钩子

## 违规代码速查

|| 代码 | 类型 | 处理 |
||------|------|------|
| VIOLATION_STEP_JUMP | 跳步执行 | 立即停止 → 回退 → 重新执行 |
| VIOLATION_SKIP_SKILL | 跳过技能调用 | 强制补调用 |
| VIOLATION_SKIP_AUDIT | 跳过审核 | 补审后才能继续 |
| VIOLATION_SKIP_SYNC | 跳过同步 | 先同步才能继续 |
| VIOLATION_SKIP_CONTEXT | 跳过上下文恢复 | 强制恢复才能继续 |
| VIOLATION_REUSE_FAILED | 复用失败项目 | 立即停止 → 报告违规 |

## 执行前自检流程

**新项目创建前必检**：
```
□ [CP-0] 识别到触发词（写小说/新建项目）
□ [CP-1] 弹出参数询问并确认
□ [CP-2] 调用上下文管理器
□ [CP-3] 调用网文架构师（大纲/人设/世界观）
□ [CP-4] 调用网文创意总监（风格/节奏）
□ [CP-5] 同步 INDEX.md 状态
全部通过才能继续
```

**章节创作前必检**：
```
□ [A1] 上章审核结论为"通过"
□ [A2] 调用上下文管理器
□ [A3] 读 memory/outline.md
□ [A4] 读 memory/foreshadow.md
□ [A5] 读 memory/characters.md
□ [A6] 上下文恢复正确
全部通过才能开始创作
```

## 核心禁止规则

|| 禁止项 | 说明 |
||--------|------|
| 禁止跳步 | 必须按 CP-0 → CP-5 顺序执行 |
| 禁止跳过审核 | 每章必须经过 C1-C8 审核 |
| 禁止复用失败框架 | "未续写=不满意"，失败项目的框架/文风禁止复用 |
| 禁止复用之前设定 | 新项目禁止使用之前任何项目的设定 |
| 禁止不同步 | INDEX 和 memory 必须同步更新 |
| 禁止无设定却不联网 | 无设定时必须联网学习各平台爆款后随机生成 |

## 执行流程图

```
任何操作开始
    ↓
读取 L2.5 强制执行机制
    ↓
执行前自检（L2.6）
├─ 通过 → 继续执行
└─ 未通过 → 暂停+报告
    ↓
执行操作
    ↓
执行后自检（L2.6）
├─ 通过 → 交接文档
└─ 未通过 → 补执行+重新自检
    ↓
记录强制日志
    ↓
继续下一步
```

---

*来源：~/ainovel/wiki/raw/待处理_2026-4-30/rules/L1_规则总览.md*
