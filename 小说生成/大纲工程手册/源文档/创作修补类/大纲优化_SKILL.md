---
name: novel-outline-optimization
description: Iterative novel outline optimization using reader-reviewer skill + file patching, with progress reports every N rounds
category: creative
---

# Novel Outline Optimization Workflow

## When to Use

当用户要求优化小说大纲（通常5,000+字），且要求：
- 迭代优化（多轮"审核→修改"循环）
- 按用户要求定期汇报进度
- 不在对话框输出大纲（写入文件）

## 核心工作流（实测有效）

```
用户请求优化大纲
    ↓
[创建/使用 reader-reviewer skill → 读者视角审核]
    ↓
[优化第1轮：修复严重问题 → patch写入文件]
    ↓
[审核第2轮 → 报告结果]
    ↓
[优化第2轮：修复次要问题 → patch写入文件]
    ↓
每5轮简短汇报一次进展
    ↓
迭代直到用户满意或截止时间
```

## 关键约束

1. **大纲写入文件，不在对话框输出** — 用户明确要求每次迭代后"简短报告"，不是完整输出
2. **5轮一报** — 用户指定每5次迭代后报告进展
3. **用 patch 而非重写** — 大文件（60-100KB）用 patch 精准替换段落，避免重写丢失内容
4. **读者视角审核是质量门** — 每次优化后用 reader-reviewer skill 做质量检查

## 具体步骤

### Step 1：建立评估标准 + 四维复合审核（深度审核流程）

当用户要求"综合审核"或"深度审核"时（不限轮次，要求全面），使用以下流程：

#### A. 先读知识库（15-20分钟）

读4个知识库文件建立基准：
1. `~/ainovel/wiki/concepts/爽感四大基石.md`
2. `~/ainovel/wiki/concepts/节奏模组.md`
3. `~/ainovel/wiki/concepts/读者心理学.md`
4. `~/ainovel/wiki/concepts/伏笔设计体系.md`

同时读项目核心文件：
- D_分卷大纲.md / C_爽点节奏表.md / A_人物.md / B_伏笔线.md / E_时间线.md
- 精读V1-V2细纲 + 扫读V3-V5细纲

#### B. 四维复合审核（知识库框架）

**维度一：爽感密度** — 每3章1小爽点（标准3章）/ 每10章1大冲突 / 每卷≥5名场面 / 四大基石覆盖
**维度二：节奏与钩子** — 章末钩子质量 / 冲突间隔≤10章 / 持续压抑≤5章
**维度三：人设一致性** — 主角智商在线 / 感情线清爽 / 反派智商不断崖
**维度四：伏笔质量** — 回收≤150章 / 有仪式感 / 一事不烦二主

#### C. 问题分级

| 级别 | 定义 | 处理 |
|------|------|------|
| P0 | 影响留存/完读率，立即修复 | 立即patch |
| P1 | 影响精品率，30天内修复 | 计划patch |
| P2 | 锦上添花 | 记录待办 |

#### D. 审核报告结构
P0/P1/P2清单 + 四维评分(5分制) + 加权总分 + 市场定位 + 亮点 + 修复建议

### Step 2：分阶段 patch 优化

文件可能很大（60-100KB），不要试图一次重写。分批 patch：

**常见优化类型：**

| 优化目标 | 操作 |
|---------|------|
| 感情线详细化 | 找到"第X-Y章：概略版"段落 → 替换为详细章节 |
| 战斗系统模板 | 在适当位置插入"战斗描写模板"段落 |
| 章节补全 | 找到"第X-Y章"概略行 → 替换为每章一段的详细版 |
| 伏笔追踪表 | 在文件末尾追加伏笔追踪总表 |
| 删除重复内容 | 搜索并 patch 掉重复的旧段落 |

**patch 注意事项：**
- 每次 patch 后用 `execute_code` 验证文件大小和行数
- 大文件（>1500行）容易出现重复内容，用 `search_files` 定位后 patch 删除
- `read_file(limit=50)` 分段读取比一次读取更可靠

### Step 3：定期汇报

每5轮迭代后，向用户报告：
```
📊 第X-Y轮进度报告
- 文件大小变化
- 本轮新增内容
- 剩余优化项
- 下轮目标
```

## 常见问题处理

### Q：大纲文件出现两份相同章节怎么办？
A：用 `search_files` 找到两个章节标题的行号，patch 删除旧的那份

