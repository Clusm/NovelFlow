---
name: novel-outline-multi-chapter-patch
description: 大型网文大纲（500章级）的多章节协调修改 + 快速优化流程——章节完整性核查、7要素格式检测、批量patch标准化方法
version: 2
date: 2026-04-21
tags: [novel-writing, outline, web-novel]
---

# 网文大纲多章节协调修改技能

> 版本：v1（2026-04-19）
> 场景：当需要在大型网文大纲（500章级）中插入新内容、调整结构时，保持章节号整数性和多文件一致性的标准化方法

---

## 触发条件

当你需要在大纲/细纲中执行以下操作时，必须使用本技能：
- 在两个章节之间插入新的情节内容
- 新增完整章节
- 修改任意章节的章节号
- 任何涉及多文件联动修改的操作

## 实践教训：7要素格式不一致检测（2026-04-21）

**发现：** 各卷7要素格式不统一是常见问题，不能假设所有卷用相同标签名。

**扫描方法（必须分两步）：**

**第一步：用标签分布确认实际格式**
```python
all_tags = re.findall(r'【([^】]+)】', content)
from collections import Counter
Counter(all_tags).most_common(20)
```
输出示例：
```
  153x 情节主线
  153x 爽点位置      # ❌ 不是"核心爽点"
  153x 情绪价值      # ❌ 不是"情绪节奏"
  114x 冲突对手     # 缺失36章
   97x 伏笔埋设     # 缺失56章
   11x 金手指应用   # 大部分卷没有
```

**第二步：按实际标签名核查缺失**
用真实标签名（如`爽点位置`而非`核心爽点`）去检查每章节是否包含。

**常见格式差异：**
- 卷一用`金手指应用`，卷二~六可能全缺（能力成型后无需标注，非bug）
- 卷三部分章节用表格缩略版（`|| Ch201 | ...`），部分用完整7要素
- `核心爽点` vs `爽点位置`，`情绪节奏` vs `情绪价值` 混用

**章节块切分（关键）：**
```python
# ✅ 正确：用位置索引切分块
positions = [(m.start(), m.group(1)) for m in re.finditer(r'^#### Ch(\d+)：', content, re.MULTILINE)]
for i, (start, ch_num) in enumerate(positions):
    end = positions[i+1][0] if i+1 < len(positions) else len(content)
    block = content[start:end]

# ❌ 慎用 re.split 分割块（可能丢失块内内容）
# re.split(r'(?=^#### Ch\d+：)', ...) 分割结果不稳定

# 找单个章节块：
m = re.search(r'^#### ChXXX：.*?(?=^#### Ch\d+：|\Z)', content, re.MULTILINE | re.DOTALL)
```

---

## 铁律第一条：章节号必须为正整数

**绝对禁止：**
- ❌ `Ch305.5`、`Ch307.5` 等小数点章节号
- ❌ `第305.5章` 格式
- ❌ 任何非正整数的章节标记

**原因：**
1. 网文平台（起点/番茄/飞卢）不支持小数点章节号
2. 正文必须与大纲一一对应
3. 小数章节号破坏"500章"总量的语义明确性

---

## 铁律第二条：插入新内容的标准流程

当需要在 `ChN` 和 `ChN+1` 之间插入新内容时，按以下优先级选择：

### 优先级A：节末补充（最优先）

将新内容作为前一个章节（ChN）的末尾补充段落，标记为 `【节末补充·ChN内容延伸】`。

```
✅ 正确格式：

**章末钩子**：苏锦儿说："我和你一起去。"

**【节末补充·Ch305内容延伸】**：就在主角准备穿越前往第一宇宙时，系统突然发出警报：多元宇宙边缘出现"知识侵蚀者"——一种能污染修士文心的域外存在。主角开始研究知识侵蚀者的本质...
```

适用场景：新内容在5-10句以内，不构成独立章节结构。

### 优先级B：合并到相邻章节

将新内容拆分为1-3个情节点，直接并入最近章节的对应位置（不独立成节）。

```
✅ 正确格式：

**情节主线**（5个情节点）：
1. 传承仪式结束，天空出现造物主符文（原有）
2. 符文揭示第一宇宙危机（原有）
3. **【新增】多元宇宙边缘出现"知识侵蚀者"，主角决定先解决此威胁再前往第一宇宙**
4. 主角接受最终考验（原有）
5. 主角独自前往第一宇宙（原有）
```

