---
name: novel-outline-deduplication
description: 网文大纲文件去重技能——检测并修复大纲文件中重复章节、内容错位等问题。问题根源：多次追加内容时没有在正确位置插入，而是append到文件末尾，导致同一章节号出现2-3个副本（尤其是200章以后的章节）。使用本skill的检测和修复流程。
triggers:
  - "章节重复"
  - "大纲损坏"
  - "内容错位"
  - "追加后重复"
---

# 网文大纲文件去重技能

## 问题识别

**典型症状**：
- grep能找到某章节号出现2-3次
- 文件行数异常膨胀（如同文件应有长度的2-3倍）
- 200章以后每个章节都有多个副本
- 缺失章节号（如62、241）

**问题根源**：
用 `patch` 或 `append` 追加新章节时，没有在正确位置插入，而是追加到了文件末尾。多次追加后，同一章节号产生多个副本。

## 检测流程

```bash
# 1. 统计所有章节号，找出缺失和重复
grep -oP '^\*\*第\K\d+(?=章：)' ~/小说流/文道至尊_大纲_优化版.md | sort -n | uniq -c | sort -k2 -n

# 2. 找出所有章节的详细行号（注意：格式是 **第X章：标题**）
grep -n '^\*\*第' ~/小说流/文道至尊_大纲_优化版.md

# 3. 用Python解析重复（最可靠）
python3 << 'EOF'
import re
with open('~/小说流/文道至尊_大纲_优化版.md', 'r') as f:
    lines = f.read().split('\n')

chapter_positions = []
for i, line in enumerate(lines):
    m = re.match(r'\*\*第(\d+)章：(.+?)\*\*', line.strip())
    if m:
        chapter_positions.append((i+1, int(m.group(1)), m.group(2)))

by_chapter = defaultdict(list)
for line_num, ch_num, title in chapter_positions:
    by_chapter[ch_num].append(line_num)

duplicates = {ch: entries for ch, entries in by_chapter.items() if len(entries) > 1}
for ch_num in sorted(duplicates.keys()):
    print(f"第{ch_num}章: {duplicates[ch_num]}")
EOF
```

## 修复策略

### 策略A：保留最新版本（推荐）✅ 实测有效

**重要发现**：实践表明，行号较大的版本（后追加的）内容更完整。行号较小的版本（先有的）往往是简略版。

**正确算法**：
1. 对每个重复章节号，按行号升序排序
2. 保留行号最大的版本（最新/最完整）
3. 删除其他所有旧副本的**整段内容**（从该章节标题行到下一章标题行前）

```python
# 核心逻辑（Python）
import re
from collections import defaultdict

with open('大纲.md', 'r', encoding='utf-8') as f:
    lines = f.read().split('\n')

# 找所有章节位置
chapter_positions = []
for i, line in enumerate(lines):
    m = re.match(r'\*\*第(\d+)章：(.+?)\*\*', line.strip())
    if m:
        chapter_positions.append((i, int(m.group(1)), m.group(2)))

# 按章节号分组
by_chapter = defaultdict(list)
for line_num, ch_num, title in chapter_positions:
    by_chapter[ch_num].append(line_num)

# 找出需删除的行区间
lines_to_delete = set()
all_chapter_lines = sorted(set(pos for pos_list in by_chapter.values() for pos in pos_list))

for ch_num, pos_list in by_chapter.items():
    if len(pos_list) > 1:
        pos_sorted = sorted(pos_list)
        keep_pos = pos_sorted[-1]  # 保留行号最大的
        for del_pos in pos_sorted[:-1]:
            # 找到下一个章节的位置
            next_pos = None
            for i, pos in enumerate(all_chapter_lines):
                if pos == del_pos and i + 1 < len(all_chapter_lines):
                    next_pos = all_chapter_lines[i + 1]
                    break
            if next_pos is None:
                next_pos = len(lines)
            # 标记该区间所有行
            for i in range(del_pos, next_pos):
                lines_to_delete.add(i)

# 构建并写入新内容
new_lines = [line for i, line in enumerate(lines) if i not in lines_to_delete]
with open('大纲.md', 'w', encoding='utf-8') as f:
    f.write('\n'.join(new_lines))
```