### Q：文件太大（>100KB），patch 找不到唯一字符串？
A：用 `read_file(offset=X, limit=50)` 精确定位，确保 old_string 唯一

### Q：多次patch后文件出现重复段落残留怎么办？
A：**直接重写整个文件**，不要继续patch清理。原因：多次追加型patch会导致同位置内容重复（如宿命线前世表在文件里出现两次），继续patch只会越来越乱。实战经验（E_时间线.md Round1→Round2）：
  - 原始文件多次追加后产生3个商业线+2个宿命线+残留段落
  - `read_file()` 读完确认全貌 → `write_file()` 整体重写 → 比继续patch更可靠
  - 原则：当 `Found N matches for old_string` 报错超过2次时，直接重写

### Q：模块化文件夹结构（多文件）如何保持一致性？
A：当核心设定改动（如"宿命因果链"升级）时，需要同步更新所有相关文件，否则会出现矛盾。同步优先级：
  1. **F_世界观背景.md** — 核心设定/主题（最优先，因为其他文件引用它）
  2. **D_分卷大纲.md** — 各卷核心事件描述（影响爽点节奏）
  3. **E_时间线.md** — 前世/今生时间线（容易遗漏，需重点检查）
  4. **G_人物关系图.md** — 反派命名/角色关系（终极Boss命名后需同步更新）
  5. **B_伏笔线.md** — 伏笔描述（细节描述可能需要更新）
  6. **C_爽点节奏表.md** — 爽点序号（结构问题，不涉及内容升级时可不动）
  验证：用 `execute_code` 一次性搜索关键词在所有文件中的命中情况

### Q：用户要求每N轮汇报，但迭代次数不够？
A：继续迭代直到达到汇报节点，不需要主动汇报

### Q：reader-reviewer skill 给出的问题和之前重复？
A：正常——第一轮修复了严重问题后，第二轮审核会聚焦次要问题，迭代是递进的

## 系统性大纲审查（QA检查法）

当用户要求"检查大纲"而非"优化大纲"时，使用以下流程，无需动用 reader-reviewer skill：

### 核心方法：用 execute_code + regex 做机器检查

```python
import re, os

files = {
    "第一卷": "细纲/第一卷细纲.md",
    "第二卷": "细纲/第二卷细纲.md",
    "第三卷": "细纲/第三卷细纲.md",
    "第四卷": "细纲/第四卷细纲.md",
    "第五卷": "细纲/第五卷细纲.md",
}

for vol, path in files.items():
    with open(path, encoding='utf-8') as f:
        content = f.read()

    # 1. 章节覆盖检查
    chapters = [int(m.group(1)) for m in re.finditer(r'### Ch(\d+)', content)]
    expected = set(range(chapters[0], chapters[-1]+1))
    missing = sorted(expected - set(chapters))
    dup = [c for c in chapters if chapters.count(c) > 1]

    # 2. 格式五要素检查
    has_scene = content.count('**场景**')
    has_drama = content.count('**剧情**')
    has_爽点 = content.count('**爽点')
    has_台词 = content.count('**经典台词**')
    has_钩子 = content.count('**章末钩子**')

    # 3. 章节内容充实度（字数<200字标记为较薄）
    chapter_bodies = re.findall(r'### Ch(\d{3})[:：](.*?)(?=\n---|\n### |\Z)', content, re.DOTALL)
    thin = [(ch, len(body)) for ch, body in chapter_bodies if len(body.strip()) < 200]

    # 4. 关键词出现位置（用于跨卷一致性检查）
    # 例如：闪拍应该在第四卷出现，不应在第三卷出现
    if '闪拍' in content:
        flash_positions = [m.start() for m in re.finditer(r'闪拍', content)]
        # 用章节标题位置表标注每个关键词的章节归属
        ch_headers = [(m.start(), m.group(1)) for m in re.finditer(r'### Ch(\d+)', content)]
        for pos in flash_positions[:5]:  # 只取前5个
            for i in range(len(ch_headers)-1):
                if ch_headers[i][0] <= pos < ch_headers[i+1][0]:
                    print(f"  闪拍@Ch{ch_headers[i][1]}")
                    break

    print(f"{vol}: 章节{chapters[0]}-{chapters[-1]}({len(chapters)}章) | 缺失:{missing} | 重复:{set(dup)} | 薄:{thin}")
```

### 关键检查项

