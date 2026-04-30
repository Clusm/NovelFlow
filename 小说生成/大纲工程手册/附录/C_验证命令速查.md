# 附录C：验证命令速查

## 章节完整性检查

```bash
# 检查总章节数和唯一章节数
python3 -c "import re; c=re.findall(r'\*\*第(\d+)章：', open('file.md').read()); print(f'总:{len(c)} 唯一:{len(set(c))}')"

# 检查缺失章节
python3 -c "import re; c=set(int(x) for x in re.findall(r'\*\*第(\d+)章：', open('file.md').read())); print(f'缺失:{[x for x in range(1,301) if x not in c]}')"

# 检查非整数章节（.5章节）
grep -n '第.*5章：' file.md | grep -v '第[0-9]*[15]章'  # 排除15/25/35章

# 检查空壳章节（内容行少于3行）
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

---

## 重复章节检测

```bash
# 快速检测重复
grep -c '^\*\*第' file.md | grep -v ':1$'

# 统计每个章节号出现次数
grep -oP '^\*\*第\d+章：' file.md | sort | uniq -c | sort -rn | head -20

# 找出章节号范围
grep -oP '^\*\*第(\d+)章：' file.md | grep -oP '\d+' | sort -n | head -1
grep -oP '^\*\*第(\d+)章：' file.md | grep -oP '\d+' | sort -n | tail -1

# 检查重复章节号
grep "^#### Ch" 大纲.md | awk '{print $2}' | sort | uniq -c | awk '$1>1{print}'
```

---

## 格式检查

```bash
# 检查缺少钩子的章节
python3 << 'EOF'
import re
with open('file.md') as f:
    content = f.read()
chapters = re.findall(r'### 第(\d+)章[:：](.*?)(?=\n---|\\n### |\Z)', content, re.DOTALL)
missing_hook = [(ch, len(c)) for ch, c in chapters if '钩子' not in c]
if missing_hook:
    print(f"缺失钩子: {[(ch, wc) for ch, wc in missing_hook]}")
EOF

# 检查7要素字段缺失
python3 << 'EOF'
import re
required = ['情节', '爽点', '钩子']
with open('file.md') as f:
    content = f.read()
for field in required:
    count = content.count(f'**{field}**')
    print(f"{field}: {count}处")
EOF
```

---

## 文件状态检查

```bash
# 统计文件行数
wc -l 文件.md

# 统计各卷细纲行数
for f in ~/项目/细纲/*.md; do echo "$(basename $f): $(wc -l < $f)行"; done

# 检查文件大小
ls -lh 文件.md

# 检查0字节文件
find . -name "*.md" -size 0
```

---

## 伏笔追踪检查

```bash
# 检查伏笔关键词出现位置
python3 << 'EOF'
import re
keywords = ["伏笔", "悬念", "第一次", "那天", "宿命", "前世"]
for vol, fname in [("第一卷","细纲/第一卷.md"), ("第二卷","细纲/第二卷.md")]:
    content = open(fname).read()
    for m in re.finditer(r'### Ch(\d+)[:：]', content):
        ch_start = m.end()
        ch_end = content.find("---", ch_start)
        ch_text = content[ch_start:ch_end]
        hits = [kw for kw in keywords if kw in ch_text]
        if hits:
            hook_m = re.search(r'\*\*章末钩子\*\*[：:](.+?)(?:\n|$)', ch_text)
            print(f"Ch{m.group(1)}: {hits} | 钩子: {hook_m.group(1)[:50] if hook_m else ''}")
EOF

# 验证伏笔回收表章节号
python3 << 'EOF'
import re
with open('B_伏笔线.md') as f:
    content = f.read()
foreshadowings = re.findall(r'\|\s*F\d+\s*\|.*?\|\s*Ch(\d+)\s*\|.*?\|\s*Ch(\d+)\s*\|', content)
for bury, recover in foreshadowings:
    print(f"埋: Ch{bury} -> 收: Ch{recover}")
EOF
```

---

## 时间线检查

```bash
# 检查年份分布
python3 << 'EOF'
import re
with open('file.md') as f:
    content = f.read()
years = re.findall(r'20\d\d年', content)
print(f"年份分布: {sorted(set(years))}")
year_positions = [(m.start(), m.group()) for m in re.finditer(r'20\d\d年', content)]
print(f"共{len(year_positions)}个年份标注")
EOF

# 跨卷年份一致性检查
python3 << 'EOF'
import re
files = {
    "第三卷": "细纲/第三卷.md",
    "第四卷": "细纲/第四卷.md",
}
for vol, path in files.items():
    with open(path) as f:
        content = f.read()
    years = re.findall(r'20\d\d年', content)
    print(f"{vol}年份: {sorted(set(years))}")
EOF
```

---

## 爽点密度检查

```bash
# 统计爽点类型分布
python3 << 'EOF'
import re
from collections import Counter
with open('file.md') as f:
    content = f.read()
types = re.findall(r'爽点[：:]\s*([^\/\n]+)', content)
counter = Counter(types)
for t, c in counter.most_common(10):
    print(f"{t}: {c}")
EOF
```

---

## 小数章节号清理

```bash
# 找所有小数章节号
grep -n "Ch[0-9]*\.[0-9]" 文件.md

# 全局搜索小数章节号
grep -rn "Ch[0-9]*\.[0-9]" --include="*.md" .

# 搜索格式：第305.5章
grep -rn "第.*\.[0-9]章" --include="*.md" .
```

---

## 批量文件操作

```bash
# 批量重命名（处理章节号迁移）
for f in Ch30*.md; do
  new=$(echo $f | sed 's/Ch30/Ch31/')
  mv "$f" "$new"
done

# 批量统计文件行数
find ~/项目 -name "*.md" -exec wc -l {} \; | sort -n

# 批量搜索关键词
find ~/项目 -name "*.md" -exec grep -l "关键词" {} \;
```