适用场景：新内容不超过3个情节点，不需要完整的7要素结构。

### 优先级C：全文末尾追加+章号重排（最后手段）

仅当新内容需要独立完整章节结构（≥4个情节点）时，才在全文末尾追加新章节，并执行全局章号重排。

**全局章号重排的完整步骤：**

1. 确定新增章节的编号（假设在Ch305前插入 → 变成Ch306，原Ch306→Ch307…）
2. 执行重排：`sed -i 's/Ch306/Ch307/g; s/Ch305/Ch306/g' *.md`
3. **联动更新所有文件中的章节引用**：
   - [ ] `D_分卷大纲.md` — 所有章节引用
   - [ ] `B_伏笔线.md` — 伏笔回收章节引用
   - [ ] `C_爽点节奏表.md` — 爽点章节引用
   - [ ] `E_时间线.md` — 事件章节引用
   - [ ] `A_人物.md` — 角色出场章节引用
   - [ ] 细纲文件中所有 `ChXXX` 引用
4. 验证：`grep -rn "Ch[0-9]*\.[0-9]" *.md` 返回0

---

## 铁律第三条：联动修改清单

每次大纲修改后，必须检查并同步以下文件：

| 文件 | 需检查内容 |
|------|-----------|
| `D_分卷大纲.md` | 章节列表中的章节号是否连续、引用是否正确 |
| `B_伏笔线.md` | 所有伏笔的"埋设→回收"章节引用 |
| `C_爽点节奏表.md` | 爽点章节号、爽点统计数字 |
| `E_时间线.md` | 所有事件的章节号 |
| `A_人物.md` | 角色出场章节号、成长节点章节号 |
| 细纲文件 | 该卷所有章节号和章节引用 |

**关键：修改章节号时，必须用 `grep -rn "Ch305" .` 全项目搜索所有引用，再批量替换。**

---

## 铁律第四条：验证清单

每次修改完成后，执行以下验证：

```bash
# 1. 整数性检查：不允许小数点章节号
grep -rn "Ch[0-9]*\.[0-9]" --include="*.md" . | grep -v "优化设计文档"
# 预期：无输出

# 2. 重复章节号检查
grep -rn "^### 第305章" --include="*.md" .
# 预期：每个章节号仅出现1次

# 3. 全文行数统计（验证未因合并导致异常）
wc -l 细纲/*.md
# 预期：每卷细纲在预期行数范围内（参考：V1≈3000, V2≈3600, V3≈3600, V4≈3650, V5≈3600）

# 4. 伏笔引用验证（确保没有 dangling 引用）
# 人工检查 B_伏笔线.md 中的所有"埋设→回收"章节号，确认引用的章节号均存在于 D_分卷大纲.md
```

---

## 已验证的坑点（来自实践）

### 坑1：decimal章节的来源
子代理在并行执行优化任务时，会在两个整数章节之间用小数章节号（如Ch305.5）标记"插入点"。这是错误的，**大纲中永远不应该出现小数章节号**。

**教训：** 并行执行优化任务前，必须明确告知子代理"禁止使用小数章节号，所有新内容用节末补充或合并到相邻章节的方式"。

### 坑2：execute_code 报告"替换0处"但文件仍有旧内容
某些情况下（如文件编码、不可见字符、替换字符串匹配了错误的编码），execute_code 报告成功但文件中旧内容仍然存在。

**教训：** execute_code 替换后，必须用 `search_files` 或 `grep` 重新验证。执行前先读文件确认实际内容。

**执行前必须验证：**
1. 用 `repr()` 打印实际文本片段，确认特殊字符（如 `\`r`、`\`n` vs 真实换行）
2. 替换后立刻读文件验证补丁是否生效
3. 确认 Unicode 中文引号（如`""`）vs ASCII 引号（如`""`）匹配

### 坑3：修改标题后出现重复章节号
删除一个章节（如 Ch305.5）后，如果直接改成"第305章：接受考验（续）"，会和已有的 Ch305 重复。

**教训：** 删除小数章节的正确方式是"将其内容归并到前一个整数章节末尾（节末补充），然后删除整个章节标题和结构"。

