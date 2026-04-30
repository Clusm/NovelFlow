---
name: patching-large-novel-outline
description: Patch large web novel outline files where standard old_string patterns repeat across hundreds of chapters
---

# 大纲文件精准Patch技能

## 核心问题

在《文道至尊》大纲这类大型网文大纲文件中，`**爽点**：`、`**钩子**：`、`**写作提示**：`等结构标记在每个章节重复出现，导致`patch`工具的`old_string`匹配失败（常出现40-525个匹配）。

## 关键发现

1. **结构标记不是unique identifier** — 即便用15-20行上下文，包含`**爽点**：`等通用模式的文本在300章文件中仍然重复出现多次
2. **replace_all行为诡异** — 在有重复时会"跳跃式"替换，产生重复内容
3. **Python是最终解决方案** — 用Python的字符串精确查找+替换，可以完全避免这个问题

---

## 执行前必查：验证任务列表的准确性

**发现（2026-04-21）：任务列表中的"待写场景"≠ 实际缺内容场景**

场景补丁任务常基于历史规划文档生成，但文档更新可能滞后于大纲实际内容。**直接读文件确认**，避免为已完成的场景白写补丁。

### 验证步骤（每个目标章节花3分钟读文件）

1. `grep -n "Ch380" 细纲.md` — 找章节行
2. 读章节附近20-40行 — 检查是否有完整7要素正文
3. 确认标准：**有`### ChXX.5：标题`且含`【情节主线】`+`【冲突对手】`+`【章末钩子】`= 已完成，无需重写**

### 典型误区

| 情况 | 结果 |
|------|------|
| 任务列表说"缺Ch280正文" | 实际Ch280.1/280.2/280.3三段已完整 |
| 任务列表说"缺Ch320正文" | 实际Ch320.5支教场景正文完整 |
| 任务列表说"缺Ch399.5正文" | 实际沈听雪钢琴场景已完整7要素 |

**教训：读文件3分钟 vs 重写补丁30分钟。永远先验证。**

## 推荐工作流

### Step 1: 读取目标区域（精确到行）

```python
with open('/home/tao/小说流/文道至尊_大纲_优化版.md', 'r') as f:
    content = f.read()
# 先找插入/替换点的精确位置
idx = content.find('第XX章：精确标题')
# 打印周围200字符确认
print(repr(content[idx-200:idx+100]))
```

### Step 2: 构建包含唯一特征的old_string

- 包含章节的**精确标题**（含标点符号）
- 包含**情节**段落中独有的句子（不用`**爽点**：`这类通用标记）
- 包含章节之间的`**---**`分隔符（关键！）
- 长度至少覆盖`---`到下一个`---`的完整章节块
- **双`---`章节边界**：两个相邻的`---`行（即`---\n---`）比单个`---`更少见，更适合作为插入点
- **fractional章节插入**：用章节间双`---`边界插入ChXX.5格式的新章节最可靠

### Step 3: 验证marker唯一性

```python
count = content.count(marker)
print(f"Marker found {count} time(s)")
if count != 1:
    print("NOT UNIQUE - need more context")
```

### Step 4: 执行替换

```python
new_content = content.replace(marker, new_section, 1)  # 第三个参数1=只替换第一个
with open('大纲.md', 'w') as f:
    f.write(new_content)
```

## 小数章节号清理（重要补充）

**发现：插入的.5场景可能产生两种格式，需要分两次清理**

