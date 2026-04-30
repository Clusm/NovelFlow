---
name: novel-chapter-post-completion-qa
description: 对已完成的正文章节文件进行质量修复——AI格式化修复、年龄体系审计、全书批量清理
triggers:
  - 修复章节文件
  - 正文章节质量
  - AI痕迹清理
  - 章节格式化
  - 单行章节
---

# 网文正文文件质量修复流程

## 何时使用
当需要对已完成的正文文件（而非大纲）进行质量修复时使用。包括：
- AI生成文件格式化（单行/段落粘连）
- 全书年龄体系一致性审计
- AI写作痕迹清理（词汇/句式）
- 批量事实错误修复

## 前置条件
- 正文章节文件存在于 `正文/` 目录下（按卷分目录）
- 已有监督者事实台账（fact_check_batch*.md）

---

## 第一步：元数据扫描（execute_code）

```python
import os, re

base = "/path/to/正文/"

# 1. 扫描单行格式化（<5行=有问题）
for vol_name in os.listdir(base):
    vol_path = os.path.join(base, vol_name)
    for fname in os.listdir(vol_path):
        path = os.path.join(vol_path, fname)
        with open(path, encoding='utf-8') as f:
            lines = f.read().split('\n')
        if len(lines) < 5:
            print(f"⚠️ {fname}: {len(lines)}行（单行格式化）")

# 2. 扫描AI痕迹词汇
ai_patterns = [r'轻轻地', r'缓缓地', r'渐渐地', r'慢慢地', r'仿佛', r'宛如', r'好像', r'嘴角微微']
for vol_name in os.listdir(base):
    vol_path = os.path.join(base, vol_name)
    for fname in os.listdir(vol_path):
        path = os.path.join(vol_path, fname)
        with open(path, encoding='utf-8') as f:
            content = f.read()
        for p in ai_patterns:
            if re.search(p, content):
                print(f"⚠️ {fname}: 含AI痕迹")

# 3. 扫描全文完标记
for vol_name in os.listdir(base):
    vol_path = os.path.join(base, vol_name)
    for fname in os.listdir(vol_path):
        path = os.path.join(vol_path, fname)
        with open(path, encoding='utf-8') as f:
            content = f.read()
        if '全文完' in content:
            print(f"⚠️ {fname}: 含全文完标记")
```

---

## 第二步：年龄体系审计

**已知年龄锚点**（从大纲确认）：
- 陈念出生于 2018年6月1日
- 陈安（陈阳）出生于 2020年
- 苏念出生于 1993年

**快速验证**：
```python
import re
for vol_name in os.listdir(base):
    vol_path = os.path.join(base, vol_name)
    for fname in os.listdir(vol_path):
        path = os.path.join(vol_path, fname)
        with open(path, encoding='utf-8') as f:
            content = f.read()
        age_hits = re.findall(r'(\d{4}年).*?(\d{1,2})岁', content)
        for date, age in age_hits:
            year = int(date.replace('年',''))
            expected = year - 2018  # 陈念基准
            actual = int(age)
            if abs(actual - expected) > 2:
                print(f"⚠️ {fname} ({date}): {actual}岁，应为~{expected}岁")
```

---

## 第三步：批量格式化修复

**format_chapter() 函数**（核心）：
```python
def format_chapter(path):
    with open(path, encoding='utf-8') as f:
        content = f.read()
    
    # AI词汇替换
    for old, new in [(r'轻轻地',''), (r'缓缓地',''),(r'渐渐地',''),
                      (r'慢慢地',''),(r'仿佛','像'),(r'宛如','像')]:
        content = re.sub(old, new, content)
    
    # 处理literal \\n
    content = content.replace('\\n', ' ')
    
    # 按句号分句重建段落
    sentences = re.split(r'([。！？])', content)
    result, current_para, current_len = [], [], 0
    i = 0
    while i < len(sentences):
        s = sentences[i].strip()
        if not s:
            i += 1
            continue
        if i + 1 < len(sentences) and sentences[i+1] in '。！？':
            s += sentences[i+1]
            i += 1
        if '---' in s or '**' in s or s.startswith('#'):
            if current_para:
                result.append(''.join(current_para))
                current_para = []
            result.append(s)
            current_len = 0
        elif current_len > 200 and len(current_para) > 2:
            result.append(''.join(current_para))
            current_para = [s]
            current_len = len(s)
        else:
            current_para.append(s)
            current_len += len(s)
        i += 1
    if current_para:
        result.append(''.join(current_para))
    
    lines = [re.sub(r' +', ' ', r).strip() for r in result if r.strip()]
    return '\n\n'.join(lines)
```

**决策规则**：
- 文件<3KB或损坏严重 → `write_file`重写
- 文件>3KB但单行 → `format_chapter()`批量处理
- 内容正确仅个别错误 → `patch`精确替换

---

## 第四步：关键章节必须人工读

以下情况必须 `read_file` + 人工判断：
1. 章节有多个时间跳跃
2. 章节涉及多个角色互动（容易格式化破坏场景边界）
3. 年龄矛盾超过5岁以上

---

## 已知坑点

1. **Python写入破坏格式**：`terminal + f.write()` 时 `\n` 变成字面文本 `\\n`。解：用 `write_file` 工具重写。

2. **全文完标记判断**：
   - `【全文完】` 非大结局章节 → 删除
   - `【第X卷 完】` 卷末 → 合法
   - 最后一卷末 `【全文完】` → 合法

3. **Ch32类严重损坏**：整章1-2行+年龄矛盾 → 必须重写

4. **format_chapter局限性**：对话密集章节分段偏碎，需人工检查

---

## 输出文档

1. **事实台账**：`fact_check_batch*.md`
2. **挑剔读者报告**：`挑剔读者Block*报告.md`
3. **最终汇总**：更新到 `C_人物.md` 年龄体系表