**⚠️ 关键教训**：
- 初始方案错误：只删除章节标题行 → 内容残留
- 正确方案：删除从该章节到下一章节的**整段内容**
- 初始方案错误：保留行号最小的 → 删除了完整版保留了简略版
- 正确方案：保留行号最大的（最新追加的内容更完整）

### 策略D：章节块提取 + 内容行数比较（2026-04-16实测最佳）

当文件结构严重损坏（章节号顺序 ≠ 行号顺序）时，完整重建比逐个修补更快：

```python
import re
from collections import defaultdict

with open('大纲.md', 'r', encoding='utf-8') as f:
    lines = f.read().split('\n')

# Phase 1: 提取所有章节块
chapter_blocks = {}
for i, line in enumerate(lines):
    m = re.match(r'\*\*第(\d+)章：', line.strip())
    if m:
        ch_num = int(m.group(1))
        # 该章节块 = 从当前位置到下一章节位置
        end_pos = None
        for j in range(i + 1, len(lines)):
            if re.match(r'\*\*第\d+章：', lines[j].strip()):
                end_pos = j
                break
        if end_pos is None:
            end_pos = len(lines)
        
        block_lines = lines[i:end_pos]
        valid_lines = sum(1 for l in block_lines if l.strip() and l.strip() != '---')
        
        # 保留内容最丰富的版本
        if ch_num not in chapter_blocks or valid_lines > chapter_blocks[ch_num]['lines']:
            chapter_blocks[ch_num] = {
                'lines': valid_lines,
                'content': '\n'.join(block_lines)
            }

# Phase 2: 重建文件（按章节号顺序）
final_lines = []
for ch_num in range(1, 301):
    if ch_num in chapter_blocks:
        final_lines.append(chapter_blocks[ch_num]['content'].strip())
    else:
        # 缺失章节用占位符
        final_lines.append(f"**第{ch_num}章：章节待补充**")
        ...

with open('大纲.md', 'w', encoding='utf-8') as f:
    f.write('\n'.join(final_lines))
```

**关键发现**：
- 重复章节的多个实例内容行数不同，保留行数最多的版本
- 空壳章节（0行内容）说明大纲被截断或追加失败
- 章节块提取后直接重建，比逐个修补更快

### 策略B：手动对比保留更好版本

对于重要章节（剧情高潮/感情线关键点），手动对比两个版本，保留更好的。

### 策略C：完全重写

如果重复太严重（如同文件被追加了2-3次），考虑：
1. 导出所有章节到一个列表
2. 去重后重新生成文件
3. 补充缺失章节

## 预防措施

**追加章节的正确方式**：
```python
# 错误：直接append
with open(file, 'a') as f:
    f.write(new_chapter_content)

# 正确：找到插入位置
with open(file, 'r') as f:
    lines = f.readlines()
# 找到前一章节的位置，在其后插入
insert_index = find_chapter_position(lines, chapter_num - 1)
lines.insert(insert_index + 1, new_chapter_content)
with open(file, 'w') as f:
    f.writelines(lines)
```

**更好的方案**：使用章节管理系统，维护一个章节列表，按顺序插入。

## 检测命令速查

```bash
# 快速检测重复
grep -c '^\*\*第' file.md | grep -v ':1$'

# 统计每个章节号出现次数（注意格式：**第X章：标题**）
grep -oP '^\*\*第\d+章：' file.md | sort | uniq -c | sort -rn | head -20

# 找出章节号范围
grep -oP '^\*\*第(\d+)章：' file.md | grep -oP '\d+' | sort -n | head -1
grep -oP '^\*\*第(\d+)章：' file.md | grep -oP '\d+' | sort -n | tail -1

# 检查缺失章节
python3 << 'EOF'
import re
with open('file.md', 'r') as f:
    chapters = sorted(set(int(m.group(1)) for line in f if (m := re.match(r'\*\*第(\d+)章：', line.strip()))))
all_ch = set(range(min(chapters), max(chapters)+1))
missing = sorted(all_ch - set(chapters))
print(f"缺失: {missing}")
EOF
```