### 坑4：全局章号重排容易遗漏引用
当重排多个章节时（如 Ch305→Ch306, Ch306→Ch307…），如果用简单的 `sed` 替换，会出现"先替换的数字被后续操作覆盖"的问题。

**教训：** 使用 Python 的 `re.sub` 配合 `lambda` 替换函数，或先对大数字操作再对小数字操作（从大到小重排）。

---

## 附：decimal章节修复记录（2026-04-19）

| 错误的章节号 | 修复方式 | 修复后 |
|-------------|---------|-------|
| Ch305.5 域外知识侵蚀者 | 并入Ch305末尾（节末补充） | 删除 |
| Ch307.5 知识圣剑争夺战 | 并入Ch307末尾（节末补充） | 删除 |
| Ch310.5 熵的使者 | 并入Ch310末尾（节末补充） | 删除 |
| A_人物.md Ch12.5 | 修正为Ch6（最近出场章节） | Ch6 |
| A_人物.md Ch180.5 | 修正为Ch179 | Ch179 |
| A_人物.md Ch192.5 | 修正为Ch192 | Ch192 |
| A_人物.md Ch193.5 | 修正为Ch193 | Ch193 |

修复后V4细纲：3652行，无重复章节号 ✅

---

## 附：情节主线（7要素）批量扩写标准化流程（2026-04-25）

> 场景：当大纲中多章的【情节主线】内容过少（<150字），需要批量扩写到规范标准时使用本流程。
> 适用规模：50-200章级别的批量扩写
> 核心方法：Python脚本 + 终端执行 + 分批处理（10章/批）

### 第一步：扫描实际文本并统计字数

```python
import re

path = '/path/to/细纲文件.md'
with open(path, encoding='utf-8') as f:
    lines = f.read().split('\n')

for target in range(311, 351):  # 示例：Ch311-350
    for i, line in enumerate(lines):
        if re.match(f'^#### Ch{target}：', line.strip()):
            # 找到【情节主线】标记，提取其下>引用的内容
            for j in range(i+1, min(i+50, len(lines))):
                if '**【情节主线】**' in lines[j]:
                    content = []
                    k = j + 1
                    while k < len(lines):
                        stripped = lines[k].strip()
                        if re.match(r'\*\*【\w+】\*\*', stripped) or re.match(r'^####', stripped) or stripped == '---':
                            break
                        if stripped.startswith('>'):
                            content.append(stripped[1:].strip())
                        k += 1
                    text = ' '.join(content)
                    char_count = len(text)
                    status = 'OK' if char_count >= 150 else 'WEAK'
                    print(f"Ch{target}: {char_count} chars [{status}]")
                    break
            break
```

### 第二步：编写批量扩写Python脚本

每次处理10章，分批进行。脚本结构：

```python
path = '/path/to/细纲文件.md'
with open(path, encoding='utf-8') as f:
    content = f.read()

expansions = [
    # 格式：(章节号, 原文（必须精确匹配）, 扩写后内容)
    ("Ch311",
     '情节主线原文...',
     '2040年10月，地点。人物做某事。他/她说："对话。"具体描写：发生了什么，结果如何，对后续的影响。'),
    # ... 最多10条
]

modified = content
count = 0
for ch, old, new in expansions:
    if old in modified:
        modified = modified.replace(old, new, 1)
        count += 1
        print(f"OK {ch}")
    else:
        print(f"MISSING {ch}")  # 原文匹配失败，需重新扫描

print(f"\nExpanded: {count}/10")
with open(path, 'w', encoding='utf-8') as f:
    f.write(modified)
print("Saved.")
```

### 第三步：处理MISSING章节

当脚本报告MISSING时，说明原文与扫描结果不一致（如已有其他修改）：
1. 用 `execute_code` 重新扫描该章节的实际【情节主线】文本
2. 用 `repr()` 打印精确文本
3. 更新脚本中的old字符串，重新运行

