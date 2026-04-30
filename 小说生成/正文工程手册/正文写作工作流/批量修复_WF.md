# 批量修复WF：多章patch/扩写处理

> 本工作流用于多章同时修复，适用于：章节完成后的整体AI词清理、多章字数调整、批量扩写。

---

## 一、批量修复触发条件

| 条件 | 说明 |
|------|------|
| 单章AI词超标严重（>10处） | 单章patch效率低，适合批量处理 |
| 多章需要统一清理 | 如"然后"在多章中都超标 |
| 批量扩写 | 多章字数都不够 |
| 整体格式修复 | 如章节分隔符统一、标题格式统一 |

---

## 二、批量修复标准流程

```
Step 1：扫描 → 列出所有问题
Step 2：分类 → 按问题类型分组
Step 3：排序 → 优先级排序
Step 4：执行 → 按优先级逐类修复
Step 5：验证 → 每类修复后验证
Step 6：交付 → 全部通过后交付
```

---

## 三、批量AI词扫描脚本

```python
import os

folder = '/path/to/项目根目录/正文/第X卷/'
files = sorted([f for f in os.listdir(folder) if f.startswith('Ch') and f.endswith('.md')])

ai_words = ['而是','仿佛','微微','然后','最后','然而','但是','因此','所以',
            '不过','似乎','如同','宛如','首先','次','再次','因而','因为']

for filename in files:
    path = os.path.join(folder, filename)
    with open(path, 'r') as f:
        text = f.read()
    issues = []
    for w in ai_words:
        c = text.count(w)
        if c:
            issues.append(f"{w}:{c}")
    if issues:
        print(f"{filename}: {', '.join(issues)}")
    else:
        print(f"{filename}: ✅")
```

---

## 四、批量patch修复

### 4.1 单类词批量修复

```python
import os

folder = '/path/to/项目根目录/正文/第X卷/'

# 批量修复"然后"
replacement_map = {
    '然后站起身': '站起身',
    '然后转身': '转身就走',
    '然后坐下': '他一屁股坐下',
    '然后把': '把',
}

files = ['Ch01_新的开始.md', 'Ch02_新的挑战.md', 'Ch03_新的起点.md']

for filename in files:
    path = os.path.join(folder, filename)
    with open(path, 'r') as f:
        text = f.read()
    original_len = len(text)
    for old, new in replacement_map.items():
        text = text.replace(old, new)
    if original_len != len(text):
        with open(path, 'w') as f:
            f.write(text)
        print(f"✅ {filename}: {original_len} → {len(text)}")
```

### 4.2 多类词批量修复

```python
# 一次性修复多种AI词
fixes = [
    ('然后', ''),
    ('微微', '轻轻'),
    ('仿佛', ''),
    ('然而', '只是'),
    ('但是', '只是'),
]

for filename in files:
    path = os.path.join(folder, filename)
    with open(path, 'r') as f:
        text = f.read()
    for old, new in fixes:
        text = text.replace(old, new)
    with open(path, 'w') as f:
        f.write(text)
    print(f"✅ {filename} 修复完成")
```

---

## 五、批量扩写

### 5.1 通用扩写模板

```python
# 多章同时扩写
files = ['Ch01_新的开始.md', 'Ch02_新的挑战.md']

for filename in files:
    path = os.path.join(folder, filename)
    with open(path, 'r') as f:
        text = f.read()
    
    original_len = len(text.replace('\n', ''))
    
    # 扩写1：开场场景加感官描写
    old = '他出发那天，天气很好。\n\n阳光从云缝里洒下来。'
    new = '''他出发那天，天气很好。

阳光从云缝里洒下来，照得官道两旁的树梢亮晶晶的。路边的野花开得正盛，黄的、紫的、白的，一丛一丛地点缀在田埂上。'''
    text = text.replace(old, new)

    # 扩写2：增加心理活动
    old = '她比上次瘦了。'
    new = '''她比上次瘦了。他记得第一次见她的时候，她的脸还是圆圆的，笑起来有两个浅浅的酒窝。现在她的脸颊尖了，眼睛显得更大。他心里一阵疼。'''
    text = text.replace(old, new)
    
    with open(path, 'w') as f:
        f.write(text)
    new_len = len(text.replace('\n', ''))
    print(f"{filename}: {original_len} → {new_len} (+{new_len-original_len})")
```

---

## 六、批量验证

### 6.1 快速AI词扫描

```python
import os

folder = '/path/to/项目根目录/正文/第X卷/'
ai_words = ['而是','仿佛','微微','然后','最后','然而','但是','因此','所以',
            '不过','似乎','如同','宛如','首先','次','再次','因而','因为']

print(f"{'文件名':<30} {'字数':<8} {'问题'}")
print("-" * 70)

for filename in sorted(os.listdir(folder)):
    if not filename.startswith('Ch') or not filename.endswith('.md'):
        continue
    path = os.path.join(folder, filename)
    with open(path, 'r') as f:
        text = f.read()
    total = len(text.replace('\n', ''))
    issues = []
    for w in ai_words:
        c = text.count(w)
        if c:
            if c > 2:
                issues.append(f"{w}×{c}❌")
            elif c == 2:
                issues.append(f"{w}×{c}⚠️")
            else:
                issues.append(f"{w}×{c}")
    issue_str = ', '.join(issues) if issues else '✅'
    status = "❌" if any('❌' in i for i in issues) else "⚠️" if any('⚠️' in i for i in issues) else "✅"
    print(f"{filename:<30} {total:<8} {issue_str}")
```

### 6.2 交付验证

```python
# 批量交付前最终验证
def verify_chapter(filename):
    path = os.path.join(folder, filename)
    with open(path, 'r') as f:
        text = f.read()
    total = len(text.replace('\n', ''))
    zero_words = ['而是', '因而']
    threshold_words = ['仿佛','微微','然后','最后','然而','但是','因此','所以',
                       '不过','似乎','如同','宛如','首先','次','再次','因为']
    passed = True
    for w in zero_words:
        if text.count(w) > 0:
            passed = False
            print(f"  ❌ {w} 超出阈值")
    for w in threshold_words:
        c = text.count(w)
        if c > 2:
            passed = False
            print(f"  ❌ {w}: {c} > 2")
    if not (2600 <= total <= 4500):
        passed = False
        print(f"  ❌ 字数{total}不在范围内")
    return passed

files = ['Ch01_新的开始.md', 'Ch02_新的挑战.md', 'Ch03_新的起点.md']
all_passed = True
for f in files:
    ok = verify_chapter(f)
    print(f"{'✅' if ok else '❌'} {f}")
    all_passed = all_passed and ok

print(f"\n{'🎉 全部通过' if all_passed else '❌ 有章节未通过'}")
```

---

## 七、批量修复检查清单

```
□ 所有问题已扫描列出
□ 按优先级排序修复
□ 修复后立即验证
□ 字数全部在2600-4500之间
□ AI禁词全部达标
□ 无新增问题
□ 全部章节可交付
```