### 格式A：独立节（`### ChXX.5：`开头）
出现在汇总章节表后面，有完整的7要素块。清理方式：
```python
# 找所有### ChXX.X：开头的块
decimal_sections = re.findall(
    r'(### Ch(\d+)\.(\d+)：[^\n]+\n+.*?)(?=\n+### Ch\d|\n+---\n+## |\Z)',
    content, re.DOTALL
)
for match, ch, sub in decimal_sections:
    # 找前一个整数章节的位置
    prev_int_pattern = rf'\| Ch{ch} \|'
    prev_matches = list(re.finditer(prev_int_pattern, content))
    prev_pos = prev_matches[0].start()
    
    # 找这个ChXX.X块的起止
    section_start = content.find(f'### Ch{ch}.{sub}')
    next_sep = content.find('\n---\n', section_start)
    if next_sep < 0:
        next_sep = len(content)
    
    section_block = content[section_start:next_sep]
    content = content[:section_start] + content[next_sep:]
    
    # 插入到前一个整数章节的章末钩子之后
    hook_match = re.search(r'(\*\*【章末钩子】\*\*.*?)(\n+---|\n+### |\Z)',
                          content[prev_pos:], re.DOTALL)
    if hook_match:
        insert_pos = prev_pos + hook_match.end(1)
        content = content[:insert_pos] + '\n\n---\n\n' + section_block.strip() + content[insert_pos:]
```

### 格式B：表格内联行（`| ChXX.5 |` 开头）
出现在汇总章节表格中。需要先把行扩展为完整章节块，再从表格删除该行：
```python
# 在| Ch466 | ...之后插入| Ch466.5 | ...整行内容扩展的章节块
# 然后从表格中删除该行
```

### 两阶段清理流程（必须！）
1. **第一阶段**：插入场景时，用Python在同一次脚本运行中完成插入+小数章节清理
2. **第二阶段**：验证时重新扫描，若还有残留小数章节（说明该文件之前有小数章节未清理干净），再次执行清理脚本

### 为什么需要两遍？
因为V4/V5等卷中本身存在旧的小数章节（Ch466.5/Ch469.5等表格内联行），这些在第一遍清理时不会被自动处理——必须逐个手动处理或写专门的inline清理脚本。

### V1/V2卷的特殊格式问题：3链 vs 4链章节头

**发现（2026-04-22）：早期卷可能用`### ChX`（3个#）作为未展开原始标记，而后续卷用`#### ChX`（4个#）作为展开标记。**

这会导致`grep -c "^#### Ch"`只统计4链格式，遗漏3链格式的章节，造成"缺1章"的误报。

**诊断方法：**
```python
# 同时统计3链和4链
ch3 = set(int(m.group(1)) for m in re.finditer(r'\n### Ch(\d+)：', content))
ch4 = set(int(m.group(1)) for m in re.finditer(r'\n#### Ch(\d+)[：:]', content))
all_chs = ch3 | ch4  # 取并集
missing = sorted(set(range(1, 101)) - all_chs)
print(f'3链: {len(ch3)}, 4链: {len(ch4)}, 合并: {len(all_chs)}')
```

**修复方法：**
```python
# 将第一个### ChX：升级为#### ChX：
old = '### Ch1：'
new = '#### Ch1：'
content = content.replace(old, new, 1)  # 只替换第一个，避免误伤其他
```

**验证：** 替换后`grep -c "^#### Ch"`应返回正确数字（100/150等）。

### 多章节块文件的结构陷阱（2026-04-25 新发现）

**发现：《数学太凶猛》第一卷文件有多个章节扩展区域叠加**

文件结构不是简单的Ch1→Ch2→Ch3线性排列，而是包含：
1. **汇总表格**（行数统计）
2. **Ch1独立块**（4链格式，完整7要素）
3. **Ch2独立块**（4链格式，完整7要素）
4. **`### Ch2-10：详细扩展`区块**（包含Ch2-Ch10详细展开）
5. **`#### Ch3-10：加速扩展`区块**（包含Ch3-Ch10简化展开）
6. **Ch11-Ch100独立块**

**症状**：章节不按字节位置排列——Ch3的字节位置(1796) > Ch4的字节位置(1379)。`ch_pos[3] > ch_pos[4]`。