| 检查类型 | 目标 | 异常信号 |
|---|---|---|
| 章节覆盖 | 250章无缺失 | 缺失章节=问题章节 |
| 重复章节 | 无重复 | 同一章节号出现>1次 |
| 格式五要素 | 每章5个要素齐全 | 某要素计数<<章节数 |
| 内容充实度 | 每章>200字 | <200字=需补充 |
| 跨卷关键词 | 同关键词应在正确卷 | 闪拍提前出现=卷分配错误 |
| 角色出现追踪 | 核心角色全程出现 | 中途消失=遗漏 |
| 时间线一致性 | 年份递增+无倒退 | 2015→2014=年份写错 |

### 实战发现（2026-04-25）

**跨卷内容冲突的精确定位法**：
- 闪拍线在第三卷大量出现（Ch136-Ch148），但按设定应是第四卷Ch151立项
- 用位置标记法：`闪拍出现位置: [10559, 10636, ...]` → 对应 `Ch136, Ch136, Ch137...`
- 发现后直接定位问题章节，无需通读全文

**室友线追踪法**：
- 小A未出现在第三卷（仅在1、2卷出现）→ 缺失重大情感锚点
- 小C在第三卷仅出现1次 → 室友戏份不足

**跨卷年份冲突**：
- Ch149写"2014年求婚"，但第四卷Ch181才写"2016年陆家嘴求婚"
- 附录与正文冲突 → 附录也需审查

### 审查报告格式

```
## 大纲审查报告
### ✅ 通过项
### 🔴 严重问题（含位置+修复建议）
### 🟡 次要问题
### 修复优先级表
```

---

## 验证清单

每次迭代后检查：
- [ ] 文件成功写入（`execute_code` 验证 `os.path.getsize`）
- [ ] 关键章节存在（`search_files` 搜索章节标题）
- [ ] 无重复内容残留
- [ ] 爽感密度符合标准（每3章1小爽点，每10章1大冲突）

## 4步优化流程（用户审核后才执行模式）

适用于：大规模大纲优化（P0/P1/P2多级问题），需要用户确认后再执行。

```
第一步：生成优化计划（P0+P1问题清单 + 对应方案）
    ↓
第二步：审核优化计划（主智能体审核 → 用户确认 → 接受/修改）
    ↓
第三步：执行优化（子代理并行patch写入）
    ↓
第四步：整体评审打分（复审评分报告，对比优化前后）
```

### 第一步：生成优化计划

生成《P0P1优化计划.md》，包含：
- P0问题清单（影响留存/完读率，立即修复）
- P1问题清单（影响精品率，30天内修复）
- 每个问题对应优化方案（章节位置 + 具体修改内容）
- 执行顺序（分批并行）

### 第二步：审核优化计划

主智能体审核计划，提出质疑：
- ✅ 确认合理的项
- ⚠️ 需要澄清的项（修改建议）
- ❌ 计划本身的问题（可能被高估的问题）

用户说"接受"后执行第三步。

### 第三步：执行优化

分批并行执行（子代理每批最多3个）：
- 第一批：纯文本替换（P0-1名字统一）+ 独立新增（P1-4系统存在感）
- 第二批：涉及原有章节内容修改（P0-2孔德昌冲突 + P0-3知识圣剑等）
- 第三批：V3-V5章节内容修改

执行后验证：
```bash
for f in ~/小说流/文道至尊/细纲/*7要素*.md; do echo "$(basename $f): $(wc -l < $f)行"; done
```

### 第四步：整体评审打分

生成《复审评分报告.md》，包含：
- 各维度评分对比（初版 vs 优化后）
- 加权总分（5维度×权重）
- 仍存在的扣分项（P2级）
- 终审结论 + 下一阶段建议

**实测效果（文道至尊案例）：**
- 初版审核：3.46/5（精品以下）
- 优化后：4.06/5（番茄精品门槛）
- 提升：+0.60分

**关键经验：**
- P0-2孔德昌打压原计划插入Ch40-Ch50，但审核发现该区间已有密集感情线 → 改为Ch38-Ch39
- P1-2天道崩塌节奏问题原计划"压缩" → 审核发现实际问题是"铺垫不足" → 改为"增强预兆"
- 用户审核环节能发现计划本身的问题，避免白执行

---

## 双版本合并模式（新增：v3整合增强版）

当存在**两个不同版本**的大纲时（如：原文大纲+优化版大纲），按以下流程合并：

