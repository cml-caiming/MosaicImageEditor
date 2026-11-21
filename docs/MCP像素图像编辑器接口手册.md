# MCP 像素图像编辑器接口手册

> **适用版本**：Mosaic Image Editor **1.1.0.4** ｜ **13 个工具，单档**（`serverInfo.variant = "full"`，`serverInfo.toolCount = 13`）
> **对象**：通过 MCP 驱动本编辑器的 AI Agent 及其编写者。
> **边界**：本文只讲**接口契约**（参数、返回、错误、约束）。画布模型与算法细节见编辑器使用说明 `readme.md`。

---

## 0. 基本模型

| 项 | 说明 |
|---|---|
| 谁持有文档 | **GUI 进程**（`MI_Editor.exe`）。协议层 `mcp_mosaic.exe` 只做握手、工具表、参数校验、错误翻译与转发 |
| 传输 | 客户端 ⇄ `mcp_mosaic.exe`：**stdio，换行分隔的 JSON-RPC 2.0**；`mcp_mosaic.exe` ⇄ GUI：命名管道 |
| 线程 | 所有执行都在 **GUI 的 UI 线程** → 读写天然一致，无并发读写问题 |
| 撤销粒度 | **一次工具调用 = 一次撤销单位**（等价人类的一次 Ctrl+Z）；`mosaic_batch` 整批仍算一次 |
| 文档 | **索引色**：每个像素存一个 `uint8` 调色板索引，颜色在调色板里 |
| 前置 | 编辑器必须开着；否则所有工具返回 `E_NO_GUI`（`retryable:true`） |

---

## 1. 通用约定

### 1.1 索引与颜色

| 概念 | 说明 |
|---|---|
| `index` | 调色板索引，合法域 `0 … (1<<depth)-1`；`depth` ∈ {1,2,4,8}，由文档决定 |
| `maxIndex` | `meta` 回传的合法上限 = `(1<<depth)-1`。**每次调用都读，不要缓存** |
| `color` 参数 | 二选一：`{"index":n}` 或 `{"rgb":"#RRGGBB"}`。同时给或缺省 → `E_BAD_ARG` |
| `used_index` | 服务端**实际写入**的索引。给 `rgb` 时会吸附到最近索引 → 断言按 `used_index` 对齐 |
| `argb` | **输出**字段一律 8 位 `#AARRGGBB`；**输入**的 `rgb` 是 6 位 `#RRGGBB`。两者不混用 |
| `transparentIndex` | 调色板中**第一个** `alpha==0` 的索引（**不是**索引 0）；没有则 `-1` |
| `palette[0]` | **不是恒定值**：索引色下不被重置（新建为 `0x00000000`；打开 `.pxp` 时照单全收文件里的值）；灰度模式下会被写成不透明灰阶 |

> **越界要过两道门**（都返回 `E_BAD_ARG`）：协议层查语法上限（`index`、坐标 ∈ [0,255]），GUI 查语义上限（`index < (1<<depth)`、坐标在 `canvasW × canvasH` 内）。

> **透明只影响导出**：画布渲染丢弃 alpha（`#00000000` 在画布上是纯黑）。「导入 / 导出透明」与「画布上看得见的颜色」不是一回事。

**默认调色板**：索引色新建时可用色数 = `1<<depth`（1bit→2 / 2bit→4 / 4bit→16 / 8bit→256），最亮项恒为 `#FFFCFCFC`；位深用不到的尾部条目被清零为 `0x00000000`。
**灰度**：`palette[i] = i * 255 / (2^depth − 1)` → `index 0` = 纯黑，`maxIndex` = 纯白（**1bit 白底要写 `background: 1`**）。

### 1.2 坐标与画布

| 项 | 值 |
|---|---|
| 坐标系 | 左上角原点，x 向右、y 向下，单位 = **画布像素** |
| 逻辑尺寸 | `meta.canvas.w / h`（`8..256`，**可能小于 256**） |
| **行跨度 stride** | **恒为 256**（≠ 逻辑宽度）。第 `y` 行第 `x` 像素在 `y*256 + x` |
| 取值范围 | `0 ≤ x < w`、`0 ≤ y < h`，否则 `E_BAD_ARG`（默认**不静默夹取**） |
| 作用帧 | 除显式带 `frame` 的工具外，一律**只作用于当前帧**（`meta.frame`） |

> `stride 256` **从不进入协议层**：所有返回数组（`indices`）都是**紧凑行优先**，行宽 = 请求的 `w`。

### 1.3 撤销

- 写工具 = **1 次撤销**；失败时**零副作用**（先在临时缓冲上算完才写回）。
- `meta.undoDepth` = 当前帧撤销栈深度；`meta.undoDepthBefore` = 本次调用之前的深度。写工具成功后 `undoDepth == undoDepthBefore + 1`。
- **撤销栈逐帧独立**；`mosaic_undo` 只作用于当前帧。
- 撤销可能恢复**别的画布尺寸 / 位深 / 调色板** → `meta` 每次都是现取的，不要缓存。
- 人类按 Ctrl+Z 也动同一个栈 → 需要真实深度就读 `undoDepth`，不要自己计数。

### 1.4 什么时候会被拒绝

| 情况 | 写类工具 | 读类工具 |
|---|---|---|
| 人类开着模态窗体 | `E_BUSY`（`retryable:true`） | 照常执行 |
| 帧正在播放 | `E_PLAYING`（`retryable:true`） | 照常执行 |
| 没有活动画布 | `E_NO_DOC` | `E_NO_DOC`（`mosaic_doc` 例外：返回 `canvasActive:false`） |
| UI 线程 5 s 内未处理 / 管道被占 | `E_BUSY` | 同 |
| 请求已过 deadline | `E_EXPIRED`（**绝不重试**） | 同 |

> `E_BUSY` 说「等一下」，`E_EXPIRED` 说「别再来一次」——语义相反，**不要合并处理**。判断只认 `code`，不要按 `message` 文本分支。

