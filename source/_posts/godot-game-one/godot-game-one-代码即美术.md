---
title: "Godot 4 做复古弹幕游戏：代码即美术"
date: 2025-06-03 15:30:00
tags:
  - Godot
  - 游戏开发
  - 复古风格
categories:
  - 游戏开发
top_img: false
---

<!-- more -->

做独立游戏最头疼的往往不是代码，是美术。`godot-game-one` 项目一开始也卡在这里——没有美术资源，没有美术师，只有代码。

最终的解法是：**所有视觉素材全部用 GDScript 程序化生成**。整个 `SpriteFactory.gd` 文件 898 行，承担了传统项目里原画师 + 精灵图集 + 动画师的全部职责。

{% mermaid %}
graph LR
    A["游戏启动"] --> B["SpriteFactory 初始化"]
    B --> C["生成玩家飞船"]
    B --> D["生成敌人 x3 类型"]
    B --> E["生成 Boss x5 变体"]
    B --> F["生成子弹/爆炸/陨石"]
    C --> G["缓存为 ImageTexture"]
    D --> G
    E --> G
    F --> G
    G --> H["运行时直接引用 零开销"]
{% endmermaid %}

## SpriteFactory：898 行代码画出整个游戏

它生成的素材清单：

- **玩家飞船**：逐像素绘制机身、机翼、驾驶舱、引擎喷口和尾焰动画，尺寸随等级增长
- **敌人**：3 种类型各一套像素图（红色追逐型、绿色锯齿型、紫色环绕型）
- **Boss**：5 种配色变体，每种独立生成
- **子弹**：玩家 4 级渐变色（蓝→青→紫→金），敌人统一红色
- **爆炸**：10 帧逐帧动画，全部代码生成
- **陨石**：程序化形状 + 随机裂纹纹理

核心原理是 Godot 的 `Image.set_pixel()` API。以玩家飞船为例：

```python
func create_player_sprite(level: int = 1) -> ImageTexture:
    var w: int = 64 + level * 8
    var h: int = 96 + level * 8
    var img := Image.create(w, h, false, Image.FORMAT_RGBA8)
    img.fill(Color(0, 0, 0, 0))  # 透明底
    var cx: int = int(w * 0.5)
    var cy: int = int(h * 0.5)

    # 机身颜色随等级变化
    var body_r: float = 0.12 + level * 0.04
    var body_g: float = 0.30 + level * 0.06
    var body_b: float = 0.75 - level * 0.02

    # 逐像素绘制：机身 → 机翼 → 驾驶舱 → 引擎喷口
    for y in range(20, 70):
        for x in range(cx - 12, cx + 12):
            img.set_pixel(x, y, Color(body_r, body_g, body_b))
```

Boss 的 5 种配色通过一个字典管理，切换变体只需改 key：

```python
var _boss_colors := {
    "boss_01": {"body": Color(0.05, 0.08, 0.25), "stripe": Color(0.8, 0.1, 0.1)},
    "boss_02": {"body": Color(0.15, 0.15, 0.15), "stripe": Color(0.9, 0.5, 0.1)},
    "boss_03": {"body": Color(0.05, 0.20, 0.22), "stripe": Color(0.2, 0.5, 0.9)},
    "boss_04": {"body": Color(0.15, 0.05, 0.20), "stripe": Color(0.9, 0.1, 0.6)},
    "boss_05": {"body": Color(0.12, 0.12, 0.12), "stripe": Color(0.9, 0.8, 0.2)},
}
```

所有素材在游戏启动时一次性生成，缓存为 `ImageTexture`，后续直接引用，运行时零开销。

## 复古感从哪里来？

不是靠滤镜，不是靠着色器。项目里**没有任何 .gdshader 文件**，复古感完全来自两个 `project.godot` 配置：

```ini
[rendering]
textures/canvas_textures/default_texture_filter=0    # NEAREST过滤
2d/antialiasing/quality=0                             # 无抗锯齿
```

`default_texture_filter=0` 关闭双线性插值，像素边缘锐利不模糊；`antialiasing/quality=0` 保留锯齿感。就这两行配置，配合像素风格的程序化素材，画面立刻有了 NES 那味儿。

## 为什么不直接画图？

三个原因：

1. **迭代快**：改配色改参数就行，不用重新导图
2. **体积小**：没有外部图片资源，APK 体积极小
3. **可扩展**：Boss 5 种变体只是换了一组颜色参数，加第 6 种就是改一个字典的事

## 代价与收获

程序化生成的像素图细节有限，画面质感肯定比不上专业美术。但收获也很明确：

1. **APK 体积极小**：没有任何图片资源文件打包进安装包
2. **迭代速度惊人**：改配色改参数就行，不用切换到图像编辑器重新导图
3. **可扩展性强**：加第 6 种 Boss 变体只需要在字典里加一行
4. **完全可控**：颜色、尺寸、形状全部参数化，支持运行时动态调整

项目里已经有一个外部图片加载的 fallback 路径——`_load_png()` 会先检查是否存在外部 PNG，有则直接加载，没有才走程序化生成。这意味着美术替换是渐进式的，不需要一次性换完。

> 技术选型的本质是取舍。代码即美术不是最优解，是当下最务实的解。