```
两个版本文件路径
    ↓
Step 1: 对比分析（execute_code）
    - 统计行数/字数/章节数
    - 用正则提取所有章节编号
    - 找出缺失章节（set差集）
    - 找出各自独有内容（section差集）
    ↓
Step 2: 定位插入点（read_file分段读取）
    - 找到优化版的末尾（最后几行）
    - 确认章节结构
    ↓
Step 3: patch追加内容（按优先级）
    P0: 缺失章节详细内容（正文补全）
    P1: 附录精华（人物归宿/优化报告等）
    P2: 各自独有章节内容
    ↓
Step 4: 验证（execute_code）
    - 确认章节描述数=300
    - 确认附录全部存在
    ↓
输出: v3整合增强版
```

**实测数据（文道至尊案例）：**

| 指标 | 原文大纲 | 优化版v2 | 整合版v3 |
|------|---------|---------|---------|
| 文件大小 | 188.5KB | 115.9KB | 129.9KB |
| 章节描述 | 299章 | 273章 | **300章** |
| 缺失章节 | — | 27个 | **0个** |

**关键发现：**
- 优化版结构清晰但内容比原文少（删除了重复结构导致）
- 原文有27个章节优化版缺失，加上4个完整附录
- 最优策略：保留优化版结构，追加原文精华内容

## 子智能体并行扩展陷阱（实战经验）

当使用5个子智能体并行扩展各卷章节时，发现以下问题：

### 陷阱1：子智能体追加而非替换
- **现象**：子智能体将扩展内容追加到文件末尾，而非替换原有简略章节
- **后果**：文件末尾出现重复章节 + 原有简略章节未被替换
- **修复**：用 execute_code 清理合并
  ```python
  # 定位append起点（找章节重号的位置）
  for i, line in enumerate(lines):
      if '#### 第221-240章' in line and i > 3000:  # 重号章节标志
          append_start = i; break
  # 提取真正新增章节（set差集）
  new_chaps = sup_chaps - main_chaps
  # 删除append部分，只保留新章节
  new_content = [block for block in blocks if int(re.search(r'第(\d+)章', block).group(1)) in new_chaps]
  merged = original + '\n---\n\n' + '\n---\n\n'.join(new_content)
  ```

### 陷阱2：子智能体创建新文件
- **现象**：子智能体在同目录下创建 `文道至尊_章节提示补全.md` 而非修改主文件
- **后果**：新文件有59章补充内容，但主文件未更新
- **修复**：手动合并
  ```python
  # 提取新文件中的章节，替换/追加到主文件
  sup_chaps = split_chapters(supplement_file)
  for num in sorted(sup_chaps.keys(), reverse=True):  # 逆序避免位置偏移
      if num in main_chaps:
          merged = merged.replace(main_chaps[num], sup_chaps[num], 1)
      else:
          merged = merged + '\n---\n\n' + sup_chaps[num]
  ```

### Q：服务器限流(529错误)
- **现象**：子智能体并发过多时收到529错误
- **修复**：子智能体遇到错误会自动重试，但并发太高时需减少并行数

### Q：子智能体读错文件版本（章节编号系统冲突）
- **现象**：同一项目存在两套章节编号系统，子智能体按文件名判断范围，实际内容却对不上
- **案例**：文道至尊项目
  - CLAUDE.md规范：第1卷Ch1-60，第2卷Ch61-120，第3卷Ch121-180，第4卷Ch181-240，第5卷Ch241-300
  - 7要素扩写版实际：V1=Ch1-60，V2=Ch61-120，V3=Ch201-300，V4=Ch301-400，V5=Ch401-500
  - 子智能体按文件名认为"第三卷"=Ch121-Ch180，实际内容是Ch201-300
- **预防**：先用 `execute_code` 扫描所有细纲文件的实际章节范围，生成映射表，再发给子智能体
  ```python
  import re, os
  vol_files = {
      "第一卷": "细纲/第一卷细纲_7要素扩充版.md",
      "第二卷": "细纲/第二卷细纲_7要素扩充版.md",
      "第三卷": "细纲/第三卷细纲_7要素扩充版.md",
      "第四卷": "细纲/第四卷细纲_7要素扩充版.md",
      "第五卷": "细纲/第五卷细纲_7要素扩充版.md",
  }
  for vol, path in vol_files.items():
      with open(path) as f:
          content = f.read()
      chapters = sorted(set(int(m.group(1)) for m in re.finditer(r'Ch(\d{3})', content)))
      chaps_100 = sorted(set(int(m.group(1)) for m in re.finditer(r'Ch(\d{3})', content)))
      print(f"{vol}: Ch{min(chapters)}-{max(chapters)} ({len(chapters)}章)")
  ```