### 1.5 公共返回字段 `meta`

每个成功响应都带同一份 `meta`（返回字段不占用 `tools/list` 字节）：

| 字段 | 类型 | 含义 |
|---|---|---|
| `canvas` | object | `{w, h, stride, depth, colorMode, ...}`；`colorMode`：0 灰度 / 1 索引 |
| `frame` | int | **人类的当前帧**（不因查询而改变） |
| `frameCount` | int | 帧数 |
| `modified` | bool | 文档级（全局）未保存标记 |
| `maxIndex` | int | `(1<<depth)-1` |
| `undoDepth` / `undoDepthBefore` | int | 当前帧撤销栈深度 / 本次调用前的深度 |
| `transparentIndex` | int | 导出时写成透明的索引（`-1` = 不透明导出） |
| `copySkipIndex` | int | 帧间复制（`copy` op / `frame.duplicate`）跳过的「空像素」索引 |
| `path` | string | **当前文档路径**；无活动画布 → 空串。`.pxp` 优先，其次 `.pxa` |
| `loop` | bool | 文档级循环标志（`.pxa` 的 `flags.bit0`） |
| `frameWindow` | object\|null | 当前帧的帧窗口 `{x,y,w,h}`（信息性，**不是裁剪开关**；无帧时 `null`） |

`initialize` 响应里的 `serverInfo`：`{name, version, variant:"full", toolCount:13}`。

### 1.6 参数类型

整数参数接受 **JSON 数字**（整数值）与**纯整数文本**（如 `"12"`，可为负）。以下都返回 `E_BAD_ARG` 并带出实际类型（`(got string)` 等）：`"1.0"`、`"1e2"`、`" 1"`、`true`、`null`、数组、非整数值。

坐标参数接受 `[x,y]` 或 `{x,y}`；报错会指到具体叶子（`ops[0].paths[0].points[2]`、`(got {x=missing,y=2})`）。

---

## 2. 工具参考

### 2.1 `mosaic_view` — 看画布

返回**一帧**画布的**内容**（flat 展开：像素严格等于 `palette[index]`，不含斜面高光、棋盘格、网格、帧线框、参考层）。

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `zoom` | int 1..8 | ❌ | 最近邻放大（不引入新信息）。缺省 `clamp(1, 8, round(512/max(w,h)))`；超限静默钳制，实际值见 `meta.zoom` |
| `frame` | int | ❌ | 要渲染的帧 `0..frameCount-1`，缺省当前帧。**只读他帧，不切换当前帧** |
| `format` | string | ❌ | `"png"`（默认）／`"ascii"`（纯文本，给无视觉能力的模型） |
| `x` / `y` / `w` / `h` | int | ❌ | 只渲染该区域（与 `region_get` 同名同义）。缺省 `0 / 0 / canvasW-x / canvasH-y`；**越界 → `E_BAD_ARG`**；png 与 ascii 都生效 |

**返回（png）**

```jsonc
{ "meta": { ... },
  "frame": {"index":2, "count":3, "current":false},   // 本次实际渲染的帧
  "mode": "flat", "zoom": 8,
  "png_color_type": 3,        // 3 = indexed（含 PLTE）；6 = truecolor 回退
  "png_bytes": 2769,
  "thumb": {"grid":"0f3a0f3a…", "legend":{"0":7}, "note":"…"},  // 16×16 文本缩略
  "palette_used": [{"index":7,"argb":"#FFFF8800","count":812}], // count 降序，≤16 项
  "resource_uri": "mosaic://doc/frame/0.png" }
```

- `meta.frame` = 人的当前帧；顶层 `frame.index` = 本次渲染的帧。二者不同是正常的。
- `resource_uri` 带被渲染帧号；资源缓存是**单槽**，只有 `mosaic_view` 会填充 → 要 URI 就先 `mosaic_view`，不要依赖 `resources/list` 枚举。
- 输出**恒为整幅画布**，不按帧窗口裁剪；`zoom=1` 时 PNG 索引与画布 1:1（`mode == 'P'`，`getdata()` 即索引数组）。

**返回（ascii）**：`ascii: {step, w, h, legend, map, text, note, region}`

| 字段 | 说明 |
|---|---|
| `step` | 降采样步长（= 生效的 `zoom`，同 `meta.zoom`） |
| `w` / `h` | **单元格数**（`ceil(画布宽/step)` × `ceil(画布高/step)`），不是像素尺寸 |
| `legend` / `map` | 字符表 / `{字符: 调色板索引}` 对象 |
| `text` | 文本网格：行首 `%3d|` = y，其后第 k 字符 = `x = 区域左边界 + k*step`；首行标尺 `col|` 恰好 4 字符（与数据行等宽），每 10 格打一个十位数字 |
| `region` | 本次实际渲染区域 `{x,y,w,h}`（画布绝对坐标） |

- 字符按颜色**出现次数降序**映射到 `0-9a-zA-Z`（同次数按索引升序 = 确定性）；超过 62 色用 `~` 占位。
- `zoom` 在 ascii 下就是降采样步长；不传时自动放大到单元格数 ≤ **16 384**，仍超 → `E_BAD_ARG`。
- ⚠️ `zoom ≥ 2` 会抹掉细结构（轮圈缺一环、腿断没断、遮挡层级）→ **构图看 ascii，质量看 `region_get{format:"raw"}`**。

### 2.2 `mosaic_draw` — 画图元

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `verb` | enum | ✅ | `point` / `line` / `rect` / `rect_fill` / `circle` / `circle_fill` / `ellipse` / `ellipse_fill` / `polyline` / `polygon_fill` |
| `color` | object | ✅ | `{"index":n}` 或 `{"rgb":"#RRGGBB"}` |
| `paths` | array | ✅ | 每个元素 = 一个独立图元：`{"points":[[x,y],…], "closed":bool}` |
| `thickness` | int 1..8 | ❌ | 笔刷**边长**（画布像素），缺省 1。同 GUI 工具栏线宽，但**只影响本次调用** |