```python
# 扫描精确原文
for i, line in enumerate(lines):
    if re.match(f'^#### Ch{target}：', line.strip()):
        for j in range(i+1, min(i+50, len(lines))):
            if '**【情节主线】**' in lines[j]:
                content = []
                k = j + 1
                while k < len(lines):
                    stripped = lines[k].strip()
                    if re.match(r'\*\*【\w+】\*\*', stripped) or re.match(r'^####', stripped) or stripped == '---':
                        break
                    if stripped.startswith('>'):
                        content.append(stripped[1:].strip())
                    k += 1
                print(repr(' '.join(content)))  # 精确文本
                break
        break
```

### 第四步：验证扩写结果

扩写后重新执行第一步的字数统计脚本，确认所有章节≥150字。

---

### 已验证的坑点（情节主线扩写专项）

**坑0：提取函数与替换逻辑错位（最隐蔽）**

在本次实践中，发现了一个极其隐蔽的bug：用于"统计字数"的提取函数，与用于"执行替换"的提取逻辑不同，导致：
- 统计脚本报告某章"150字"（使用了某种启发式截断）
- 但该章实际在文件中只有120字
- 替换脚本用统计脚本的字数去文件中查找，找不到，报告"MISSING"
- 操作者以为是"已有内容被改过"，实际上统计函数本身有误

**症状：** 批量脚本反复报告MISSING，但文件内容实际上从未被修改过。验证字符计数与文件实际内容不符。

**根因：** 统计脚本中的 `re.search(r'\n\s*\*\*【', text[start:])` 在某些章节的plot文本内部（而非边界处）就匹配到了 `\n\s*\*\*【`，导致截断位置错误。

**解决：** 对每个需要修复的章节，直接在文件级别定位：
1. 用 `m = re.search(r'^#### ChXXX：', content)` 找章节头
2. 用 `next_m = re.search(r'^#### Ch\d+：', content[ch_m.end():])` 找下一章
3. 在 `[ch_m.start():ch_m.start() + next_m.start()]` 区间内找 `**【情节主线】**`
4. 在该区间内做精确替换

```python
# 绝对位置替换（最可靠）
ch_m = re.search(rf'^#### Ch{ch}：', content, re.MULTILINE)
next_m = re.search(r'^#### Ch\d+：', content[ch_m.start() + 10:], re.MULTILINE)
ch_end = ch_m.start() + 10 + next_m.start() if next_m else len(content)
ch_section = content[ch_m.start():ch_end]  # 在此区间内查找和替换

# 在ch_section内提取plot文本
plot_m = re.search(r'\*\*【情节主线】\*\*', ch_section)
after = ch_section[plot_m.end():]
field_m = re.search(r'\n\s*\*\*【', after)
plot_text = after[:field_m.start()] if field_m else after

# 提取>行内容
content_parts = [l.strip()[1:].strip() for l in plot_text.strip().split('\n') if l.strip().startswith('>')]
old_text = ''.join(content_parts)

# 在modified的ch_section范围内替换
if old_text in ch_section:
    abs_pos = ch_m.start() + ch_section.find(old_text)
    modified = modified[:abs_pos] + new_text + modified[abs_pos + len(old_text):]
```

**核心原则：** 永远在"章节目录区间"内做替换，不要跨章节查找。

---

**坑1：Python字符串引号冲突**
脚本中包含中文引号 `""` 或英文引号 `"` 的内容时，Python字符串字面量会报错。

**解决：** 中文引号直接写在字符串内（不需要转义），英文单双引号交替使用避免冲突。或使用原始字符串 `r'''...'''`。

**坑2：文件实际文本与扫描结果不一致**
曾发现Ch298的实际文本是 `"华国向比邻星方向发送了回复信号..."` (45字)，但之前某次扩写后的文本被意外写入了Ch295的槽位（280字）。同一文件中存在两处相同的扩写内容。

**解决：** 用 `execute_code` 直接定位并打印 `repr()` 精确文本，再用字符串替换修复。

**坑3：批量扩写后章节内容重复**
当同一个脚本运行两次（或多批次脚本有重叠章节）时，可能导致内容重复追加。

**解决：** 严格控制每批章节范围（10章），避免重叠。每批完成后立即验证。

**坑4：扩写文本未生效（替换0处）**
`content.replace(old, new, 1)` 报告OK但文件内容未变时，可能是old字符串包含了不可见字符或Unicode空格。

**解决：** 用 `repr(old)` 确认精确字符，特别是全角空格 vs 半角空格的差异。