## 正文多文件章节去重法（跨目录章节文件版）✅ 2026-04-18实测

当章节文件分布在多卷目录（`正文/第一卷/`、`正文/第二卷/`等）时，同一章节号可能以不同文件名出现在同一目录：

**典型症状**：
- `Ch74_苏念身世揭露.md`（1142字）+ `Ch74_苏念的大学.md` + `Ch74_星火游戏出海.md` 同时存在
- `Ch75_京城苏家.md` + `Ch75_沈家的觊觎.md` 同时存在
- `Ch74` 出现3个版本，其他章节各有2个版本

**检测命令**：
```bash
# 在目标卷目录中统计各章节号文件数量
cd "正文/第四卷/"
for ch in 71 72 73 74 75 76 77 78 79 80; do
  count=$(ls Ch${ch}*.md 2>/dev/null | wc -l)
  if [ "$count" -gt 1 ]; then echo "Ch${ch}: ${count}个文件"; fi
done

# 快速获取所有Ch##文件的字数（前3行标题行）
for f in Ch7*.md; do echo "$f: $(wc -m < $f)字"; done
```

**去重决策流程**：
1. 对每个重复章节号，读取各版本的第1-3行（标题+日期）
2. **选版本标准**（按优先级）：
   - 内容最丰富的（字数最多）
   - 时间线与上下文衔接最顺的（检查章节末尾的日期/事件）
   - 主题与大纲要求最匹配的
3. 将被淘汰版本移动到备份目录（`.bak4`后缀）
4. 确认最终版本存在

**实测关键教训**：
- ⚠️ `ls -lt`（最新修改时间）不代表内容最完整——测试中最新修改的文件反而是简短的摘要版
- ✅ **正确方法：逐文件读取前几行，对比内容深度**
- ✅ 备份文件用`.bak4`后缀区分（`.bak`是去重前的旧备份，`.bak4`是本次去重的备份）

**Python检测脚本**：
```python
import os
from collections import defaultdict

vol_path = "/path/to/正文/第四卷/"
chapter_versions = defaultdict(list)

for f in os.listdir(vol_path):
    if f.startswith("Ch") and f.endswith(".md"):
        ch_num = f.split("_")[0].replace("Ch","")
        path = os.path.join(vol_path, f)
        size = os.path.getsize(path)
        with open(path, encoding='utf-8') as fh:
            lines = fh.readlines()
        title = lines[0].strip() if lines else "(空)"
        date = lines[1].strip() if len(lines)>1 else ""
        chapter_versions[ch_num].append({
            'file': f, 'size': size,
            'title': title, 'date': date
        })

# 打印所有重复
for ch, versions in sorted(chapter_versions.items(), key=lambda x: int(x[0])):
    if len(versions) > 1:
        print(f"\nCh{ch} 有 {len(versions)} 个版本:")
        for v in sorted(versions, key=lambda x: -x['size']):
            print(f"  {v['size']:>6}字 | {v['file']}")
            print(f"         | {v['date']}")
```

**执行流程（实测）**：
```bash
# Step 1: 备份所有文件
mkdir -p 备份_20260418/正文_backup/
cp 正文/第四卷/Ch7*.md 备份_20260418/正文_backup/

# Step 2: 确定每章保留哪个
# Ch74: 保留Ch74_苏念身世揭露.md（内容最丰富：手游出海+身世揭露）
mv Ch74_苏念身世揭露.md.bak ../Ch74_保留.md  # 保留的临时名
# ... 依次处理 ...

# Step 3: 移动被淘汰版本到备份
mv Ch74_苏念的大学.md 备份_20260418/正文_backup/
mv Ch74_星火游戏出海.md 备份_20260418/正文_backup/

# Step 4: 重命名保留版本为标准名
mv Ch74_保留.md Ch74_手游出海与身世线.md

# Step 5: 验证
ls *.md | wc -l  # 确认最终文件数量正确
```

