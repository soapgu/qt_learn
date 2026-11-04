# Qt Quick Shapes：从第一条线到座位调整辅助线

在会议系统的座位图里，选中一个人的座位并开始调整后，鼠标移到空座位上，需要提示“移动到这里”；移到已经有人的座位上，则需要提示“交换这两个位置”。

最直接的表达，是在两个座位之间画出带方向的辅助线：移动用一条单向虚线，交换用两条分别向两侧弯曲、方向相反的虚线，箭头保持实线。

这篇文章从 Qt Quick Shapes 的基础讲起，再拆解项目 `red-conf-system` 的 `src/qml/views/seat/SeatMapView.qml` 中的实现。重点是理解路径如何描述形状，以及项目如何计算端点、控制点和箭头。

本文以 **Qt 6.8.3** 为基准，项目代码核对日期为 **2026-11-02**。文中标明“最小教学示例”的代码可以独立运行；“项目代码节选”依赖原项目的状态和对象，不能单独运行。

## 1. Shapes 负责什么

Qt Quick Shapes 提供声明式的路径绘制能力，可以描述直线、曲线、轮廓和填充。使用时导入：

```qml
import QtQuick
import QtQuick.Shapes
```

先区分三个层次：

```text
Shape                         场景中的可视项，负责显示路径
├── ShapePath                  路径及其描边、填充样式
│   ├── PathLine               画一段直线
│   └── PathCubic              画一段三次贝塞尔曲线
└── ShapePath                  另一条采用独立样式的路径
    ├── PathLine
    └── PathLine
```

`Shape` 继承 `Item`，可以设置位置、尺寸、层级和透明度。`ShapePath` 继承 `Path`，不是可布局的 `Item`，它描述“从哪里开始、经过哪些路径元素、如何描边和填充”。单独声明一个没有路径内容的 `Shape`，不会出现可见图形。参考：[Shape](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shape.html)、[ShapePath](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shapepath.html)。

### 1.1 常见绘制方式怎么选

| 方式 | 适合的内容 | 放到本例中考虑 |
| --- | --- | --- |
| `Rectangle` | 矩形背景、边框、圆角块 | 座位方块继续使用它；任意方向曲线需要其他方式 |
| `Canvas` | 使用 JavaScript 绘图上下文组织绘制步骤 | 可以实现连线，但需要自行管理重绘时机 |
| `Image` 加 SVG 资源 | 图标、预先设计好的固定图形 | 适合菜单图标；两个端点随悬停变化的连线需要动态几何 |
| `Shape` | 能通过属性描述的路径、曲线和轮廓 | 端点与控制点直接绑定座位状态，适合这组辅助线 |

采用 Shapes 的主要理由，是它的属性能够参与 QML 绑定，不必在每次鼠标变化时手动组织一遍绘图命令。