**诊断方法**：
```python
# 检查章节是否按数值顺序排列
headings = [(int(m.group(1)), m.start()) for m in re.finditer(r'^#### Ch(\d+)：(.*)$', content, re.MULTILINE)]
sorted_by_pos = sorted(headings, key=lambda x: x[1])
in_order = all(sorted_by_pos[i][0] < sorted_by_pos[i+1][0] for i in range(len(sorted_by_pos)-1))
print(f"章节按数值顺序排列: {in_order}")  # 如果False，说明有错位

# 打印错位情况
for i in range(len(sorted_by_pos)-1):
    ch_curr, pos_curr = sorted_by_pos[i]
    ch_next, pos_next = sorted_by_pos[i+1]
    if ch_curr > ch_next:
        print(f"  错位: Ch{ch_curr}(pos={pos_curr}) > Ch{ch_next}(pos={pos_next})")
```

**修复方法**：用Python按数值顺序重建：
```python
# 提取所有#### ChN：块
chapters = {}
for m in re.finditer(r'^#### Ch(\d+)：(.*)$', content, re.MULTILINE):
    ch_num = int(m.group(1))
    start = m.start()
    end = m.end()
    # 找下一个#### Ch的位置作为块结束
    next_m = re.search(r'^#### Ch\d+：', content[end:], re.MULTILINE)
    block_end = end + next_m.start() if next_m else len(content)
    chapters[ch_num] = content[start:block_end]

# 验证无重复
counts = {}
for ch in chapters:
    counts[ch] = counts.get(ch, 0) + 1
dups = {k: v for k, v in counts.items() if v > 1}
print(f"重复章节: {dups}")  # 必须为空

# 按数值顺序重建
new_content = content[:headings[0][1]]  # 保留文件头部（表格等）
for ch in sorted(chapters.keys()):
    new_content += chapters[ch]
new_content += "\n"

with open(path, 'w', encoding="utf-8") as f:
    f.write(new_content)
```

### 废弃章节标记行导致块边界计算错误

**发现（2026-04-25）：`#### Ch3-10：加速扩展`等废弃标记行被`re.finditer`误认为章节块起点**

该行不是真正的章节（`#### Ch3：`才是），但`####`格式相同，会干扰块边界判断。

**诊断**：
```bash
grep -n "^#### Ch" 文件.md  # 找所有疑似章节头
# 真正的章节：#### Ch3：标题（有标题）
# 废弃标记：#### Ch3-10：加速扩展（无标题，是汇总行）
```

**修复**：删除废弃标记行（`#### Ch3-10：加速扩展`等），保持文件干净。

### 表格行章节：只有`|| ChXXX |`但无`####`展开段

**发现（2026-04-22）：部分章节在汇总表格中有行`|| Ch520 | 名场面 | ...`，但文件里没有对应的`#### Ch520`展开段落。**

`grep -c "^#### Ch"`会漏计这些，导致"缺1章"误报。

**诊断：**
```python
# 找所有表格行
table_rows = re.findall(r'\|\| Ch(\d+) \|', content)
# 找所有####展开
expanded = set(int(m.group(1)) for m in re.finditer(r'\n#### Ch(\d+)[：:]', content))
# 表格有但展开没有的
in_table_not_expanded = [int(x) for x in table_rows if int(x) not in expanded]
print(f'表格有但无展开: {in_table_not_expanded}')
```

**修复：** 生成完整的`#### ChXXX`7要素段落，插入到下一个有`#### ChXXX+1`的位置之前。

### 子代理生成内容的叙事冲突检测

**发现（2026-04-22）：子代理批量生成的章节（如Ch301-Ch350）可能与已有章节（Ch294-Ch300）产生叙事冲突。**

症状：
- 已有Ch294-Ch300描写外星人首次接触，子代理从Ch301又描写一次"外星人接触"
- 子代理在Ch350写"周牧之去世"，但V4文件里周牧之还活着（V4 Ch481是"周数的追悼"→ 周数是主角，周牧之应活着）

