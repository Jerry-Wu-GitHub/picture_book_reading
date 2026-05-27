# 任务书-漫画网格

## 任务描述

用 Python 编写一个函数。

- 输入：一张图片文件（`bytes`），是两页漫画。
    - 文件格式：JPG/PNG
    - 图片大小：不超过 1600×1200 像素。
- 输出：一个列表 `list[list[int]]` ，列表中的每个元素表示在漫画上识别到的一个方框的坐标 `[x1, y1, x2, y2]` 。其中，`(x1, y1)` 是方框的左上角坐标，`(x2, y2)` 是方框的右下角坐标。
    - 坐标原点位于图片的左上角，往右为 x 轴正方向，往下为 y 轴正方向。
    - 单位长度：图片在该维度上的长度 / 1000 。即，图片的右下角的坐标为 `(1000, 1000)` 。

## 函数签名

你实现的函数的签名应类似：

```python
def identify_grid(image_bytes: bytes) -> List[List[int]]
```

或更准确地：

```python
def identify_grid(image_bytes: bytes) -> List[Tuple[int, int, int, int]]
```

或它们的异步形式：

```python
async def identify_grid(image_bytes: bytes) -> List[List[int]]
async def identify_grid(image_bytes: bytes) -> List[Tuple[int, int, int, int]]
```

## 参数

- `image_bytes` (bytes)：要处理的照片文件（字节），如：

    ![01](./comic_grid.assets/03.jpg)

你可以把 [comic_grid.assets](comic_grid.assets) 里的图片作测试用。

## 返回值

示例：

```python
[
    [40, 61, 259, 275],
    [271, 60, 486, 275],
    ...
]
```

- 输出的方框顺序不重要。

你最好写个 demo ，调用你写的 `identify_grid` 函数，把输出的矩形框位置标在图上，方便可视化检查效果。

## 调用示例

```python
import os
os.mkdir("output")


# 假设这些图片文件在 input 文件夹里
image_names = [
    "01.jpg",
    "02.jpg",
    ...,
    "06.jpg"
]

for image_name in image_names:
    with open(f"input/{image_name}", mode="rb") as file:
        image_bytes = file.read()

    grids = identify_grid(image_bytes)
    print(grids)
    
    # 假设你实现了类似的标注方框的函数
    annotated_image_bytes = annotate(image_bytes, grids)

    with open(f"output/{image_name}", mode="wb") as file:
        file.write(annotated_image_bytes)

```

## 要求

- Python 版本：3.12
- 允许使用第三方库（如 OpenCV、Pillow、numpy 等）
- 不能本地部署大模型
- 性能：在单核 CPU 上：单次调用的最大运行时间 $\leq 0.5$ 秒。


暂时先不考虑运行时内存吧。

## 提交方式

自己创建一个 GitHub 仓库，README 里写清楚运行耗时、效果图、调用方式等。