### Q：多卷并行优化时子智能体数量受限
- **现象**：`delegate_task` 默认 `max_concurrent_children=3`，一次提交5个任务会报错
- **修复**：分两批执行（3+2），间隔几秒即可
- **注意**：不要在同一次 `delegate_task` call里试图突破并发限制，分批更稳定

### 安全工作流（推荐）
```
1. 先用 execute_code 备份原文件
2. 子智能体扩展时每个指定不同输出文件（避免写同一文件冲突）
3. 完成后统一合并：用 set 差集找真正新增章节，逐章替换/追加
4. 合并后验证：统计 写作提示 数量 + 检查重复章节
```

## 五卷并行优化工作流（2026-04-19实测）

适用于：多卷长篇网文（5卷×50-100章），要求"先设计→审核→执行"三步骤。

### 完整流程

```
第一步（子代理并行）：阅读细纲 + 生成优化设计文档
    ↓
第二步（主智能体）：审核5份设计文档
    ↓
第三步（子代理并行）：按P0优先级执行patch
```

### 第一步：生成优化设计文档（5个子代理并行）

每个子代理任务包含：
1. 精读对应卷的7要素扩写版 + 基础细纲 + 知识库（爽感四大基石/节奏模组/读者心理学）
2. 生成《X卷_优化设计文档.md》，结构：
   - 现状诊断（爽感密度⭐1-5 / 名场面数量 / 7要素完整度）
   - 核心爽感定位（主导类型 + 情绪曲线）
   - 优化设计方案（按章节段）
   - 新增名场面设计（目标≥5个）
   - 爽感节奏核查表（过压/过爽/节奏OK）
   - 伏笔埋设优化（与邻卷联动）
   - 主线影响评估（改动点 + 风险等级）

**注意**：
- 并行数上限3，分两批（3+2）
- 每个子代理输出到独立文件：`优化设计文档/第X卷_优化设计文档.md`

### 第二步：审核优化文档

主智能体读取全部5份文档，检查：
- 章节范围是否与实际内容匹配（防编号系统冲突）
- 主线破坏风险（高风险改动标红，直接废弃）
- P0优先级是否合理

### 第三步：执行优化

按P0优先级，5个子代理并行patch到7要素扩写版：
- 每次patch后验证文件行数
- 遇到重复内容，用 `execute_code` + `write_file` 整体重写比继续patch更可靠

### 输出物

```
~/小说流/文道至尊/优化设计文档/
├── 第一卷_优化设计文档.md
├── 第二卷_优化设计文档.md
├── 第三卷_优化设计文档.md
├── 第四卷_优化设计文档.md
└── 第五卷_优化设计文档.md
```

---

## 文道至尊优化记录（参考案例）

- 原始文件：~193KB，章节集中在第一卷
- 优化版v2：~85KB，五卷全部详细展开
- 整合版v3（双版本合并）：~130KB，300章全描述+全部附录
- 并行扩展v2（2026-04-16）：314KB，298章100%完成，读者审核4.0/5
- 总迭代：20+轮，每5轮汇报一次
- 关键改进：感情线详细化/战斗描写模板/反派四层体系连贯化/势力扩张时间线/双版本合并/并行扩展+合并清理

## 扩展进度追踪（增量优化技巧）

当大纲整合完成后（如v3整合增强版），如果用户要求**继续扩展简化章节**：

```python
# Step 1: 追踪进度
import os
opt_path = '~/小说流/文道至尊_大纲_优化版.md'
orig_path = '~/.hermes/cache/documents/原文大纲路径.md'
opt_size = os.path.getsize(opt_path)
orig_size = os.path.getsize(orig_path)
print(f"当前: {opt_size/1024:.1f}KB / 原文: {orig_size/1024:.1f}KB | 完成度: {opt_size/orig_size*100:.1f}%")
```

**扩展优先级（实测高效顺序）：**
1. **附录简化章节** — 原文约5000字→详细版约20000字，一口气补全最大缺口
2. **每卷核心爽点章节** — 第1-10章黄金开篇、每卷高潮章、每卷收尾章
3. **正文最短章节** — 无写作提示+<10行的章节优先扩展