---

## 常见逻辑矛盾修复（去重后必检）

去重后文件容易暴露以下矛盾，需逐一修复：

| 问题 | 涉及章节 | 修复方案 |
|------|----------|----------|
| **角色觉醒时间线重复** | 如赵铁柱觉醒在12/20/46章都出现 | 三章各有时序定位：发现潜力→正式收徒→实力飞跃 |
| **人名/名字前后不一** | 如院长名字在不同章节不同 | 统一为第一次出现的名字，全文替换 |
| **反派打脸节奏太慢** | 如孔德昌第64章才败露 | 重排节奏：登场→刁难(3章内)→当众打脸→彻底败露 |

**非整数章节号**：如12.5章、18.5章
- 修复：将"第X.5章"改为"第X章（下）"
- 命令：`grep -n '第.*5章' file.md` 查找

## 并行子代理审查/补全流程

### 场景1：15个缺失章节分3批并行

```
tasks = [
    {"goal": "补全第242/245/249/252/254章", "context": "混沌海探索+混沌主宰对决..."},
    {"goal": "补全第256/259/263/266/268章", "context": "混沌海净化+文道传播..."},
    {"goal": "补全第269/272/275/277/279章", "context": "结局篇+成为超存在..."},
]
```

### 场景2：5卷内容审查（每卷1个子代理）

```
tasks = [
    {"goal": "审查第一卷（第1-60章）...", "vol": "第一卷"},
    {"goal": "审查第二卷（第61-120章）...", "vol": "第二卷"},
    {"goal": "审查第三卷（第121-180章）...", "vol": "第三卷"},
    {"goal": "审查第四卷（第181-240章）...", "vol": "第四卷"},
    {"goal": "审查第五卷（第241-300章）...", "vol": "第五卷"},
]
# 由于max_concurrent_children=3，需分两批执行
```

## 补全缺失章节的插入技巧

**问题**：在有重复的大纲中插入缺失章节，行号会因重复副本而错位。

**正确做法**：
1. 先用grep找出所有章节行号：`grep -n '^\*\*第' file.md`
2. 找到目标章节（如第62章）的**任意一个**位置
3. 计算插入位置：在该章节内容结束后、下一章之前
4. **从后往前插入**（行号大的先插，小的后插），避免行号偏移

```python
# 示例：插入4个缺失章节
insertions = [
    (241, 5220, chapter_241_content),  # 行号大的先
    (226, 4210, chapter_226_content),
    (131, 5182, chapter_131_content),
    (62, 1540, chapter_62_content),   # 行号小的后
]
for ch_num, insert_pos, content in sorted(insertions, key=lambda x: -x[1]):
    lines.insert(insert_pos, content)  # 从后往前插
```

## 验证命令速查

```bash
# 1. 检查总章节数和唯一章节数
python3 -c "import re; c=re.findall(r'\*\*第(\d+)章：', open('file.md').read()); print(f'总:{len(c)} 唯一:{len(set(c))}')"

# 2. 检查缺失章节
python3 -c "import re; c=set(int(x) for x in re.findall(r'\*\*第(\d+)章：', open('file.md').read())); print(f'缺失:{[x for x in range(1,301) if x not in c]}')"

# 3. 检查非整数章节
grep -n '第.*5章：' file.md | grep -v '第[0-9]*[15]章'  # 排除15/25/35...章

# 4. 检查空壳章节（<3行内容）
python3 << 'EOF'
import re
lines = open('file.md').read().split('\n')
chapter_positions = [(i, int(re.match(r'\*\*第(\d+)章：', line).group(1))) 
                     for i, line in enumerate(lines) if re.match(r'\*\*第\d+章：', line)]
for i, (pos, ch) in enumerate(chapter_positions):
    next_pos = chapter_positions[i+1][0] if i+1 < len(chapter_positions) else len(lines)
    valid = sum(1 for j in range(pos+1, next_pos) if lines[j].strip() and lines[j].strip() != '---')
    if valid < 3:
        print(f"第{ch}章: 空壳({valid}行)")
EOF
```