| `verb` | 每条 path 点数 | 语义 |
|---|:--:|---|
| `point` | 恰好 1 | 一个点 = 一个 `paths` 元素 |
| `line` | 恰好 2 | 独立直线 |
| `rect` / `rect_fill` | 恰好 2 | 两点为对角（顺序无关，内部取 min/max）；边框 / 填充 |
| `circle` / `circle_fill` | 恰好 2 | 两点定外接框，圆心 = 框心，半径 = 框内可容纳的最大**整数**半径（内接） |
| `ellipse` / `ellipse_fill` | 恰好 2 | 两点定外接框；像素中心坐标下**内接**于该框（偶数宽高无 0.5 px 偏移；极扁的偶数框最外列/行可能差 1–3 px）；宽高自由 |
| `polyline` | ≥2 | 折线；`closed:true` 时首尾相接 |
| `polygon_fill` | ≥3 | 闭合多边形 + **奇偶规则**填充；顶点隐式闭合；越界 → `E_BAD_ARG`（不钳制） |

- `circle`（整数半径）与 `ellipse`（内接）口径不同：4×4 框 → 圆 5 px、椭圆 12 px。
- `polygon_fill` 的边界约定：**y 半开** `[y0,y1)`、**x 跨度含两端** `[ceil(xa)..floor(xb)]`（11×11 方形填 110 px）。
- 🔴 **自交多边形会静默画出碎块**（不报错）→ 凸形请按**凸包顺序**给点。
- 🔴 **末行与亚像素端点会丢**（半开区间 + 顶点附近宽度 <1 px）→ 尖顶单独补 1–3 px 短线；通用解法是「整块画低 1 px + 用描边盖住接缝」。
- 上限：`paths ≤ 64` 条；单条 path `≤ 256` 点；总点数 `≤ 4096`。
- `thickness`：仅描边类生效（`point`/`line`/`rect`/`circle`/`ellipse`/`polyline`）；**填充类忽略**（`used_thickness` 恒 1）。`1`=`(x,y)`、`2`=`(x..x+1, y..y+1)`（向右下）、`3`=`(x-1..x+1, y-1..y+1)`（居中）。笔刷是**方块沿轮廓盖章**，外接框每边外扩 `(thickness-1)/2`。
- 越界 → `E_BAD_ARG`，零副作用。

**返回**：`meta` + `used_index` + `used_thickness` + `changed_pixels`（+ `changed_bbox`）

### 2.3 `mosaic_fill` — 油漆桶

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `x` / `y` | int | ✅ | 种子像素（必须在画布内） |
| `color` | object | ✅ | 同上 |

**四邻域**连通填充。**返回**：`meta` + `used_index` + `changed_pixels`（+ `changed_bbox`）

### 2.4 `mosaic_pixel_get` — 读像素

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `points` | array | ✅ | `[{"x":n,"y":n},…]`，**≤ 256** 个 |

- **只读当前帧**（没有 `frame` 参数）。读他帧用 `mosaic_region_get {frame:N, format:"raw"}`。
- 返回里 `meta.frame` = 本次实际读取的帧号；**没有**顶层 `frame` 对象（`view` / `region_get` 有）。
- 响应里没有顶层 `frame`；判帧号一律读 `meta.frame`。
- 不要为了读其他帧去 `frame{action:"set"}` —— 那会切走人类正在看的那一帧。

**返回**：`points: [{x, y, index, argb, indexUsage}]`

| 字段 | 说明 |
|---|---|
| `index` | 该点的调色板索引 |
| `argb` | 该索引的颜色，8 位 `#AARRGGBB` |
| `indexUsage` | 该索引在**逻辑画布** `w×h` 上的出现次数（不含 stride 填充区）—— 可当免费直方图 |

> 上限不要往上调：每点约 60~70 B，16384 点会超过管道单帧 1 MB。要整帧像素走 `region_get{format:"raw"}`。

### 2.5 `mosaic_open` — 打开工程

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `path` | string | ✅ | `.pxp` 工程文件路径，必须存在；**只支持 `.pxp`**（其它扩展名 → `E_BAD_ARG`，不做格式转换） |

- **替换当前文档**（破坏性）。
- 覆盖前**先做只读预检**（头部级；当前文档脏且有工程路径时另做帧级预检）→ 目标打不开就**零副作用**失败。
- 覆盖前的处理：**有工程路径 → 静默自动保存**；**无路径 → 静默丢弃**；全程**不弹框**。
- 覆盖前另做一次 shadow-save 到 `%TEMP%\MosaicMcp\autosave-<ts>.pxp`。
- **先读后切**：全部帧读成功才切换，任何失败 → 原文档原封不动，返回 `E_IO` / `E_PATH`。
- 单文件最多 **512 帧**（`MAX_FRAMES`），超出 → `E_IO`「帧数超过上限」。

**返回**：`meta` + `auto_saved_to?` + `shadow_saved_to?`（+ `shadow_warning?`）

| 场景 | `auto_saved_to` | `shadow_saved_to` |
|---|---|---|
| 文档干净 | 无 | 有 |
| 脏 + 有工程路径 | **有**（真保存） | 有 |
| 脏 + 无路径 | 无 | 有（唯一安全副本） |

> 字段名与 `mosaic_new` 不对称（那里是 `auto_saved`(bool) + `saved_to`(路径)）—— 按字段名读，不要按对称性推断。

### 2.6 `mosaic_save` — 保存工程 / 导出发布格式

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `path` | string | ✅ | 目标路径；扩展名必须是 `.pxp` / `.pxa` / `.gif` / `.png` / `.bmp` 之一 |
| `format` | string | ❌ | `pxp` / `pxa` / `gif` / `png` / `bmp`；缺省 = 按扩展名推断 |
| `transparent` | bool | ❌ | **只对 `gif` / `png` 有效**；缺省 = 文档开关。其它格式给它 → `E_BAD_ARG` |
| `overwrite` | bool | ❌ | 缺省 `false`；目标已存在 → `E_EXISTS`；置 `true` 才覆盖 |

