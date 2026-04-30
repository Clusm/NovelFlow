# 项目状态追踪系统

## 🎯 设计目标

全局适用的项目状态管理，自动追踪每个小说项目的生命周期。

---

## 📊 项目状态定义

| 状态值 | 含义 | 显示优先级 |
|--------|------|-----------|
| `planning` | 规划中 | 低 |
| `active` | 进行中 | 高 |
| `paused` | 暂停中 | 中 |
| `completed` | 已完结 | 低 |
| `abandoned` | 已放弃/烂尾 | 低 |

---

## 📁 状态文件位置

每个项目根目录必须包含 `status.json`：

```
projects/
├── _global/
│   ├── status_template.json    # 全局模板
│   └── status_rules.md         # 本规则文件
├── 项目A/
│   └── status.json             # 项目A状态
└── 项目B/
    └── status.json             # 项目B状态
```

---

## 🔄 状态流转图

```
[planning] → [active] → [completed]
     ↓            ↓
  [abandoned]  [paused] → [active]
                      ↓
                 [abandoned]
```

---

## ✅ 完结判断标准

**必须满足以下条件之一才能标记为 `completed`：**

| 条件 | 说明 | 必需 |
|------|------|------|
| `userDeclared` | 用户明确宣布完结 | ✅ |
| `allArcsFinished` | 所有故事线/大篇章已完成 | 建议 |
| `allForeshadowingResolved` | 所有伏笔已回收 | 建议 |
| `hasEndingChapter` | 有明确的结局章节 | 建议 |

**禁止完结的情况：**
- ❌ 未填完的坑超过3个
- ❌ 重要角色线未完成
- ❌ 用户未明确表态

---

## ❌ 放弃判断标准

**标记为 `abandoned` 的条件：**

| 条件 | 说明 |
|------|------|
| 用户主动放弃 | 用户明确说"不写了"、"放弃" |
| 超过60天未更新 | 项目处于paused状态超过60天 |
| 连续3次创作失败 | 同一项目连续3次无法满意续写 |

**放弃项目处理：**
- 自动从 INDEX.md 活跃项目列表移除
- 保留存档（不删除文件）
- 禁止复用该项目的框架/文笔

---

## 🔧 状态更新规则

### 何时必须更新状态？

| 触发事件 | 必须更新内容 |
|---------|-------------|
| 用户开始新项目 | `status: planning → active` |
| 用户停止创作超过7天 | `status: active → paused` |
| 用户恢复创作 | `status: paused → active` |
| 用户宣布完结 | `status: active → completed` |
| 用户宣布放弃 | `status: any → abandoned` |
| 暂停超过60天 | `status: paused → abandoned` |

### History 记录格式

每次状态变更必须记录到 `history` 数组：

```json
{
  "timestamp": "2026-04-27T06:30:00Z",
  "from": "active",
  "to": "completed",
  "reason": "用户宣布完结",
  "note": "所有伏笔已回收，大结局已完成"
}
```

---

## 📋 INDEX.md 自动索引规则

INDEX.md 必须包含项目状态索引，自动分类显示：

```markdown
## 📁 项目状态总览

### 🔄 进行中 (active)
| 项目 | 类型 | 字数 | 最后更新 |
|------|------|------|----------|
| xxx | 都市 | 30万 | 2026-04-25 |

### ⏸️ 已暂停 (paused)
| 项目 | 暂停原因 | 暂停日期 |
|------|---------|----------|
| xxx | 卡文 | 2026-04-20 |

### ✅ 已完结 (completed)
| 项目 | 总字数 | 完结日期 |
|------|--------|----------|
| xxx | 100万 | 2026-04-15 |

### ❌ 已放弃 (abandoned)
| 项目 | 放弃原因 | 放弃日期 |
|------|---------|----------|
| xxx | 用户放弃 | 2026-04-10 |
```

---

## 🔍 完结检测逻辑（上下文管理器职责）

每次对话开始时，上下文管理器必须检查：

```
□ 是否有项目超过60天未更新？
    → 是 → 自动标记为 abandoned
□ 是否有项目用户宣布放弃？
    → 是 → 自动标记为 abandoned
□ 是否有项目达到完结标准？
    → 是 → 询问用户是否标记为 completed
```

---

## ⚠️ 强制规则

1. **禁止隐藏状态**：所有项目必须有准确的状态值
2. **禁止僵尸项目**：paused 超过60天自动转为 abandoned
3. **禁止复用失败**：abandoned 项目的框架/文笔禁止复用
4. **禁止手动覆盖自动状态**：自动检测结果必须保留

---

## 📝 全局模板使用说明

### 创建新项目时：

1. 复制 `projects/_global/status_template.json` 到项目文件夹
2. 填写 `projectName`、`targetChapters`、`targetWords`
3. 设置 `status: "planning"`
4. 在 INDEX.md 添加项目条目

### 状态变更时：

1. 更新 `status.json` 中的 `status` 字段
2. 添加 `history` 记录
3. 更新 `updatedAt` 时间戳
4. 同步更新 INDEX.md

---

*最后更新：2026-04-27 | 新增：全局项目状态追踪系统*