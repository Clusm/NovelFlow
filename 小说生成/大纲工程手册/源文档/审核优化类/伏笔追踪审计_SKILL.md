---
name: novel-outline-foreshadowing-audit
description: 系统性审查网文大纲伏笔设计与回收——扫描五卷细纲，逐章验证伏笔链，修复断裂点，更新追踪文档。适用于万行级大纲的伏笔审查与修复。
---

# Novel Outline Foreshadowing Audit

Verify all foreshadowing threads in a multi-volume web novel outline are properly set up and resolved.

## Trigger Conditions
- User asks to "检查伏笔" / "审查伏笔" / "伏笔回收" / "检查逻辑"
- After completing a major outline revision (new chapters, character arcs, plot reversals)
- Before starting main text writing

## Workflow

### Phase 1: Read Foreshadowing Tracking Doc
```python
with open("B_伏笔线.md", encoding='utf-8') as f:
    foreshadowing_doc = f.read()
```

### Phase 2: Scan All Volumes for Keywords
```python
import re

volumes = {}
for vol, fname in [
    ("第一卷", "第一卷细纲.md"),
    ("第二卷", "第二卷细纲.md"),
    ("第三卷", "第三卷细纲.md"),
    ("第四卷", "第四卷细纲.md"),
    ("第五卷", "第五卷细纲.md"),
]:
    with open(f"细纲/{fname}", encoding='utf-8') as f:
        volumes[vol] = f.read()

all_content = "\n".join(volumes.values())

# Keywords to search: 伏笔/悬念/预示/暗示/宿命/前世/重生/母亲/身世/豪门/血脉/车祸/旧伤/左臂
```

### Phase 3: Verify Each Foreshadowing Thread
```python
def get_ch_drama(content, ch_num):
    for m in re.finditer(r'### Ch(\d+)[:：]', content):
        if m.group(1) == ch_num:
            ch_start = m.end()
            ch_end = content.find("---", ch_start)
            ch_text = content[ch_start:ch_end]
            drama_m = re.search(r'\*\*剧情\*\*[：:]\s*(.+)', ch_text)
            return drama_m.group(1)[:70] if drama_m else ""
    return ""
```

### Phase 4: Identify 4 Common Problem Types

| Problem Type | Detection | Fix Location |
|-------------|-----------|--------------|
| **Wrong chapter numbers in tracking doc** | Search shows different chapter | Update tracking doc |
| **Missing cause chain** | Foreshadowing but no resolution | Add resolution in later chapter |
| **Missing emotional payoff** | Resolution but no setup | Add foreshadowing in earlier chapter |
| **Continuity gap** | Character disappears mid-arc | Add bridging chapter |

### Phase 5: Execute Patches
```python
# BEFORE patching: always read exact content first
# Use repr() to see exact whitespace/characters

for m in re.finditer(r'### Ch33[:：]', content):
    print(repr(content[m.end():content.find("---", m.end())]))

# Then patch with full surrounding context
```

### Phase 6: Verify Each Fix
```python
checks = [
    ("Ch33 左臂旧伤", vol1_content, "左臂"),
    ("Ch119 室友焦急", vol3_content, "小B当场"),
    ("Ch137 母亲身份", vol3_content, "林雪琴"),
    ("Ch181 旧伤呼应", vol4_content, "左臂"),
]
for name, content, keyword in checks:
    print(f"{'✅' if keyword in content else '❌'} {name}")
```

## Known Pitfalls

1. **Chapter numbers in tracking doc are often wrong**: The doc may say Ch11 but it's actually Ch10. Always verify actual chapter numbers.
2. **Sub-chapters (Ch45a/Ch45b)**: Pattern `f"### Ch{num}[:：]"` matches both `Ch45：` and `Ch45:`.
3. **Empty chapters**: Some chapters have `### ChXX：` immediately followed by `---`. Skip these.
4. **Patch fails with "not found"**: Chapter formatting differs. Use `repr()` before patching.
5. **Adjacent chapters merge**: If `### Ch119：...### Ch120：...` without separator, original had no content.

## Verification Checklist
- [ ] All "已回收" foreshadowing threads verified in actual chapter content
- [ ] All "待补充" issues have patches applied to correct chapters
- [ ] Tracking doc version number updated
- [ ] Line counts of patched files verified (wc -l before and after)