**检测方法：** 生成章节后，检查子代理生成内容与文件已有内容的关键人物状态：
```python
# 检查关键人物在子代理生成内容和文件末尾的一致性
# 例如：周牧之在Ch350的状态
if '周牧之在睡梦中安详离世' in generated_ch350:
    # 检查V4开头是否周牧之还活着
    if '周牧之' not in v4_content[:5000]:
        print('警告：V4主角状态矛盾！')
```

**教训：** 跨卷叙事一致性在批量补丁时必须检查。子代理不知道其他卷的内容，需要主控会话在派发任务前提供叙事走向说明。

## ⚠️ 绝对禁区：subagent不得使用patch工具修改细纲文件

**教训来源：V4文件损坏事件（2026-04-22）**

subagent在执行细纲展开任务时，自行使用`patch`工具向Ch380位置插入内容。由于模板段落结构高度相似（#### ChXXX：标题 → 6个【】字段 → --- 分隔），fuzzy matching匹配到了错误位置，导致：
- Ch378/Ch379 出现两次（重复插入）
- Ch380 内容缺失
- 后续章节整体错位

**根本原因**：subagent没有`patching-large-novel-outline`这个skill的上下文，不知道patch对这类文件是危险的。

### 解决方案：子代理任务必须使用Python整体重建

当需要批量修改（如：为100个表格章节生成####展开段落），子代理应：
1. 用Python读取整个文件到内存
2. 在Python中解析文件结构、生成内容、注入新章节
3. 用Python一次性写回完整文件

```python
# 子代理应使用的安全模式
with open(path, encoding="utf-8") as f:
    content = f.read()

# 解析、修改全部在Python中完成
new_content = modify_all_chapters(content)

# 一次性写回
with open(path, "w", encoding="utf-8") as f:
    f.write(new_content)
```

绝对不要：`patch(old_string=..., new_string=..., path=...)` 在大文件上逐个替换。

## 验证步骤

插入后立即验证：

1. `grep -c "ChXX.5" 大纲.md` 确认只有1个实例
2. `wc -l 大纲.md` 确认行数增加了正确量
3. 读取插入点附近确认内容连续无重复
4. **全面扫描** `grep "Ch[0-9]+\.[0-9]" 大纲.md` 确认所有小数章节已清理
5. **去重扫描** `grep "^#### Ch" 大纲.md | awk '{print $2}' | sort | uniq -c | awk '$1>1{print}'` 确认无重复章节号

### V4文件损坏后的Python整体重建法

当文件已被patch损坏（重复章节/错位），用以下Python流程重建：

```python
import re

path = "第四卷_问道（Ch351-500）_7要素版.md"
with open(path, encoding="utf-8") as f:
    content = f.read()

# 按章节头精确切分（不能用双换行符——章节间有时只有单换行）
raw_chunks = re.split(r'(?=\n#### Ch\d)', content)

chapters = {}  # ch_num -> text
for chunk in raw_chunks:
    chunk = chunk.strip()
    if not chunk.startswith('#### Ch'):
        continue
    m = re.match(r'^#### Ch(\d+)：', chunk)
    if not m:
        continue
    ch_num = int(m.group(1))
    # 过滤掉范围外的垃圾条目
    if 351 <= ch_num <= 500:
        chapters[ch_num] = chunk

# 填缺失章节
for ch in range(351, 501):
    if ch not in chapters:
        chapters[ch] = gen_placeholder(ch)

# 去重验证
counts = {}
for ch in chapters:
    counts[ch] = counts.get(ch, 0) + 1
dups = {k: v for k, v in counts.items() if v > 1}
print(f"Duplicates: {dups}")  # 必须为空

# 整体重建
lines = [header] + [chapters[ch] for ch in sorted(chapters.keys())] + [""]
new_content = '\n'.join(lines)

with open(path, "w", encoding="utf-8") as f:
    f.write(new_content)
```

