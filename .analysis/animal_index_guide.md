# Animal Index Guide — 动物索引操作指南

## 概述

`animal_index.json` 收录 `picx-images-hosting` 中含真实动物的图片，按动物种类（中英双语）归类，值为 `folder/filename.ext` 格式的相对路径列表。

本索引只收真实动物。玩偶、雕塑、绘画、模型、装饰图案、食物里的动物制品不收录。

---

## 已扫描文件夹（截至 2026-04-30）

| 文件夹 | 扫描状态 | 动物结果 |
|--------|----------|----------|
| `2025summer/` | ✅ 完成 | 无真实动物 |
| `cat/` | ✅ 完成 | 纯白猫、橘色虎斑猫 |
| `dust/` | ✅ 完成 | 无真实动物 |
| `example/` | ✅ 完成 | 无真实动物（插画/物品排除） |
| `fuji/` | ✅ 完成 | 无真实动物（海洋馆剪影太小/不明确，玩偶排除） |
| `hokkaido/` | ✅ 完成 | 棕熊、王企鹅 |
| `item/` | ✅ 完成 | 无真实动物（照片/玩偶/模型排除） |
| `kobe/` | ✅ 完成 | 无真实动物 |
| `others/` | ✅ 完成 | 蜜蜂、乌鸫、红嘴蓝鹊 |
| `sakura/` | ✅ 完成 | 疣鼻天鹅、梅花鹿 |
| `summer-pockets/` | ✅ 完成 | 海鸥 |
| `yakushima/` | ✅ 完成 | 屋久鹿、日本猕猴 |

---

## 扫描方法

### 路径变量

在相册仓库根目录执行时，可使用：

```bash
GALLERY_ROOT=.
THUMB_ROOT=<thumbnail-worktree>
WORK_DIR=.analysis
```

缩略图分支可用：

```bash
git worktree add <thumbnail-worktree> origin/thumbnail
```

### 缩略图联系表 / sprite sheet

先把每个文件夹生成一张带编号和文件名的联系表，再只放大疑似动物图。这样比逐张读取更省 token，也更容易排除连续的非动物照片。

```bash
python3 - <<'PY'
from PIL import Image, ImageDraw, ImageFont
import os, math

base = os.environ.get('THUMB_ROOT', '<thumbnail-worktree>')
out = os.path.join(os.environ.get('WORK_DIR', '.analysis'), 'contact_sheets')
os.makedirs(out, exist_ok=True)
exts = ('.webp', '.jpg', '.jpeg', '.png', '.JPG', '.JPEG', '.PNG')

try:
    font = ImageFont.truetype(os.environ['FONT_PATH'], 15)
except Exception:
    font = ImageFont.load_default()

folders = [d for d in sorted(os.listdir(base)) if os.path.isdir(os.path.join(base, d)) and not d.startswith('.')]
for folder in folders:
    dirp = os.path.join(base, folder)
    files = sorted(f for f in os.listdir(dirp) if f.endswith(exts))
    if not files:
        continue

    tw, th, lh, cols = 180, 135, 40, 5
    rows = math.ceil(len(files) / cols) or 1
    sheet = Image.new('RGB', (cols * tw, rows * (th + lh)), 'white')
    draw = ImageDraw.Draw(sheet)

    for i, name in enumerate(files):
        im = Image.open(os.path.join(dirp, name)).convert('RGB')
        im.thumbnail((tw, th), Image.Resampling.LANCZOS)
        x = (i % cols) * tw + (tw - im.width) // 2
        y = (i // cols) * (th + lh)
        sheet.paste(im, (x, y))
        draw.text(((i % cols) * tw + 4, y + th + 2), f'{i + 1}. {name}'[:35], fill=(0, 0, 0), font=font)

    sheet.save(os.path.join(out, folder + '_contact.jpg'), quality=90)
PY
```

### 判读规则

- 只记录真实动物；照片框中的照片、玩偶、卡通、雕塑、装饰和食物不收录。
- 小到无法确认的动物不收录，除非单独打开原图后能确认。
- 同一张图有多个真实动物种类时，可让路径出现在多个物种 key 下。
- 写入 `animal_index.json` 时使用主分支真实存在的原图路径；缩略图分支有但主分支没有的文件不要写入。

### 合并后校验

```bash
python3 -m json.tool "$WORK_DIR/animal_index.json" >/dev/null
```

检查索引路径是否存在：

```bash
python3 - <<'PY'
import json, os
base = os.environ.get('GALLERY_ROOT', '.')
index = json.load(open(os.path.join(base, '.analysis/animal_index.json'), encoding='utf-8'))
missing = []
for key, paths in index.items():
    if key == '_meta':
        continue
    missing.extend(p for p in paths if not os.path.exists(os.path.join(base, p)))
print('missing', len(missing))
for path in missing:
    print(path)
PY
```

---

## 暂存产物

`contact_sheets/` 是缩略图联系表目录，可保留用于复核；`animal_index.json` 是主动物索引。