| 情形 | 结果 |
|---|---|
| `format` 缺省 + 扩展名可识别 | 按扩展名导出 |
| `format` 缺省 + 扩展名不可识别 | `E_BAD_ARG`（消息列出 5 种扩展名） |
| `format` 显式 + 与扩展名不一致 | `E_BAD_ARG`（消息写明期望扩展名），**零副作用** |

| 格式 | 导出内容 |
|---|---|
| `.pxp` | 工程（唯一保留撤销栈的格式） |
| `.pxa` | 全帧动画（帧序列 + 帧窗口 + 延时 + 合成模式 + 循环标志） |
| `.gif` | 全帧动画（逐帧延时 + 循环标志） |
| `.png` / `.bmp` | 当前帧单张图 |

**返回**：`.pxp` → `meta` + `path` + `bytes_written`；发布格式 → `meta` + `path` + `format` + `bytes_written`。

- `.pxp` 成功后工程路径改指新路径（等价「另存为」）；发布导出**不改**工程路径。
- 导出失败（路径不可写 / 磁盘满）→ `E_IO`。

### 2.7 `mosaic_new` — 新建空白画布

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `width` / `height` | int | ✅ | `8..256` |
| `depth` | int | ✅ | `1` / `2` / `4` / `8` |
| `colorMode` | int | ❌ | `0` 灰度 / `1` 索引色（默认 1） |
| `palette` | array | ❌ | `[{"index":0,"rgb":"#FFFFFF"}]` 或 `[{"index":0,"argb":"#AARRGGBB"}]`，1..256 项 |
| `background` | int | ❌ | 新画布整幅填该索引，缺省 0；必须 `< (1<<depth)` |
| `delayMs` | int | ❌ | 首帧延时 `10..65535`，缺省 100 |
| `loop` | bool | ❌ | 文档级循环标志 |

- **替换当前文档**（破坏性）。无未保存改动时什么都不做；有改动时：**有工程路径 → 静默自动保存**（落点见 `saved_to`）；**无路径 → 静默丢弃**（`discarded_unsaved:true`）。全程零弹框。自动保存失败 → `E_IO` 且旧文档原封不动。
- 参数越界 → `E_BAD_ARG`，**一个字节都不写**（先校验、后自动保存、再建画布）。
- 建完：单帧、撤销栈空、`modified=false`、工程路径被清空（之后的 `save` 是「另存为」）。
- `palette` 在建好画布之后灌入（不会被默认调色板覆盖）。
- `background` **只作用于新建出来的那一帧**；之后 `frame{action:"add"}` 的新帧是空白帧（索引全 0），不继承背景。背景只填逻辑区域 `w×h`，不碰 stride 填充列。
- 没有「保留原调色板」选项：需要保留就先用 `palette{get}` 读出，新建后灌回。

**返回**：`meta` + `palette_applied` + `background`（`{index, argb, transparent}` 实际生效值）+ `discarded_unsaved` + `auto_saved` + `saved_to?`

| `auto_saved` | `saved_to` | `discarded_unsaved` | 发生了什么 |
|:--:|---|:--:|---|
| `false` | 无 | `false` | 文档本来就干净 |
| `true` | 有 | `false` | 覆盖前把未保存改动存盘了 |
| `false` | 无 | `true` | 没有路径可存 → 未保存改动被丢弃 |

### 2.8 `mosaic_palette` — 调色板读写

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `action` | string | ✅ | `"get"`（只读）或 `"set"`（写） |
| `entries` | array | `set` 时必填 | `[{"index":n,"rgb":"#RRGGBB"}]` 或 `[{"index":n,"argb":"#AARRGGBB"}]`，1..256 项 |

**`get`** 返回全部 **256** 项：`entries: [{index, argb, usedCount}]`，另带 `depth` / `colorMode` / `transparentIndex`。

- `usedCount` = 该索引在**当前帧画布**上的像素数（不含 stride 填充区）。
- ⚠️ 低色深时尾部是**残留值** → **只信前 `maxIndex+1` 项**。

**`set`**：一步撤销（可撤销）；只改颜色不动像素；`index` 必须 `< (1<<depth)`，越界或同一 `index` 给两次 → `E_BAD_ARG`。`rgb` 的 alpha 强制 `0xFF`；`argb` 用显式 alpha。置 `alpha==0` 不拒绝，但该索引在用时会返回 `warnings`（GIF 导出会把它当透明色）。
**返回**：`meta` + `updated` + `transparentIndex` + `warnings?`

### 2.9 `mosaic_undo` — 撤销 / 重做

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `action` | string | ✅ | `"undo"` 或 `"redo"` |
| `steps` | int | ❌ | 默认 1；超过剩余只做到底，**不报错** |

**返回**：`meta` + `action` + `steps_applied` + `undoDepth` + `redoDepth` + `changed_pixels` + `changed_bbox`

- 只作用于**当前帧**的撤销栈。栈空 → `E_UNDO_EMPTY`。
- `changed_bbox` 是**闭区间** `{x,y,w,h}`；无变化时必须是 `null`。

### 2.10 `mosaic_frame` — 帧操作（8 动作）

| 参数 | 类型 | 必填 | 适用动作 |
|---|---|:--:|---|
| `action` | string | ✅ | `list` / `set` / `add` / `duplicate` / `delete` / `setMode` / `setDelay` / `setLoop` |
| `index` | int | `set` 时必填 | 目标帧 `0..frameCount-1`；**只对 `set` / `setDelay` / `duplicate` 合法**（其它动作带上 → `E_BAD_ARG`）。`setDelay` 缺省当前帧；`duplicate` 缺省复制当前帧 |
| `count` | int | ❌ | `1..512`；**只对 `add` / `duplicate` 合法**；缺省 1 |
| `delayMs` | int | ❌ | `10..65535`。`add`/`duplicate` 用它设新帧（缺省**继承源帧**）；`setDelay` 必填 |
| `compositeMode` | int | ❌ | `0..7`：`bit0` = 合成前整幅清零，`bit1..2` = 算子（0 覆盖 / 1 XOR / 2 AND / 3 OR）。用于 `add` / `duplicate` / `setMode` |
| `loop` | bool | ❌ | `setLoop` 必填；其它动作带上 → `E_BAD_ARG` |