这不意味着 Shapes 在所有情况下都更快。Qt 的通用几何渲染方式需要处理路径几何，频繁改变路径也有成本；不同渲染器的处理方式存在差异。本文围绕一组辅助线展开，不据此推断大规模动态路径的性能。参考：[Shape 的渲染与性能说明](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shape.html#details)。

## 2. 最小教学示例：先画一条直线

把下面的代码保存为 `Main.qml`，使用 Qt 6.8.3 的 `qml Main.qml` 运行：

```qml
import QtQuick
import QtQuick.Window
import QtQuick.Shapes

Window {
    width: 420
    height: 240
    visible: true
    title: "Shapes 直线示例"
    color: "#F3F5F9"

    Shape {
        anchors.fill: parent

        ShapePath {
            strokeColor: "#356AE6"
            strokeWidth: 2
            fillColor: "transparent"
            capStyle: ShapePath.FlatCap
            startX: 60
            startY: 120

            PathLine {
                x: 360
                y: 120
            }
        }
    }
}
```

图中会出现一条从 `(60, 120)` 到 `(360, 120)` 的蓝色水平线。

这里有两组不同的属性：

- `startX`、`startY` 与 `PathLine.x/y` 描述几何位置。
- `strokeColor`、`strokeWidth`、`fillColor`、`capStyle` 描述外观。

`anchors.fill: parent` 让 `Shape` 占据窗口内容区域；本例路径坐标因此可以直接按窗口内的位置理解。它不会自动把任意路径拉伸成窗口大小，也不会自动把路径限制在该矩形内。

QML 常规坐标系的原点位于左上角，x 向右增大，y 向下增大：

```text
(0, 0) ───────────────────────────→ x
  │
  │      (60, 120) ──────── (360, 120)
  ↓
  y
```

### 2.1 为什么写透明填充

“描边”与“填充”是独立的。我们只需要路径轮廓，因此设置 `fillColor: "transparent"`。一条直线不一定能让你察觉填充，但换成有面积的折线或曲线后，忘记关闭填充就可能出现额外色块。

本文示例没有设置填充渐变或填充项；如果设置了它们，还要考虑它们对 `fillColor` 的覆盖。参考：[fillColor](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shapepath.html#fillColor-prop)。

### 2.2 路径元素如何衔接

路径有一个“当前位置”。第一个元素从 `startX/startY` 开始，后续元素从前一个元素的终点开始。

| 元素 | 做什么 | 需要重点理解的参数 |
| --- | --- | --- |
| `PathLine` | 从当前位置画直线到终点 | `x`、`y` |
| `PathMove` | 把当前位置移到另一处，不画连接线 | `x`、`y` |
| `PathQuad` | 画二次贝塞尔曲线 | 一个控制点与终点 |
| `PathCubic` | 画三次贝塞尔曲线 | 两个控制点与终点 |
| `PathArc` | 画椭圆弧 | 终点、半径、方向等 |

例如，把最小示例中的 `PathLine` 替换为：

```qml
PathLine { x: 180; y: 120 }
PathMove { x: 240; y: 120 }
PathLine { x: 360; y: 120 }
```

会得到两段分开的直线，中间从 180 到 240 的部分没有连线。`PathMove` 不会创建一套新样式；同一个 `ShapePath` 内的路径元素仍共用描边设置。

参考：[PathLine](https://doc.qt.io/qt-6.8/qml-qtquick-pathline.html)、[PathMove](https://doc.qt.io/qt-6.8/qml-qtquick-pathmove.html)、[PathQuad](https://doc.qt.io/qt-6.8/qml-qtquick-pathquad.html)、[PathArc](https://doc.qt.io/qt-6.8/qml-qtquick-patharc.html)。

## 3. 从实线到虚线，再到箭头

### 3.1 dashPattern 的数值不是直接写像素

在最小示例的 `ShapePath` 中增加：

```qml
strokeStyle: ShapePath.DashLine
dashPattern: [2, 1.5]
```

这表示反复绘制“2 倍线宽的线段、1.5 倍线宽的间隔”。线宽为 2 时，名义线段长为 4，间隔长为 3，单位是当前绘制坐标中的长度。

`dashPattern` 使用正数、偶数项，按“线段、间隔、线段、间隔……”解释。项目使用 `[2, 1.5]`，而不是把屏幕期望的 4 和 3 直接填进去。参考：[dashPattern](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shapepath.html#dashPattern-prop)。

### 3.2 线帽影响端点外观

| 设置 | 外观 |
| --- | --- |
| `ShapePath.FlatCap` | 端点处平切，不向外延伸 |
| `ShapePath.SquareCap` | 方形端点，向外延伸半个线宽 |
| `ShapePath.RoundCap` | 圆形端点 |

项目在线身上用 `FlatCap`，使虚线段边缘利落；箭头用 `RoundCap`，使两翼末端柔和。路径拐角的连接方式由另一项 `joinStyle` 控制，不能把线帽和拐角混为一谈。参考：[capStyle](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shapepath.html#capStyle-prop)。

### 3.3 箭头是另外两段实线

Shapes 不会因为路径有方向就自动显示箭头。这里的箭头是“翼点 A → 尖端 → 翼点 B”的两段折线。

在最小示例的 `Shape` 中，保留虚线线身，再添加第二个 `ShapePath`：

```qml
ShapePath {
    strokeColor: "#356AE6"
    strokeWidth: 2
    strokeStyle: ShapePath.SolidLine
    fillColor: "transparent"
    capStyle: ShapePath.RoundCap
    startX: 352
    startY: 116
    PathLine { x: 360; y: 120 }
    PathLine { x: 352; y: 124 }
}
```

这样箭头尖端恰好位于线身终点 `(360, 120)`。

如果用 `PathMove` 把箭头接到同一个虚线 `ShapePath` 中，它仍然受同一套虚线样式影响，可能断成几小段。**需要独立样式时，拆分 `ShapePath`**。项目最终使用四条路径，原因也在这里。

## 4. 回到项目：一组辅助线如何组成

座位调整的交互有两种：

| 鼠标目标 | 含义 | 连线表达 |
| --- | --- | --- |
| 空座位 | 将源座位的人移动过去 | 源到目标的单向虚线直线，加目标端实线箭头 |
| 已占用座位 | 两个座位的人交换 | 两条向相反两侧弯曲的虚线，各自在终点加实线箭头 |

实现上，这两种情况共用一个 `Shape`：

```text
adjustmentLink（Shape）
├── 路径 1：源 → 目标的虚线线身，始终参与有效连线
├── 路径 2：目标 → 源的虚线线身，仅交换时显示
├── 路径 3：目标端实线箭头
└── 路径 4：源端实线箭头，仅交换时显示
```

“仅交换时显示”在源码中是把对应路径的 `strokeColor` 改为透明，并没有为每次悬停创建或销毁对象。整个 `Shape` 还受状态与几何有效性的统一控制。

### 4.1 连线与座位共享坐标系

下面是省略了无关属性的**项目结构示意，不能独立运行**：

```qml
Flickable {
    id: viewport
    clip: true

    Item {
        id: canvas
        scale: root.displayScale
        transformOrigin: Item.TopLeft

        Repeater {
            id: seats
            model: root.viewModel.seats
            // delegate 在 canvas 中生成座位 Item。
        }

        Shape {
            id: adjustmentLink
            anchors.fill: parent
            z: 2
            opacity: 0.45
            // 几何计算与四条路径见后文。
        }
    }
}
```

`Repeater` 生成的座位项与 `adjustmentLink` 位于同一个 `canvas` 中，因此座位的 `x/y` 和路径使用同一套逻辑坐标。它们随 `canvas.scale` 一起缩放、随视口一起移动。

这里不需要把座位坐标先乘缩放系数，再给路径使用；那样会把缩放计算两次。如果以后把连线挪到另一个覆盖层，就需要通过 `mapToItem()` 等方式转换坐标，不能继续直接使用原来的 `x/y`。

### 4.2 悬停状态如何变成一条线

项目座位委托里的悬停处理如下：

```qml
HoverHandler {
    onHoveredChanged: {
        if (hovered)
            root.hoveredSeat = seat
        else if (root.hoveredSeat === seat)
            root.hoveredSeat = null
    }
}
```

离开时先检查当前引用是否仍指向自己，避免把其他座位刚设置的悬停对象清空。

连线随后用属性绑定读取目标，并从 `Repeater` 中找到源座位：

```qml
readonly property Item targetSeat: root.hoveredSeat
readonly property Item sourceSeat: {
    if (!targetSeat || !root.viewModel.selectingAdjustmentTarget)
        return null
    for (let i = 0; i < seats.count; ++i) {
        const candidate = seats.itemAt(i)
        if (candidate && candidate.model.seatId === root.viewModel.adjustmentSourceSeatId)
            return candidate
    }
    return null
}
readonly property bool swapping: targetSeat !== null && targetSeat.model.occupied
```

`swapping` 只回答目标是否有人；是否正在选择调整目标，由后面的 `visible` 条件判断。

整个数据流是：

```text
HoverHandler 更新 hoveredSeat
    → targetSeat / sourceSeat / swapping
    → geometry 与控制点
    → ShapePath 起点、终点、样式
    → Shapes 更新显示
```

`readonly property` 仍然能通过绑定重新计算；它限制的是外部赋值。这里没有 `requestPaint()`，几何变化直接通过绑定传递给路径。

## 5. 第一步几何计算：找到座位方块的中心

座位委托并非只有一个正方形。上面是编号文字，下面才是表示座位的正方形色块：

```text
座位 Item，位置 (x, y)，宽 w、高 h
┌────────────────────┐
│      编号文字       │ 高 h - w
├────────────────────┤ ← 方块顶部 y + h - w
│                    │
│         ●          │ 正方形，边长 w
│                    │
└────────────────────┘ ← 底部 y + h
```

对应源码中，文字高度为 `parent.height - parent.width`，方块的 `height: width`，并且 `anchors.bottom: parent.bottom`。

所以方块中心应为：

```javascript
const sx = sourceSeat.x + sourceSeat.width / 2
const sy = sourceSeat.y + sourceSeat.height - sourceSeat.width / 2
const tx = targetSeat.x + targetSeat.width / 2
const ty = targetSeat.y + targetSeat.height - targetSeat.width / 2
```

横向取半个宽度；纵向从底部向上走半个方块边长。不能简单写成 `y + height / 2`，那是整个委托的中心，连线会向编号区域偏移。

例如委托 `y = 20`、`height = 30`、`width = 20`，方块从 y=30 延伸到 y=50，中心是 y=40；整个委托中心则是 y=35。

## 6. 第二步几何计算：把端点从中心移到边缘

如果从中心直接连到中心，路径会穿过座位色块。项目先计算中心连线方向，再沿该方向截去两个方块内部的部分。

### 6.1 方向向量和单位向量

```javascript
const dx = tx - sx, dy = ty - sy
const distance = Math.hypot(dx, dy)
const ux = dx / distance, uy = dy / distance
```

`Math.hypot(dx, dy)` 等价于 `sqrt(dx * dx + dy * dy)`。`(ux, uy)` 是长度为 1 的方向向量：沿它走 k 个单位，就是移动 `(ux * k, uy * k)`。

这些语句之前要判断 `distance <= 0`，防止两个中心重合时除以零。源码也提前处理了源或目标缺失、源目标为同一对象的情况。

### 6.2 edgeFactor 为什么取两个绝对值的最大值

设正方形半边长为 r。从中心沿单位方向走 t，横向位移为 `t * ux`，纵向位移为 `t * uy`。到达边界时，至少一个位移的绝对值等于 r：

```text
max(|t × ux|, |t × uy|) = r
t × max(|ux|, |uy|) = r
t = r / max(|ux|, |uy|)
```

于是源码写成：

```javascript
const edgeFactor = Math.max(Math.abs(ux), Math.abs(uy))
const sourceInset = sourceSeat.width / 2 / edgeFactor
const targetInset = targetSeat.width / 2 / edgeFactor
```

不能对任意方向都直接减去半个宽度。斜着走到正方形角点的路程，比水平走到边中点更长。

假设两个方块的边长都为 20：

| 方向 | 单位向量 | `edgeFactor` | 中心到边缘的路程 |
| --- | --- | --- | --- |
| 向右 | `(1, 0)` | 1 | 10 |
| 向下 | `(0, 1)` | 1 | 10 |
| 右下 45° | `(0.7071, 0.7071)` | 约 0.7071 | 约 14.1421 |

### 6.3 得到最终的起终点

源端沿正方向前进，目标端沿反方向后退：

```javascript
return {
    start: Qt.point(sx + ux * sourceInset, sy + uy * sourceInset),
    end: Qt.point(tx - ux * targetInset, ty - uy * targetInset),
    length: Math.max(0, distance - sourceInset - targetInset),
    ux: ux,
    uy: uy
}
```

以方块边长 20 为例：

- 水平：中心从 `(0, 0)` 到 `(100, 0)`，截取后从 `(10, 0)` 到 `(90, 0)`，可用长度为 80。
- 垂直：中心从 `(0, 0)` 到 `(0, 100)`，截取后从 `(0, 10)` 到 `(0, 90)`，可用长度同样为 80。
- 斜向：中心从 `(0, 0)` 到 `(100, 100)`，截取后从 `(10, 10)` 到 `(90, 90)`，可用长度约为 113.1371。

这里的 `geometry.length` 是**截取后两个端点之间的直线距离**，不是贝塞尔曲线的弧长。方块相接或重叠而导致可用长度不大于零时，项目通过 `visible` 隐藏辅助线。

这一算法针对轴对齐的正方形座位块，求的是中心连线与方块边界的交点。后面即使改画曲线，也继续沿用这些端点；它没有额外计算任意曲线与方块的精确交点，也不负责绕开中间其他座位。

## 7. 第三步：用控制点把直线变成交换曲线

### 7.1 先理解 PathCubic 的四个点

三次贝塞尔曲线由四个点决定：起点 P0、控制点 P1、控制点 P2、终点 P3。在 `PathCubic` 中，起点来自路径当前位置，因此只需要显式填写两个控制点和终点。

把第 2 节最小示例的 `PathLine` 替换成下面这一段，就能看到一条向下弯的曲线：

```qml
PathCubic {
    control1X: 160
    control1Y: 180
    control2X: 260
    control2Y: 180
    x: 360
    y: 120
}
```

起点仍是 `(60, 120)`。两个控制点向下移动，把曲线向下拉；曲线通常不会穿过控制点。

```text
P0 ●                                      ● P3
     ╲                                  ╱
       ╲__________ 曲线 _______________╱

              ● P1          ● P2
              控制点不在曲线上
```

控制点还决定端点处的方向：起始切线沿 `P1 - P0`，终点切线沿 `P3 - P2`。后面让箭头贴合曲线时，正好用到第二个关系。API 参数含义参考：[PathCubic](https://doc.qt.io/qt-6.8/qml-qtquick-pathcubic.html)。

### 7.2 项目如何计算控制点

项目希望曲线随两个座位的相对位置自动转向，而不是固定让控制点向下移动。先取中心连线的单位方向 `u = (ux, uy)`，再构造与它垂直的单位法向量：

```text
n = (-uy, ux)
u · n = ux × (-uy) + uy × ux = 0
```

沿连线前进一部分，再沿法向偏移，就能得到控制点：

```qml
function controlPoint(fraction: real, side: real): point {
    return Qt.point(
        geometry.start.x + geometry.ux * geometry.length * fraction
            - side * geometry.uy * curveOffset,
        geometry.start.y + geometry.uy * geometry.length * fraction
            + side * geometry.ux * curveOffset)
}
```

分开理解这个公式：

```text
控制点 = 起点 + 连线方向 × 可用长度 × fraction
              + 法向量 × curveOffset × side
```

- `fraction` 取 `1 / 3` 或 `2 / 3`，确定沿连线前进到哪里。
- `side` 取 1 或 -1，确定向连线哪一侧偏移。
- `curveOffset` 控制偏移大小。

这里的 `side` 是相对于源到目标的方向定义的，不等于屏幕中固定的上侧或下侧。向右连线时，`u = (1, 0)`、`n = (0, 1)`，正侧就是屏幕下方；换一个方向，法向也会跟着改变。

### 7.3 两条反向曲线如何排列

设 `S = geometry.start`、`E = geometry.end`，两条路径分别使用：

| 路径 | 起点 | 第一个控制点 | 第二个控制点 | 终点 |
| --- | --- | --- | --- | --- |
| 源到目标 | S | `controlPoint(1/3, 1)` | `controlPoint(2/3, 1)` | E |
| 目标到源 | E | `controlPoint(2/3, -1)` | `controlPoint(1/3, -1)` | S |

第二条曲线不仅把偏移侧改成 -1，也把控制点的使用顺序反转。因为路径从 E 出发，它应先经过靠近 E 的控制点，再经过靠近 S 的控制点。

用 `S=(0,0)`、`E=(120,0)`、`curveOffset=24` 举例：

```text
正向：起点 (0,0)   → 控制点 (40,24)、(80,24)   → 终点 (120,0)
反向：起点 (120,0) → 控制点 (80,-24)、(40,-24) → 终点 (0,0)
```

正向曲线向下弯，反向曲线向上弯，两个箭头分别指向目标与源，表达交换。

控制点偏移 24，并不意味着曲线中点也偏移 24。本例在参数 `t=0.5` 处的偏移为 `0.75 × 24 = 18`。因此源码中的 32 上限指控制点偏移上限，不能直接称为曲线最大拱高。

### 7.4 为什么移动模式也使用 PathCubic

项目的偏移量是：

```qml
readonly property real curveOffset: swapping
    ? Math.min(geometry.length * 0.2,
               32 / Math.max(root.displayScale, 0.01)) : 0
```

交换时，偏移随可用长度增长，但有显示尺寸上限。移动时，偏移为零，两个控制点分别落在线段的三分之一与三分之二处，此时三次贝塞尔路径就是直线。

这样，移动和交换可以共用第一条 `PathCubic`；切换的是控制点位置与反向路径的颜色。

## 8. 第四步：让箭头沿着曲线末端指向目标

### 8.1 箭头不能一直朝中心连线的方向

对于直线，中心连线的方向就是路径方向。对于曲线，末端方向应看终点与最后一个控制点，不能继续只看 `(ux, uy)`。

三次贝塞尔曲线的终点导数是：

```text
B'(1) = 3 × (P3 - P2)
```

因此箭头的前进方向为 `P3 - P2`。但我们是从尖端向后找两个翼点，所以源码反过来取 `P2 - P3`。

项目的函数如下：

```qml
function wing(tip: point, control: point, side: real): point {
    const dx = control.x - tip.x, dy = control.y - tip.y
    const length = Math.max(Math.hypot(dx, dy), 0.000001)
    return Qt.point(
        tip.x + dx / length * arrowLength
            - side * dy / length * arrowHalfWidth,
        tip.y + dy / length * arrowLength
            + side * dx / length * arrowHalfWidth)
}
```

`tip` 是箭头尖端，`control` 是这条曲线的最后一个控制点。`length` 的下限用于避免归一化时除以零；它不会给零向量凭空补出一个有效方向。

把归一化后的向后方向记为 `v = (vx, vy)`，公式就是：

```text
翼点 = 尖端 + v × 箭头长度 + (-vy, vx) × 箭头半宽 × side
```

第一项位移让翼点退到尖端后面，第二项位移让两个翼点分居左右。

对于向右的直线，如果尖端是 `(100, 0)`，向后方向为 `(-1, 0)`，箭头长度 8、半宽 4，那么两个翼点就是 `(92, -4)` 和 `(92, 4)`。

### 8.2 正向与反向箭头分别传什么

| 箭头 | 尖端 `tip` | 最后一个控制点 `control` |
| --- | --- | --- |
| 目标端箭头 | `geometry.end` | `controlPoint(2/3, 1)` |
| 源端箭头 | `geometry.start` | `controlPoint(1/3, -1)` |

反向路径的最后一个控制点是靠近 S 的那一个，因此源端箭头要传 `controlPoint(1/3, -1)`。

路径本身仍然很简单。下面是目标端箭头的**项目代码节选**：

```qml
ShapePath {
    strokeColor: UiTheme.accent
    strokeWidth: adjustmentLink.lineWidth
    fillColor: "transparent"
    capStyle: ShapePath.RoundCap
    startX: adjustmentLink.wing(adjustmentLink.geometry.end,
                adjustmentLink.controlPoint(2 / 3, 1), 1).x
    startY: adjustmentLink.wing(adjustmentLink.geometry.end,
                adjustmentLink.controlPoint(2 / 3, 1), 1).y
    PathLine {
        x: adjustmentLink.geometry.end.x
        y: adjustmentLink.geometry.end.y
    }
    PathLine {
        x: adjustmentLink.wing(adjustmentLink.geometry.end,
               adjustmentLink.controlPoint(2 / 3, 1), -1).x
        y: adjustmentLink.wing(adjustmentLink.geometry.end,
               adjustmentLink.controlPoint(2 / 3, 1), -1).y
    }
}
```

项目没有显式写 `strokeStyle`，使用默认的实线。复杂的是翼点坐标的计算；真正绘制箭头只需要两个 `PathLine`。

## 9. 缩放、短距离与显示时机

### 9.1 几何跟随座位缩放，线宽反向补偿

项目的实际缩放系数为 `displayScale = fitScale * zoomFactor`，既包含视口适配，也包含用户缩放。`canvas` 对整张座位图应用这个缩放，因此线宽也会随之变化。

若希望显示线宽保持约 2 个设备无关像素，逻辑坐标中的线宽应设置为：

```qml
readonly property real lineWidth: 2 / Math.max(root.displayScale, 0.01)
```

在有效缩放不低于 0.01 时，有：

```text
显示线宽 = 逻辑线宽 × displayScale
         = (2 / displayScale) × displayScale
         = 2
```

| `displayScale` | 逻辑线宽 | 缩放后显示线宽 |
| --- | --- | --- |
| 0.5 | 4 | 2 |
| 1 | 2 | 2 |
| 1.25 | 1.6 | 2 |
| 2 | 1 | 2 |

`0.01` 是除数保护。缩放低于该阈值时，补偿会被限制，不能再保证上述恒等关系。

这里的“2”是 Qt Quick 的设备无关显示尺寸，不保证覆盖恰好两个物理像素。高 DPI 屏幕还要考虑设备像素比与抗锯齿。

由于虚线长度按线宽倍数计算，在同样的均匀缩放与补偿条件下，`[2, 1.5]` 的名义显示线段长度也能保持约 4、间隔约 3。本文按 Qt 6.8.3 解释源码中的补偿写法。

### 9.2 箭头不能把短连线全部占满

项目对箭头尺寸设置了距离限制：

```qml
readonly property real arrowLength: Math.min(
    8 / Math.max(root.displayScale, 0.01),
    geometry.length / (swapping ? 3 : 2))
readonly property real arrowHalfWidth: arrowLength / 2
```

第一项表达期望的显示长度约为 8；第二项限制箭头在短连线中的占比。移动模式不超过可用长度的一半，交换模式不超过三分之一。

例如，可用长度为 12、缩放为 1 时，移动箭头长为 6，交换箭头长为 4。距离足够大时才恢复到 8；因此箭头显示尺寸是有上限的，不是任何距离都固定为 8。

曲线偏移同理，取 `min(可用长度 × 0.2, 32 / 缩放)`，让相邻座位间的曲线更收敛。这些是项目的视觉约束，并非 Shapes 内置的避碰算法。

### 9.3 透明度、层级与裁剪各管一件事

```qml
z: 2
opacity: 0.45
```

`z: 2` 让辅助线画在同级座位内容上方。`opacity: 0.45` 统一淡化线身和箭头，降低视觉干扰，但不能把它理解成“最终所有像素的 alpha 都恰好是 0.45”；路径叠加和抗锯齿仍会影响合成。

连线位于 `Flickable` 内，超出视口的部分由 `viewport.clip: true` 裁剪。`anchors.fill` 负责尺寸关系，本身不提供裁剪。

### 9.4 什么时候不画线

项目的显示条件是：

```qml
visible: root.viewModel.selectingAdjustmentTarget && !viewport.moving
    && !root.viewportUpdatePending && geometry.length > 0
```

它要求正在选择调整目标、视口没有移动、没有待处理的视口更新，并且连线长度有效。

缺少目标、源目标相同或源座位未找到时，`geometry` 返回零长度，最终也会隐藏。项目还在拖动开始与滚轮处理时清空 `hoveredSeat`，避免视口变化后继续使用已经失效的悬停目标。

这些可见性和输入状态由项目维护；Shapes 根据收到的路径属性绘制，不会自行理解“正在调整座位”这件事。

## 10. 完整串联：项目连线代码节选

下面保留当前源码的连线 `Shape`，仅调整缩进，便于对照前面的分析。它不是独立演示程序，需要放回原来的 `canvas` 中理解。

| 外部对象 | 连线使用的内容 |
| --- | --- |
| `root` | `hoveredSeat`、`displayScale`、`viewportUpdatePending`、`viewModel` |
| `root.viewModel` | `selectingAdjustmentTarget`、`adjustmentSourceSeatId` |
| `seats` | 座位 `Repeater`，通过 `count` 和 `itemAt()` 找到源座位 |
| 座位委托 | `x/y/width/height`、`model.seatId`、`model.occupied` |
| `viewport` | `Flickable.moving`；外层同时负责视口裁剪 |
| `UiTheme` | 项目主题单例中的 `accent` 颜色 |

文中端点、控制点、箭头、缩放补偿的计算，都能在这段代码中找到。几何算法是项目自己的 JavaScript/QML 逻辑，`ShapePath` 和路径元素负责把计算结果显示出来。

```qml
Shape {
    id: adjustmentLink
    objectName: "seatAdjustmentLink"
    anchors.fill: parent
    z: 2
    opacity: 0.45
    // 与座位共用逻辑坐标，线宽和箭头尺寸反向补偿缩放。
    readonly property Item targetSeat: root.hoveredSeat
    readonly property Item sourceSeat: {
        if (!targetSeat || !root.viewModel.selectingAdjustmentTarget)
            return null
        for (let i = 0; i < seats.count; ++i) {
            const candidate = seats.itemAt(i)
            if (candidate && candidate.model.seatId === root.viewModel.adjustmentSourceSeatId)
                return candidate
        }
        return null
    }
    readonly property bool swapping: targetSeat !== null && targetSeat.model.occupied
    readonly property var geometry: {
        if (!sourceSeat || !targetSeat || sourceSeat === targetSeat)
            return { start: Qt.point(0, 0), end: Qt.point(0, 0), length: 0, ux: 0, uy: 0 }
        const sx = sourceSeat.x + sourceSeat.width / 2
        const sy = sourceSeat.y + sourceSeat.height - sourceSeat.width / 2
        const tx = targetSeat.x + targetSeat.width / 2
        const ty = targetSeat.y + targetSeat.height - targetSeat.width / 2
        const dx = tx - sx, dy = ty - sy
        const distance = Math.hypot(dx, dy)
        if (distance <= 0)
            return { start: Qt.point(sx, sy), end: Qt.point(tx, ty), length: 0, ux: 0, uy: 0 }
        const ux = dx / distance, uy = dy / distance
        const edgeFactor = Math.max(Math.abs(ux), Math.abs(uy))
        const sourceInset = sourceSeat.width / 2 / edgeFactor
        const targetInset = targetSeat.width / 2 / edgeFactor
        return { start: Qt.point(sx + ux * sourceInset, sy + uy * sourceInset),
            end: Qt.point(tx - ux * targetInset, ty - uy * targetInset),
            length: Math.max(0, distance - sourceInset - targetInset), ux: ux, uy: uy }
    }
    readonly property real lineWidth: 2 / Math.max(root.displayScale, 0.01)
    readonly property real arrowLength: Math.min(8 / Math.max(root.displayScale, 0.01), geometry.length / (swapping ? 3 : 2))
    readonly property real arrowHalfWidth: arrowLength / 2
    readonly property real curveOffset: swapping
        ? Math.min(geometry.length * 0.2, 32 / Math.max(root.displayScale, 0.01)) : 0
    visible: root.viewModel.selectingAdjustmentTarget && !viewport.moving
        && !root.viewportUpdatePending && geometry.length > 0

    function controlPoint(fraction: real, side: real): point {
        return Qt.point(geometry.start.x + geometry.ux * geometry.length * fraction - side * geometry.uy * curveOffset,
            geometry.start.y + geometry.uy * geometry.length * fraction + side * geometry.ux * curveOffset)
    }

    function wing(tip: point, control: point, side: real): point {
        const dx = control.x - tip.x, dy = control.y - tip.y
        const length = Math.max(Math.hypot(dx, dy), 0.000001)
        return Qt.point(tip.x + dx / length * arrowLength - side * dy / length * arrowHalfWidth,
            tip.y + dy / length * arrowLength + side * dx / length * arrowHalfWidth)
    }

    ShapePath {
        strokeColor: UiTheme.accent
        strokeWidth: adjustmentLink.lineWidth
        fillColor: "transparent"
        strokeStyle: ShapePath.DashLine
        dashPattern: [2, 1.5]
        capStyle: ShapePath.FlatCap
        startX: adjustmentLink.geometry.start.x
        startY: adjustmentLink.geometry.start.y
        PathCubic {
            control1X: adjustmentLink.controlPoint(1 / 3, 1).x
            control1Y: adjustmentLink.controlPoint(1 / 3, 1).y
            control2X: adjustmentLink.controlPoint(2 / 3, 1).x
            control2Y: adjustmentLink.controlPoint(2 / 3, 1).y
            x: adjustmentLink.geometry.end.x
            y: adjustmentLink.geometry.end.y
        }
    }
    ShapePath {
        strokeColor: adjustmentLink.swapping ? UiTheme.accent : "transparent"
        strokeWidth: adjustmentLink.lineWidth
        fillColor: "transparent"
        strokeStyle: ShapePath.DashLine
        dashPattern: [2, 1.5]
        capStyle: ShapePath.FlatCap
        startX: adjustmentLink.geometry.end.x
        startY: adjustmentLink.geometry.end.y
        PathCubic {
            control1X: adjustmentLink.controlPoint(2 / 3, -1).x
            control1Y: adjustmentLink.controlPoint(2 / 3, -1).y
            control2X: adjustmentLink.controlPoint(1 / 3, -1).x
            control2Y: adjustmentLink.controlPoint(1 / 3, -1).y
            x: adjustmentLink.geometry.start.x
            y: adjustmentLink.geometry.start.y
        }
    }
    ShapePath {
        strokeColor: UiTheme.accent
        strokeWidth: adjustmentLink.lineWidth
        fillColor: "transparent"
        capStyle: ShapePath.RoundCap
        startX: adjustmentLink.wing(adjustmentLink.geometry.end, adjustmentLink.controlPoint(2 / 3, 1), 1).x
        startY: adjustmentLink.wing(adjustmentLink.geometry.end, adjustmentLink.controlPoint(2 / 3, 1), 1).y
        PathLine { x: adjustmentLink.geometry.end.x; y: adjustmentLink.geometry.end.y }
        PathLine {
            x: adjustmentLink.wing(adjustmentLink.geometry.end, adjustmentLink.controlPoint(2 / 3, 1), -1).x
            y: adjustmentLink.wing(adjustmentLink.geometry.end, adjustmentLink.controlPoint(2 / 3, 1), -1).y
        }
    }
    ShapePath {
        strokeColor: adjustmentLink.swapping ? UiTheme.accent : "transparent"
        strokeWidth: adjustmentLink.lineWidth
        fillColor: "transparent"
        capStyle: ShapePath.RoundCap
        startX: adjustmentLink.wing(adjustmentLink.geometry.start, adjustmentLink.controlPoint(1 / 3, -1), 1).x
        startY: adjustmentLink.wing(adjustmentLink.geometry.start, adjustmentLink.controlPoint(1 / 3, -1), 1).y
        PathLine { x: adjustmentLink.geometry.start.x; y: adjustmentLink.geometry.start.y }
        PathLine {
            x: adjustmentLink.wing(adjustmentLink.geometry.start, adjustmentLink.controlPoint(1 / 3, -1), -1).x
            y: adjustmentLink.wing(adjustmentLink.geometry.start, adjustmentLink.controlPoint(1 / 3, -1), -1).y
        }
    }
}
```

## 11. 阅读与使用时常见的问题

| 现象 | 先检查什么 |
| --- | --- |
| 曲线或箭头出现多余色块 | 是否设置透明填充，是否另有填充渐变或填充项 |
| 线连在编号附近 | 是否误用了整个座位委托的中心，而非下方方块中心 |
| 线穿进座位块 | 是否计算了边缘截取；是否仍满足轴对齐正方形的前提 |
| 连线随缩放偏离座位 | 是否把逻辑坐标又乘了一次缩放，或跨坐标系直接使用 x/y |
| 箭头断成虚线 | 箭头是否与虚线身共用一个 ShapePath |
| 箭头没有贴合曲线末端 | 是否使用该曲线最后一个控制点计算方向 |
| 反向曲线绕得不对 | 反向路径是否交换了控制点顺序，并使用相反偏移侧 |
| 缩放后线宽太粗或太细 | 是否按实际 displayScale 进行补偿 |
| 相邻座位的箭头显得拥挤 | 是否保留 arrowLength 的距离限制与 curveOffset 的距离限制 |
| 滚动后出现指向旧目标的线 | 是否清理悬停状态并保留移动、视口更新时的隐藏条件 |

如果把示例迁移到矩形、圆形或旋转座位上，需要重新设计边界求交；如果连接大量节点，则还需要评估源节点查找、几何更新频率和实际渲染成本。当前实现适合解释这组座位辅助线，不能据此认定它已经解决所有图形连线问题。

## 12. 验证范围与延伸阅读

文章中的项目节选与 2026-11-02 的 `SeatMapView.qml` 连线块核对。验证只针对本文示例与说明，不代表重新验收了会议系统的全部交互。

本次实际完成的检查：

- 将最小直线示例、增加虚线和实线箭头后的版本、替换为三次贝塞尔曲线后的版本，分别提取为独立 QML 文件；三个版本均通过 Qt 6.8.3 的 `qmllint` 检查。
- 使用 Qt 6.8.3 的 `qml` 在离屏平台、软件渲染后端运行上述三个版本，通过临时截图回调获取图像，检查直线、虚线箭头和曲线的绘制结果。该检查不覆盖 Metal 等硬件后端的抗锯齿差异。
- 核验水平、垂直、45° 斜向的边缘端点，以及 0.5、1、1.25、2 倍缩放的线宽补偿；核验短距离箭头限制和控制点偏移与曲线中点偏移的数值。
- 检查完整项目连线节选与源码一致，以及文章内部文件链接和 README 新增索引的目标存在。

官方资料：

- [Qt Quick Shapes 类型概览](https://doc.qt.io/qt-6.8/qtquick-shapes-qmlmodule.html)
- [Shape：可视项、渲染方式与性能说明](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shape.html)
- [ShapePath：描边、填充、虚线和线帽](https://doc.qt.io/qt-6.8/qml-qtquick-shapes-shapepath.html)
- [PathCubic：三次贝塞尔曲线参数](https://doc.qt.io/qt-6.8/qml-qtquick-pathcubic.html)
- [Item.mapToItem()：跨坐标系映射](https://doc.qt.io/qt-6.8/qml-qtquick-item.html#mapToItem-method)

本仓库相关内容：[QML 学习资料](QML学习资料.md)、[Qt QML Signal 与分层事件流](Qt%20QML%20Signal与分层事件流.md)、[Qt Q_PROPERTY 与跨平台双向绑定对照](Qt%20Q_PROPERTY与跨平台双向绑定对照.md)。