关键：`re.split(r'(?=\n#### Ch\d)', content)` 用前瞻断言在每个`\n#### Ch`前切分，保证每个块完整且无遗漏。

## 已知文件结构

- `/home/tao/小说流/文道至尊_大纲_优化版.md` — ~300章节，~7000行，~340KB
- 每章结构：`**第X章：标题**` → `**情节**` → `**爽点**` → `**钩子**` → `**写作提示**` → `---`
- `**第X-Y章：...**` 为汇总章节标题，后面跟着多个详细章节

## 《数学太凶猛》大纲文件格式（2026-04-22 确认）

- 章节头：`#### Ch{N}：{title}`（**全角冒号`：`**，不是半角`:`——`re.split`对此无效！）
- 分章正确方法：`re.finditer(r'^#### Ch(\d+)：', content, re.MULTILINE)` 获取 `(position, ch_num)` 列表，再用 `content[start:next_start]` 切片
- 字段格式：`**【字段名】**`（粗体markdown，不是普通`【】`）
- 伏笔埋设子字段：每行一个，如 `短线：（待补充）` / `中线：（待补充）` / `长线：（待补充）`
- 各卷结构差异：
  * 卷一：有【核心爽点】【情绪节奏】【冲突对手】【金手指应用】【伏笔埋设】【章末钩子】
  * 卷二~六：有【爽点位置】【情绪价值】（注意不是情绪节奏！）、【冲突对手】【伏笔埋设】【章末钩子】，无【核心爽点】无【金手指应用】
- 修补示例：`re.sub(r'\*\*【冲突对手】\*\*\n- （待补充）', '**【冲突对手】**\n- 生成内容', result)`

## 适用场景

- 大纲文件章节数≥50章时
- 需要在两个章节之间插入内容时
- `patch`工具报告"Found N matches"（N>1）时

---

## 伏笔/铺垫分散策略（重要约束）

**Tao明确偏好：禁止创建新章节文件（Ch50.5等半编号章节）**

当需要为某条故事线（如"京城苏家突然出现"）添加前文铺垫时，**必须**将内容分散到已有章节中，而不是创建新的章节文件。

### 分散策略示例：苏家身世线铺垫

目标：为Ch51~Ch53的"京城苏家出现"添加前文铺垫（原来完全无铺垫）

| 章节 | 时间 | 插入位置 | 内容 |
|------|------|---------|------|
| Ch49 | 5月（早于苏家2个月） | 章节开场 | 凌晨下班，陌生西装男人在街对面盯着她看 |
| Ch50 | 6月（苏家来之前） | 章节结尾 | 苏念向陈权倾诉不安，神秘京城车牌轿车的暗夜镜头 |
| Ch52 | 8月（苏家来时） | 章节开头 | 回忆承接"被盯视感越来越强"，自然过渡到迈巴赫到来 |
| Ch53 | 8月15日（苏家确认后） | 章节结尾 | 刘伯电话汇报：锦川少爷知道苏念存在，很不高兴 |

**效果**：从"突然出现"变为"逐渐逼近的阴影"，叙事节奏更自然。

### 删除重复章节的正确方式

当两章内容完全重复时（如Ch40/Ch45都是"2025年9月1日江城大学报名"）：

1. 比较两章内容丰富度，保留更丰富的那章
2. 删除时**不要清空内容**，用`mv`移至`.bak`备份：
   ```bash
   mv Ch40_她还在读大三.md Ch40_她还在读大三.md.bak
   ```
3. 不要试图合并内容（两章重复=删一章即可）

### 重写优于Patch的情况

当章节存在根本性时间线矛盾（如"2020年陈念2岁却上大学"），修复patch的文本量超过原章节60%时，**直接重写整个章节**更高效。

### 低字數章节处理

- 初稿章节字数<1000字的，通常意味着情节严重缩水，优先检查是否需要重写
- "766字"等早期统计可能是误报，patch前先用Python确认实际字数