**`list`**（只读）返回 `current` + `frames: [{index, delayMs, window, sizeBytes, hash, undoDepth, modified, compositeMode}]`

| 字段 | 说明 |
|---|---|
| `window` | 帧窗口（画布像素坐标）—— **不是裁剪开关**，出图恒为整幅画布 |
| `sizeBytes` | 该帧 **LZ4 压缩后**的字节数（不是 65536） |
| `hash` | 内容 **CRC32**，8 位小写 hex；口径 = 逻辑画布 `w×h` 行优先连续（不含缓冲里逻辑尺寸外的隐藏像素）→ 与 `region_get{format:"crc32"}` **同源**，可直接交叉断言。坏帧 → 空串 |
| `undoDepth` | 该帧自己的撤销栈深度（逐帧独立） |
| `modified` | **文档级**标记（各帧相同，别当 per-frame 用） |

**`set`**：走 GUI 同一条切帧路径；**不产生撤销步、不改文档**、不置 `modified`；切到当前帧 → `changed:false`。返回 `current` / `previous` / `changed`。播放中调用**照常执行**（不返回 `E_PLAYING`），但播放会从新的当前帧继续推进。

**`add` / `duplicate` / `delete`**（写类，**不可撤销**，`undoable:false`）

| action | 语义 | `frameCount` | 执行后当前帧 |
|---|---|---|---|
| `add` | 追加到文档**末尾**一个空白帧（像素全 0） | +N | 新帧 |
| `duplicate` | 追加到末尾一个源帧副本（像素 / 窗口 / delay / 合成模式全复制，**不含撤销历史**） | +N | 带 `index` 时**不切**，否则切到新帧 |
| `delete` | 删除当前帧 | −1 | 删的是末帧 → 回退到 `frameCount-1`；否则指向原下一帧 |

- 落点恒为**文档末尾** → 既有帧索引**不变**（帧索引 = 创建顺序）。
- `count:N` 一次追加 N 帧，N 帧全部继承**同一个**源帧延时；`frameCount + count > 512` → `E_BAD_ARG` 且**零副作用**。
- 返回：`action` + `frame`（产出的帧） + `previous` + `current` + `frameCount` + `delayMs` + `hash`（`add`/`duplicate` 有） + `undoable:false`。
- 单帧文档 `delete` → `E_BAD_ARG`。

**`setMode`**：改当前帧合成模式，**可撤销**，同值 → `changed:false` 不压栈。返回 `compositeMode` / `previousCompositeMode` / `changed` / `undoable:true`。
⚠️ 算子只作用于该帧窗口内（清除标志清整幅）；作用在**索引**上而不是颜色上；帧 i 的模式决定它叠加到帧 0..i-1 的方式，**最后一帧的模式看不到效果**。⚠️ `.pxp` **不持久化**合成模式 → 存盘重开归 0；只有 `.pxa` 保留。

**`setDelay`**：改延时，**可撤销**（压帧描述符快照，进**目标帧自己**的撤销栈）；同值 → `changed:false`。`index` 只改指定帧，**绝不切换**人的当前帧（返回 `current` = 未变的当前帧、`switched:false`）。返回 `frame` / `delayMs` / `previousDelayMs` / `changed` / `current` / `switched` / `undoable:true`。会置 `modified`。

**`setLoop`**：文档级循环标志（单一真相源 `m_loopFlag`，同步工具栏）。**不可撤销**（`undoable:false`）；**只在 `.pxa` 导出时生效**，`.pxp` 不存 → 存盘重开归默认。返回 `loop` / `previous` / `changed` / `undoable:false`。

**帧动画的调用节奏**：`batch`（第 0 帧）→ 循环 `{"action":"duplicate"} → batch` → 共 `2N+1` 次调用画完 N 帧（`duplicate` 自动切到新帧，不要补 `set`）。
**视觉周期自检**：`list` 的 `hash` 按值分组 —— 全不同 = 每帧新姿态；出现成对相同 = 周期 < 帧数。`视觉周期 = 对称角 / gcd(步进, 对称角)`（= `LCM(步进,对称角)/步进`）。`hash` 全不同也**不代表主运动正确**（旁支元素会撑开哈希）。

### 2.11 `mosaic_region_get` — 区域统计 / 指纹

| 参数 | 类型 | 必填 | 缺省 | 说明 |
|---|---|:--:|---|---|
| `x` / `y` | int | ❌ | 0 | 区域左上角（画布绝对坐标） |
| `w` / `h` | int | ❌ | `canvasW-x` / `canvasH-y` | 区域宽高 |
| `frame` | int | ❌ | 当前帧 | 只读该帧，**不切换当前帧** |
| `format` | string | ❌ | `"runs"` | `"runs"` / `"raw"` / `"sha256"` / `"crc32"` |
| `expect` | string | ❌ | — | 期望指纹：**64 hex = sha256**、**8 hex = crc32**（按长度判别）→ 进入比对模式 |

| `format` | 返回 | 上限 |
|---|---|---|
| `runs` | 行游程 `[[y, x0, index, len], …]` + `runsTruncated` | 2 048 段（超出置 `runsTruncated:true`，只留前缀） |
| `raw` | `indices`（行优先 `y` 后 `x`，每格 `0..255`）+ `pixels` | **16 384 px**（= 128×128），超出 `E_BAD_ARG`（不截断） |
| `sha256` | `sha256`（区域全部像素的哈希）+ `pixels` | 无 |
| `crc32` | `crc32`（同一串字节的 CRC-32/ISO-HDLC，8 位小写 hex） | 无 |