```python
# Step 2: 找最短章节（用于确定扩展优先级）
for ch, pos in sorted(chapter_positions.items()):
    non_empty = [l for l in content_lines if l.strip()]
    has_tips = any('写作提示' in l for l in content_lines)
    if len(non_empty) < 10 and not has_tips:
        shortest.append((ch, len(non_empty), content_lines[0][:50]))
```

**扩展格式标准（4要素）：**
```
**第X章：章节名**

**情节**：
- 要点1（含仪式感描写）
- 要点2

**【名场面】**：核心场景描写

- 爽点：XXX
- 钩子：XXX

**写作提示**：
- 写作要点1
- 写作要点2
```

**实测数据：**
- 整合版v3初始：130KB（vs原文188KB，完成度69%）
- 扩展附录E后：150KB（vs原文188KB，完成度81%）
- 扩展第1-9章后：153.5KB（vs原文188KB，完成度81.4%）

---

## 爽感模组批量升级工作流（2026-04-25实测）

当需要对多卷大纲进行爽感模组批量升级（200+章节）时，按以下流程：

### Step 1：先检查字段格式（不同卷格式可能不同）

```python
# 检查各卷的爽感字段名是否一致
for vol, filename in vol_files:
    with open(path) as f:
        content = f.read()
    fields = re.findall(r'\*\*【([^】]+)】\*\*', content)
    from collections import Counter
    field_counts = Counter(fields)
    print(f"{vol}: {dict(field_counts.most_common(5))}")
```

**实战发现**：同一项目的不同卷可能使用不同格式：
- 新格式卷（第一卷）：`【爽感模组】` + `【情绪节奏】`
- 旧格式卷（第二-六卷）：`【爽点位置】` + `【情绪价值】`（爽感模组在情绪价值字段的子行）

### Step 2：找到"常规爽感"目标文本

```python
# 在各卷中定位"常规爽感"
for vol, filename in vol_files:
    content = open(path).read()
    # 新格式：爽感模组：常规爽感
    # 旧格式：爽感模组：常规爽感（在情绪价值子行）
    generic = re.findall(r'爽感模组[：:]\s*(常规爽感|待定|[^\n]+)', content)
    print(f"{vol}: 常规爽感章节={len([g for g in generic if g.strip() in ['常规爽感','待定']])}")
```

### Step 3：基于上下文智能推导升级目标

```python
# 读取爽点位置+冲突对手+情绪价值，综合判断升级目标
for ch in weak_chapters:
    block = get_chapter_block(ch)
    suidian = extract_field(block, '爽点位置')
    conflict = extract_field(block, '冲突对手')
    emotion_pattern = extract_field(block, '情绪价值')
    
    # 基于关键词匹配推导
    if any(kw in suidian for kw in ['芯片', '禁令', '科技战']):
        upgrade = "国家荣耀+大国博弈"
    elif any(kw in conflict for kw in ['IMO', '竞赛', '集训']):
        upgrade = "国家荣耀+名场面蓄力"
    elif any(kw in emotion_pattern for kw in ['狂喜', '震撼']):
        upgrade = "极致史诗+名场面"
    elif any(kw in conflict for kw in ['学霸', '天才']):
        upgrade = "碾压天才+优越感"
    else:
        upgrade = "阶段性成就+名场面蓄力"
    
    new_block = block.replace('爽感模组：常规爽感', f'爽感模组：{upgrade}')
```

### Step 4：分卷执行+验证

```python
# 每卷单独处理，写入后立即验证
for vol, filename in vol_files:
    # ... 执行升级 ...
    with open(path, 'w') as f:
        f.write(content)
    
    # 验证：统计剩余常规爽感
    remaining = len(re.findall(r'爽感模组[：:]\s*(常规爽感|待定)', content))
    print(f"{vol}: 剩余常规爽感={remaining}")
```

### 关键经验

1. **格式先行**：升级前必须先确认目标卷的字段格式，否则替换失败
2. **智能推导**：根据爽点位置/冲突对手/情绪价值的实际内容推导升级目标，不用盲填
3. **多轮清理**：大卷（150章）往往需要2-3轮才能清完所有"常规爽感"
4. **全文替换 vs 逐块替换**：用 `content.replace(block, new_block, 1)` 比正则替换更可靠
5. **验证比不可少**：每次写入后立即统计剩余"常规爽感"数量