**通用返回**：`meta` + `frame{index,count,current}` + `region{x,y,w,h}` + `transparentIndex` + `histogram[{index,argb,count}]`（count 降序）+ `backgroundIndex` / `backgroundCount`（区域众数）+ `foregroundCount` + `bbox`（非背景像素外接框，无则 `null`）。

- `raw` / `sha256` 的返回里**没有** `runs` / `runsTruncated`。
- `raw` 的 `indices` 与 `sha256` 哈希的是同一串字节（行优先、只含逻辑区域）。
- 🔴 **`indices` 的行宽 = 本次请求的 `w`**，与画布宽 / stride 256 **无关**：
  `x = region.x + (i % w)`、`y = region.y + floor(i / w)`。按画布宽换算会**整行错位**（整画布读回时 `w == canvasW` 才恰好等价）。
- `runs` 的坐标是**绝对画布坐标**；同索引相邻像素合并成最大连续段；无损可还原。
- 256×256 整幅的 `runs` 会截断（上限 2 048 段）→ 整幅用 `sha256`；`raw` 整幅 65 536 px 超上限 → 按 ≤128×128 分片。

**比对模式 `expect`**：只回 `meta` / `region` / `frame` / `pixels` / `algorithm`（`sha256`|`crc32`）/ `match` / `expected` / `actual`，**不回像素载荷**。与 `format:"raw"` / `"runs"` 同传 → `E_BAD_ARG`；与 `"sha256"` / `"crc32"` 同传时必须与按长度推断的算法一致。

- 越界（`x<0` / `w<1` / `x+w>canvasW` / `frame` 越界）→ `E_BAD_ARG`，零副作用。
- 纯读：不产生撤销步、不改 `modified`、不切换帧，幂等。

### 2.12 `mosaic_batch` — 原子批绘图

一批 op 顺序执行，**一次落笔、一次撤销**；任一 op 非法 → 整批 `E_BAD_ARG` 且**画布零改动**（两遍处理：先全量校验，再在临时缓冲上落笔）。

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `ops` | array | ✅ | **1..256** 项；`atomic:true` 时上限 **2048** |
| `atomic` | bool | ❌ | `true` → 上限 256 → 2048，整批仍是 1 次撤销（服务端预算 30 s）；缺省 `false` |
| `thickness` | int 1..8 | ❌ | 本批所有**描边类** op 的笔宽（填充类忽略），缺省 1；返回 `used_thickness` |
| `reportOps` | bool | ❌ | `true` → 返回多一个 `ops` 数组（逐 op `changed` / `bbox`）；缺省 `false`（响应与不带该参数时逐字节一致） |
| `clip` | bool | ❌ | `true` = 越界几何**裁剪**而不是整批报错；缺省 `false`（越界即 `E_BAD_ARG`） |

**四种 op**（`kind` 必填；出现未声明的键 → `E_BAD_ARG`；每种都可带 `frame`）

| `kind` | 允许的键 | 说明 |
|---|---|---|
| `draw` | `kind`, `verb`, `color`, `paths`, `thickness`(op 级), `frame` | 图元，`verb` 取值与 `mosaic_draw` 相同；`paths` 每项 `{points, closed}` |
| `fill` | `kind`, `x`, `y`, `color`, `frame` | 洪水填充种子点 |
| `copy` | `kind`, `srcFrame`, `x`, `y`, `w`, `h`, `dstX`, `dstY`, `mode`, `skipIndex`, `frame` | 帧间区域复制（源 = `srcFrame` 的矩形；目标 = `frame`（缺省当前帧）的 `(dstX,dstY)`） |
| `write` | `kind`, `indices`, `x`, `y`, `w`, `h`, `frame` | 像素直写（整帧或矩形） |

**`kind:"write"`**

| 字段 | 必填 | 缺省 | 说明 |
|---|:--:|---|---|
| `indices` | ✅ | — | 长度**必须 == `w*h`**；**行优先（先 y 后 x）**，与 `region_get{format:"raw"}` 的 `indices` 完全同序 → 读回来改完可直接写回 |
| `x` / `y` | ❌ | 0 | 目标矩形左上角（画布像素） |
| `w` / `h` | ❌ | `canvasW-x` / `canvasH-y` | 目标矩形尺寸（≥1，且 `w*h ≤ 65536`） |

- 索引值域 `0 .. (1<<depth)-1`；越界 → `E_BAD_ARG`，零副作用。
- 🔴 缓冲区恒为 256×256，**写入按行跨度 256 推进**（按逻辑宽寻址会在小画布上写串行）。
- 载荷闸：**单请求所有 `write` op 的 `indices` 总数 ≤ 131 072**（≈450 KB，低于管道单帧 1 MB）；超了 → `E_BAD_ARG`（提示拆批）。
- 整帧写法：`{ "kind":"write", "indices":[...] }`（`x/y` 缺省 0，`w/h` 缺省到画布右下角）。

**`kind:"copy"`**

| 字段 | 必填 | 缺省 | 说明 |
|---|:--:|---|---|
| `srcFrame` | ✅ | — | 源帧号 |
| `x` / `y` / `w` / `h` | ✅ | — | 源矩形（原始画布坐标，必须整块在画布内） |
| `dstX` / `dstY` | ✅ | — | 目标左上角（目标帧画布坐标） |
| `mode` | ❌ | `"copy"` | `"copy"` = 整块覆盖（透明像素也写，等于擦掉目标背景）；`"stamp"` = 只写非透明像素 |
| `skipIndex` | ❌ | 第一个 `alpha==0` 的索引 | 仅 `stamp` 用；非整数或 `≥ 1<<depth` → `E_BAD_ARG`；调色板没有透明索引时 `stamp` 退化为 `copy` |

- 源 = 该帧**已提交**的像素；但 `srcFrame == 目标帧` 时源读**本批工作缓冲** → 同批前面的 op 对它可见。
- 同帧且源/目标重叠是安全的（实现先做源矩形快照）；源帧不被修改。

**op 级 `frame`（跨帧写入）**

- 缺省 = 当前帧；显式 `frame:N` → 写第 N 帧，**不切当前帧、不动人正在看的画面**。
- 🔴 **撤销按帧各压一次**：一批触及 N 帧 → 每个被改帧在**自己的**撤销栈里各压一步（在哪帧 Ctrl+Z 只回退那帧；没有跨帧事务）。零变化的帧不压栈。
- 响应里的 `changed_pixels` / `changed_bbox` 始终是**当前帧**口径；触及 >1 帧时另给 `frames_touched`（升序）。
- 非当前帧写完会同步压缩快照，切帧回来内容不丢。

**`clip`：越界裁剪（缺省 `false`）**

| `kind` | `clip:true` 的行为 |
|---|---|
| `draw` | 越界像素静默丢弃（图元照画）；整条 path 全在外 = 什么都不画 |
| `fill` | 种子点在画布外 → 跳过该 op |
| `copy` | 源/目标矩形裁剪到画布，只写两边都在画布内的像素；完全不落点 → 跳过 |
| `write` | 矩形裁剪到画布，只写交集；`indices` 按行列偏移取值 |

- `clip:true` 时 path 坐标幅值上限 **±65535**（Bresenham 循环次数 = 坐标跨度），超限 → `E_BAD_ARG`。
- `clip` 用在其它工具上 → `E_BAD_ARG`。

**返回**：`meta` + `applied`（已应用 op 数）+ `used_thickness` + `changed_pixels` + `changed_bbox`（无变化 → `null`）+ `ops`（仅 `reportOps:true`）+ `frames_touched`（仅 >1 帧时）。

`reportOps` 的 `ops` 项：`{op, changed, bbox}`

| 字段 | 说明 |
|---|---|
| `op` | 该 op 在请求 `ops` 数组里的下标（0 起） |
| `changed` | **该 op 相对它执行前**的像素变化数（重复画同色 = 0） |
| `bbox` | 该 op 变化像素的外接框；`changed == 0` 时恒为 `null` |

> `sum(ops[].changed) - changed_pixels` = 被后续 op 盖掉的量 → 「这一笔被涂掉了」可机检。它不回答「某个 op 的贡献最后是否可见」，那要看最终画面。

- 越界错误会指到具体叶子：`ops[0].paths[0].points[2] point out of canvas (32x32): [99,99]`。
- **帧动画固定套路**：`duplicate` 出来的帧，**第一个 op 必须整屏刷背景**（否则上一帧轮廓会留下）；`add` 出来的帧是空白帧，**不需要刷**。

### 2.13 `mosaic_doc` — 只读身份

| 参数 | 类型 | 必填 | 说明 |
|---|---|:--:|---|
| `focus` | bool | ❌ | `true` = 把绑定的编辑器窗口提到前台（不碰画布、不产生撤销点）；缺省 `false` |

**返回**：`meta` + `pid` + `hwnd` + `title` + `focused` + `focus_requested` + `focus_applied` + `canvasActive` + `frameCount` + `modalActive` + `playing`

| 字段 | 说明 |
|---|---|
| `pid` / `hwnd` | 绑定的是哪个编辑器进程 / 窗口（多开时唯一的身份凭据） |
| `focused` | 调用那一刻窗口是否在前台 |
| `focus_applied` | `focus:true` 时提窗是否成功（被系统拒绝**不报错**，只是没提到前面） |
| `canvasActive` | 有无活动画布；`false` → 写类工具 `E_NO_DOC`，且 `meta.path` 恒为空串 |
| `modalActive` | 有人正在操作界面（模态窗体开着）→ 写类工具 `E_BUSY` |
| `playing` | 正在播帧 → 写类工具 `E_PLAYING` |

> ⚠️ **文档路径在 `meta.path`，本工具没有顶层 `path` 字段**。开工前先 `mosaic_doc` 确认 `pid` / `canvasActive` / `modalActive` / `playing` 再动手。多实例可用环境变量 `MOSAIC_MCP_TARGET_PID=<pid>` 指定目标实例。

---

## 3. 错误码与重试

| code | 含义 | `retryable` | 该怎么办 |
|---|---|:--:|---|
| `E_BAD_ARG` | 参数缺失 / 类型错 / 越界（坐标、索引、点数、颜色形态、扩展名） | ❌ | 改参数再试 |
| `E_NO_DOC` | 当前没有文档 | ❌ | 先 `mosaic_new` 或 `mosaic_open` |
| `E_NO_GUI` | GUI 没开 / 已退出 / MCP 总开关被关闭 | ✅ | 等编辑器起来再试（**不是**「人不在」） |
| `E_BUSY` | UI 线程当前不可用：模态窗体打开中、5 s 内未处理、管道被占 | ✅ | 稍后重试 |
| `E_EXPIRED` | 请求到达时已过 deadline，未执行任何副作用 | ❌ | **绝不重试** |
| `E_PLAYING` | 帧播放中，拒绝写操作 | ✅ | 停播后重试 |
| `E_UNDO_EMPTY` | 当前帧撤销 / 重做栈已空 | ❌ | 先画一笔 |
| `E_PATH` | 路径非法 / 不存在 / 无权限 / 目录不存在（不自动建目录） | ❌ | 改路径 |
| `E_IO` | 读写失败（磁盘 / 格式） | ⚠️ | `open` 失败时原文档未动，可重试 |
| `E_EXISTS` | `overwrite=false` 且目标已存在 | ❌ | 改路径或置 `overwrite:true` |
| `E_HOST_GONE` | 连上之后管道中途断开 | ✅ | 重连后重试 |
| `E_INTERNAL` | 其他异常 | ⚠️ | 上报，不要盲目重试 |

`E_CANCELLED` 目前没有触发路径（保留码位，不会返回）。

错误结果形态：`isError = true`，且**错误 JSON 同时出现在 `content[0].text` 里**：

```json
{"error":{"code":"E_BUSY","message":"modal dialog is open","retryable":true,"hint":"a human is using the UI; retry later"}}
```

`message` 给人看，`hint` 给 Agent 看（未知码不带 `hint`）。传输层错误（`E_NO_GUI` / `E_BUSY`）另带 `error.data.detail` 说明原因（试过哪些管道 / 实例）。

---

## 4. 典型流程

**看 → 画 → 核对**

```
① mosaic_view  { "zoom": 1 }                      → 尺寸、调色板、构图
② mosaic_draw  { "verb":"rect_fill", "color":{"index":1}, "paths":[{"points":[[2,2],[9,9]]}] }
③ mosaic_region_get { "format":"histogram"等价: 省略 format }   → 数「画对没有」
```

**从零画一张图**

```
① mosaic_new  { "width":128, "height":48, "depth":1, "background":1 }
② mosaic_batch { "thickness":3, "ops":[ {"kind":"draw","verb":"circle","color":{"index":1},"paths":[…]} ] }
③ mosaic_view { "zoom":1 }
④ mosaic_save { "path":"D:\\art\\out.pxp", "overwrite":true }
```

**多帧动画（`2N+1` 次调用）**

```
① mosaic_new {…} 或 mosaic_open {…}
② mosaic_batch {…}                                    # 第 0 帧
③ 循环 N-1 次： mosaic_frame {"action":"duplicate"} → mosaic_batch {…}
④ mosaic_frame {"action":"list"}                      # 看 hash：帧是否真的不同
⑤ mosaic_save {"path":"out.gif"}                      # 全帧导出
```

**整帧直写（一帧算好的索引图）**

```
① mosaic_region_get { "frame":N, "format":"raw" }     # 拿到 indices（行宽 = w）
② 本地改完 → mosaic_batch { "ops":[ {"kind":"write","indices":[…]} ] }
```

**写别的帧而不挪动人的视图**

```
mosaic_batch { "ops":[ {"kind":"write","frame":3,"indices":[…]} ] }
```

---

## 5. 排障

| 症状 | 原因 | 处理 |
|---|---|---|
| 图案「斜切」 | 把 `stride 256` 当成了宽度 | 一律 `row*256 + col`；宽度取 `meta.canvas.w` |
| 画的颜色不是给的色 | `rgb` 被吸附到最近索引 | 断言按 `used_index`；索引色下**直接给 `index`** |
| 参数看着没问题却 `E_BAD_ARG` | 索引 ≥ `1<<depth`，或坐标超出 `canvasW×canvasH` | 读 `meta.maxIndex` / `meta.canvas.w,h`（别缓存） |
| 一次画了多个图元只撤销一次 | 设计如此（1 次调用 = 1 次撤销） | 要分开撤销就分多次调用 |
| `E_NO_GUI` 一直来 | 编辑器没开 / 刚退出 / MCP 总开关被关 | 启动编辑器；状态栏若显示「MCP 已停用」→ 设置菜单打开 Enable MCP |
| `E_BUSY` 一直来 | 人开着模态窗体 / UI 正忙 / 管道被另一条连接占着 | 等人用完再试，别当成「编辑器没开」 |
| 画上去了但看不见 | 画到了**非当前帧** | `meta.frame` 是当前帧；`frame{action:"set"}` 切过去再看 |
| `raw` 读回整行错位 | 按画布宽（或 256）换算 `indices` | 行宽 = 请求的 `w`：`x = region.x + i % w` |
| `batch` 里某个 op「没画上」 | 画的颜色同底色 / 几何退化 / 被后续 op 覆盖 | 开 `reportOps:true` 看 `ops[i].changed`（0 = 一个像素都没变） |
| 读资源报没资源 | 资源缓存是单槽，只有 `mosaic_view` 会填 | 先 `mosaic_view` 拿 `resource_uri` |
| 预览图有斜面阴影 | 看的是屏幕截图 | `mosaic_view` 输出是 flat 展开，不含任何界面装饰 |

---

## 6. 明确不提供

| 不提供 | 替代 |
|---|---|
| `mosaic_status` | 每个工具都带 `meta`；`mosaic_doc` 兼作身份/状态探针 |
| 帧重排（`move`） | 撤销不恢复帧序，故不提供 |
| 修改帧窗口 | 未暴露（`window` 只读） |
| 图片 / GIF 导入 | 走 GUI 菜单；MCP 只能打开 `.pxp` |
| 文本工具 / 历史 / 等待变更 / diff | 超出接口范围 |

---

## 7. 工具速查

| 工具 | 作用 | 撤销 | 必填参数 |
|---|---|:--:|---|
| `mosaic_view` | 看画布（flat PNG / ascii） | ❌ | — |
| `mosaic_draw` | 画图元（10 种 verb） | 1 次 | `verb`, `color`, `paths` |
| `mosaic_fill` | 洪水填充（四邻域） | 1 次 | `x`, `y`, `color` |
| `mosaic_pixel_get` | 读 ≤256 个点的索引（只读当前帧） | ❌ | `points` |
| `mosaic_doc` | 身份 / 状态（路径在 `meta.path`） | ❌ | — |
| `mosaic_open` | 打开 `.pxp`（替换文档） | ❌ | `path` |
| `mosaic_save` | 存 `.pxp` 或导出 `.pxa`/`.gif`/`.png`/`.bmp` | ❌ | `path` |
| `mosaic_new` | 新建空白画布（替换文档） | ❌ | `width`, `height`, `depth` |
| `mosaic_palette` | 调色板读写 | `set` 时 1 次 | `action` |
| `mosaic_undo` | 撤销 / 重做（当前帧） | — | `action` |
| `mosaic_frame` | 帧列举 / 切帧 / 增删 / 延时 / 合成模式 / 循环 | 视动作 | `action` |
| `mosaic_region_get` | 区域统计 / 指纹比对（自检） | ❌ | — |
| `mosaic_batch` | 原子批绘图（`draw`/`fill`/`copy`/`write`） | 单帧 1 次；跨帧每帧 1 次 | `ops` |
