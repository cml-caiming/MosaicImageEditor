# .pxa 像素动画文件格式 — 第三方解码技术规格

> 适用版本：Mosaic Image Editor **1.1.0.5** ｜ 读者：**第三方解码器作者**（任何语言 / 平台）
> 本文只描述**文件格式与播放语义**：任何语言/平台的解码器都能在只读本文的情况下正确解码 `.pxa`。
> 文中示例与常量均为格式契约本身（字节序、位打包、偏移、校验规则），可直接照抄使用。

---

## 1. 概述

`.pxa`(Pixel Animation)是本编辑器原生支持的**多帧像素动画发布/交换格式**,由帧序列、调色板、合成窗口信息组成。设计目标:

- **直读友好**:固定头部 + 显式帧偏移表,支持随机抽帧 `fseek(offsets[i])`,只需申请单帧缓冲。
- **位压缩**:1/2/4 位深度按 MSB 优先打包到字节流,降低磁盘体积;8 位不压缩(字节对齐,位打包收益为负)。
- **窗口增量**:每帧只存 `window_w × window_h` 子区域,窗口外保留上一帧内容(支持透明叠加动画)。
- **大端独立性**:全部字段使用**小端序**(与 x86/ARM 默认字节序一致)。

### 1.1 典型用例

| 场景 | 解码端使用 |
|------|-----------|
| 嵌入式 LCD 显示 | 单帧 `fseek + fread`,MCU 内存仅需 `W*H/depth` 字节 |
| Web 播放器 | 抽帧 + JS Canvas 渲染 `palette[index]` |
| GIF 转码器 | 读全帧 → 量化 → 调色板重映射 → 写 GIF |
| 校验/编辑器 | 写回需严格遵守布局与对齐规则 |

### 1.2 字段宽度与对齐约定

- 整数字段(除 `magic` 外)使用**小端序**(little-endian)。
- 所有 `#pragma pack(1)` 等价(无内填充)。
- 帧描述符结束后追加 0~3 字节 `0x00`,使下一帧描述符从 **4 字节对齐**的偏移开始(见 §7.3)。

---

## 2. 术语表

| 术语 | 含义 |
|------|------|
| 画布 (Canvas) | `canvas_w × canvas_h` 像素的逻辑画布,作为帧合成的最终目的地 |
| 帧 (Frame) | 一次显示周期内的单张图像,由 `window` 子区域与合成参数描述 |
| 窗口 (Window) | 帧实际存储/绘制的子区域 `(window_x, window_y, window_w, window_h)`,画布坐标系 |
| 深度 (Depth) | 每像素位数:1 / 2 / 4 / 8 / 16(16 = RGB565 / RGB555 输出模式) |
| 调色板 (Palette) | Indexed 与 RGB 模式下每像素值为调色板索引(0..255),调色板 256 × 4 字节 |
| 灰阶 (Grayscale) | color_mode=0,无调色板,像素值即亮度(0..255) |
| RGB 输出 (RGB Output) | color_mode=2,帧数据为 16bpp 像素(RGB565 或 RGB555);调色板**仍落盘**,用于回读时把 16 位字还原成调色板槽位 |
| 位打包 (Bit-Packing) | 1/2/4 位深度将多个像素塞进 1 字节,MSB 优先(高位先放) |
| 合成模式 (Composite Mode) | 帧与画布的混合规则(擦除位 + 操作位) |

---

## 3. 文件总体布局

```
+-----------------------------------------------+
|  HEADER  (32 bytes, 固定)                      |  offset 0x00
+-----------------------------------------------+
|  PALETTE  (Indexed 与 RGB 模式)                |  offset 0x20
|  256 × uint8_t[4] = 1024 bytes                 |
+-----------------------------------------------+
|  FRAME_OFFSET_TABLE                            |  offset 0x20 (Grayscale)
|  uint32_t × frame_count                        |  offset 0x420 (Indexed / RGB)
+-----------------------------------------------+
|  FRAME_0                                       |
|    ├── 11 bytes 描述头 (PxaFrameHeader)        |
|    ├── 位打包像素数据 (window_w × window_h)     |
|    └── 0..3 字节 padding (4 字节对齐)          |
+-----------------------------------------------+
|  FRAME_1                                       |
|    └── ...                                      |
+-----------------------------------------------+
|  ...                                            |
+-----------------------------------------------+
```

### 3.1 关键偏移速查

| 字段 | Indexed 偏移 | RGB 偏移 | Grayscale 偏移 |
|------|:------------:|:--------:|:--------------:|
| HEADER 起始 | `0x000` | `0x000` | `0x000` |
| HEADER 结束 | `0x020` | `0x020` | `0x020` |
| PALETTE 起始 | `0x020` | `0x020` | (不存在) |
| PALETTE 结束 | `0x420` | `0x420` | (不存在) |
| FRAME_OFFSET_TABLE 起始 | `0x420` | `0x420` | `0x020` |
| FRAME_0 起始 | `offsets[0]` | `offsets[0]` | `offsets[0]` |

> ⚠️ 帧偏移表的字节长度 = `frame_count × 4`,但解码器应通过**读完 HEADER 后读调色板块(若存在)再读 4×frame_count 字节**得到偏移表,而非依赖固定偏移。

---

## 4. HEADER(32 字节,小端序)

> 全部字段在文件中按以下偏移与大小排列;未对齐字段间**无填充**(`#pragma pack(1)` 等价)。

| 偏移 | 大小 | 字段 | 类型 | 必填 | 默认 | 含义与约束 |
|:----:|:---:|------|------|:----:|:----:|------------|
| `0x00` | 4 | `magic` | `uint32_t` | ✓ | — | **魔数**,固定 `0x50584130`。解码器必须先校验,否则拒绝加载。字节序:小端(LSB 在前),即文件首字节为 `0x30`,次字节 `0x41`,再 `0x58`,再 `0x50` → ASCII 序列 `"0AXP"`。 |
| `0x04` | 2 | `version` | `uint16_t` | ✓ | — | 格式版本 `0x0100`。**主版本号** = `version >> 8`(= `0x01`),**次版本号** = `version & 0xFF`(= `0x00`)。实现仅校验主版本号,次版本号兼容忽略。 |
| `0x06` | 2 | `canvas_w` | `uint16_t` | ✓ | — | 画布宽度 ∈ `[8, 256]`。 |
| `0x08` | 2 | `canvas_h` | `uint16_t` | ✓ | — | 画布高度 ∈ `[8, 256]`。 |
| `0x0A` | 1 | `depth` | `uint8_t` | ✓ | — | 像素位深,`∈ {1, 2, 4, 8, 16}`。**16 = 文件帧位深为 16bpp**,此时必须 `color_mode = 2`。 |
| `0x0B` | 1 | `color_mode` | `uint8_t` | ✓ | — | `0` = Grayscale(无调色板),`1` = Indexed(必须存在 PALETTE),`2` = RGB(RGB565/RGB555 输出模式,**必须存在 PALETTE**,且 `depth` 必须 = 16)。 |
| `0x0C` | 2 | `frame_count` | `uint16_t` | ✓ | — | 总帧数 ≥ 1。 |
| `0x0E` | 1 | `flags` | `uint8_t` | ✓ | — | 见 §4.1 位定义。 |
| `0x0F` | 17 | `reserved` | `uint8_t[17]` | — | 全 0 | 保留字段,当前实现写入全 0。解码器应**忽略内容**继续读取,不可视为错误。 |

> **C/C++ 等价结构(参考)**:
>
> ```c
> #pragma pack(push, 1)
> typedef struct {
>     uint32_t magic;        // 0x50584130
>     uint16_t version;      // 0x0100
>     uint16_t canvas_w;
>     uint16_t canvas_h;
>     uint8_t  depth;        // 1, 2, 4, 8, 16
>     uint8_t  color_mode;   // 0=Gray, 1=Indexed, 2=RGB
>     uint16_t frame_count;
>     uint8_t  flags;
>     uint8_t  reserved[17];
> } PxaHeader;
> #pragma pack(pop)
> ```
>
> `sizeof(PxaHeader) == 32`,与 `appconfig.h` 中常量无关。

### 4.1 `flags` 位定义

| 位 | 名称 | 含义 |
|:--:|------|------|
| Bit 0 | `LOOP` | 1 = 循环播放;0 = 单次播放 |
| Bit 1 | (保留) | 当前规范未定义;解码器应忽略 |
| Bit 2 | `RGB_FMT` | **仅 `color_mode = 2` 有效**:0 = RGB565,1 = RGB555。`color_mode ≠ 2` 时必须为 0。 |
| Bit 3 | `RGB555_LAYOUT` | **仅 `color_mode = 2` 且 `RGB_FMT = 1` 有效**:0 = MSB-first(bit15..11=R5),1 = Windows DIB(bit14..10=R5)。其余情况必须为 0。 |
| Bit 4..7 | (保留) | 当前规范未定义;解码器应忽略 |

> **注意**:本字段与 `[ref]` 文档中描述的"Bit1:行填充标记, Bit2:GIF循环"**不一致**(以代码为准)。
> 📌 **Bit2 的语义已被改写**:旧文档说的 "Bit2:GIF循环" **从未被代码实现**;现在的 Bit2 是 `RGB_FMT`,与 Bit0(LOOP)无关。

### 4.2 校验顺序(解码器必做)

```
1. magic == 0x50584130          (字节序:小端)
2. (version >> 8) == 0x01       (主版本号)
3. canvas_w ∈ [8, 256]
4. canvas_h ∈ [8, 256]
5. depth ∈ {1, 2, 4, 8, 16}
6. color_mode ∈ {0, 1, 2}
7. (depth == 16) ⟺ (color_mode == 2)      // 两者必须同时成立
8. frame_count ≥ 1
```

任何一项不满足即**拒绝加载**(返回 `nullptr` / 抛异常 / `false`)。

---

## 5. PALETTE(Indexed 与 RGB,color_mode ∈ {1, 2} 时存在)

| 偏移 | 大小 | 字段 | 类型 | 含义 |
|:----:|:---:|------|------|------|
| `0x20` | 1024 | `palette[256]` | `uint8_t[256][4]` | 256 项调色板,**每项 4 字节 BGRA** |

### 5.1 单项布局(4 字节,小端写入)

| 字节 | 名称 | 含义 |
|:----:|------|------|
| `byte[0]` | `B` | 蓝色通道 `0..255` |
| `byte[1]` | `G` | 绿色通道 `0..255` |
| `byte[2]` | `R` | 红色通道 `0..255` |
| `byte[3]` | `_pad` / `A` | 写入时固定 `0x00`;读取时**强制 `0xFF` (不透明)**(实现不保留 alpha) |

> **关键事实**:
>
> 1. **磁盘字节序为 BGR**(不是 RGB、不是 RGBA)。
> 2. 写入端将 alpha 固定为 `0x00`,**alpha 不承载语义**;读取端将 alpha 强制覆盖为 `0xFF`。
> 3. 解码器重建颜色时: `(R, G, B) = (entry[2], entry[1], entry[0])`,`A = 0xFF`(不透明)。

### 5.2 注意事项

- **始终读写全部 256 色**,与 `depth` 无关(不根据 `depth` 精简调色板大小)。
- 只有 Grayscale 模式(`color_mode = 0`)代码**不会**写入调色板块;解码器需根据 `color_mode` 判断是否存在。
- Grayscale 模式下,帧偏移表从 `0x20` 开始;Indexed 与 RGB 模式下从 `0x420` 开始。
- 📌 **RGB 模式(`color_mode = 2`)同样写 PALETTE** —— 帧数据里是 16bpp **颜色**而非索引,
  读回时必须靠 PALETTE 才能把 16 位字还原成调色板槽位(见 §8.6)。

---

## 6. FRAME_OFFSET_TABLE

紧接 HEADER(Grayscale)或 PALETTE(Indexed / RGB)之后,连续排列。

| 字段 | 类型 | 数量 | 含义 |
|------|------|:---:|------|
| `offsets[i]` | `uint32_t` | `frame_count` | 从**文件起始**(即 HEADER 首字节)到第 `i` 帧描述符首字节的字节偏移 |

- 字节序:小端。
- `offsets[0]` 通常 > HEADER+调色板+偏移表总长,但**无强制约束**(理论上可为零,但通常 ≥ `frame_count × 4 + 32 + 1024`)。
- 解码器必须用 `offsets[i]` 作为帧随机访问入口;**严禁**按固定 stride 步进(因为每帧大小不同,见 §7.3)。

### 6.1 写入策略(参考)

写入端采用两遍法:

```
1. 计算每帧大小 → 构建完整偏移表(内存)
2. 写入 HEADER → PALETTE(若需要) → 偏移表 → 各帧数据
```

或单遍 + `fseek` 回填:

```
fwrite(HEADER);
fwrite(占位偏移表 4×frame_count 字节);   // 填 0,记下位置
for i = 0 .. frame_count-1:
    offsets[i] = ftell;
    fwrite(帧描述符);
    fwrite(像素数据);
    fwrite(padding);
fseek(偏移表起始位置, SEEK_SET);
fwrite(填充后的偏移表);
```

---

## 7. FRAME_DESCRIPTOR(每帧)

每帧 = **11 字节描述头** + **变长位打包像素数据** + **0..3 字节对齐 padding**。

### 7.1 描述头布局(11 字节,`#pragma pack(1)`)

| 偏移(帧内) | 大小 | 字段 | 类型 | 含义 |
|:----------:|:---:|------|------|------|
| `0x00` | 2 | `delay_ms` | `uint16_t` | 帧延迟,毫秒;取值范围由应用层决定(编辑器默认 **100**,UI 上限 **9000**,播放时按 `MIN_FRAME_DELAY_MS` = **10** 兜底),解码器应原样保留并由调用方决定 |
| `0x02` | 2 | `window_x` | `uint16_t` | 窗口左上角 X(画布坐标系),须满足 `window_x + window_w ≤ canvas_w` |
| `0x04` | 2 | `window_y` | `uint16_t` | 窗口左上角 Y(画布坐标系),须满足 `window_y + window_h ≤ canvas_h` |
| `0x06` | 2 | `window_w` | `uint16_t` | 窗口宽度(像素),须 `1 ≤ window_w ≤ canvas_w` |
| `0x08` | 2 | `window_h` | `uint16_t` | 窗口高度(像素),须 `1 ≤ window_h ≤ canvas_h` |

> ⚠️ **窗口越界是硬失败,不是「通常满足」**：编辑器**写**盘时若任一窗口越界 → **整个保存失败**；**读**盘时若某帧窗口越界 → **该帧读取失败**(,文件本身仍能打开)。
> 判定共 4 条：`window_w ≤ canvas_w`、`window_h ≤ canvas_h`、`window_x + window_w ≤ canvas_w`、`window_y + window_h ≤ canvas_h`(相加用 uint32 防溢出)。
| `0x0A` | 1 | `composite_mode` | `uint8_t` | 合成模式,见 §7.2 |

> **C/C++ 等价结构**:
>
> ```c
> #pragma pack(push, 1)
> typedef struct {
>     uint16_t delay_ms;
>     uint16_t window_x;
>     uint16_t window_y;
>     uint16_t window_w;
>     uint16_t window_h;
>     uint8_t  composite_mode;
> } PxaFrameHeader;            // sizeof == 11
> #pragma pack(pop)
> ```

### 7.2 `composite_mode` 位定义

| 位 | 名称 | 取值 | 含义 |
|:--:|------|:---:|------|
| Bit 0 | `ERASE` | `0`/`1` | 1 = 绘制本帧前**全画布清零**(`memset(dst, 0, FRAME_BYTES)`);0 = 不清 |
| Bits 1-2 | `OP` | `00`/`01`/`10`/`11` | 合成操作(在窗口范围内逐像素):`00`=覆盖(Overwrite),`01`=XOR,`10`=AND,`11`=OR |
| Bits 3-7 | (保留) | `0` | 当前未定义,解码器应忽略 |

#### 常用组合速查

| 值 | 擦除 | 操作 | 语义 |
|:--:|:---:|:---:|------|
| `0x00` | 否 | 覆盖 | 后续叠加帧(默认) |
| `0x01` | **是** | 覆盖 | **首帧 / 清除后开始**(编辑器的标准用法) |
| `0x02` | 否 | XOR | |
| `0x03` | **是** | XOR | |
| `0x04` | 否 | AND | |
| `0x05` | **是** | AND | |
| `0x06` | 否 | OR | |
| `0x07` | **是** | OR | |

> **写入约定(当前实现)**:**编辑器不依赖首帧的 `ERASE` 位** —— 新建文档的首帧与后续帧一律写 `0x00`(覆盖、不清屏);GIF 导入的帧也一律 `PXA_OVERWRITE`(= `0x00`)。只有用户显式点「清屏」才会把当前帧置为带 `ERASE` 的值。
> → **解码器不应假定首帧 `ERASE=1`**;播放时若需要「从空白画布开始」,由**播放器**自己把画布清零即可(见 §9.2)。

### 7.3 像素数据大小

设 `stride = pxaStride(window_w, depth)`(见 §8),则:

```
frame_data_size = stride × window_h
frame_total_size = 11 + frame_data_size
padding          = (4 - frame_total_size % 4) % 4    // 使下一帧 4 字节对齐
```

`padding ∈ {0, 1, 2, 3}`,填充字节固定为 `0x00`。

### 7.4 帧总长公式

```
bytes_per_frame = 11 + (window_h × stride) + padding
```

例如 `depth=8, window_w=16, window_h=16` → `stride=16`,`frame_data_size=256`,`frame_total=267`,`padding=1` → **每帧 268 字节**。

---

## 8. 像素位打包规则

### 8.1 行 stride 公式

| `depth` | `stride(w)` | 备注 |
|:-------:|:-----------:|------|
| `1` | `ceil(w / 8) = (w + 7) / 8` | 8 像素 / 字节 |
| `2` | `ceil(w / 4) = (w + 3) / 4` | 4 像素 / 字节 |
| `4` | `ceil(w / 2) = (w + 1) / 2` | 2 像素 / 字节 |
| `8` | `w` | **不压缩**,1 像素 / 字节 |
| `16` | `w × 2` | **不压缩**,1 像素 / 2 字节(小端 16 位字) |

### 8.2 MSB 优先(大端风格)打包

每个字节内从 **MSB(高位)到 LSB(低位)** 依次放像素,像素内部也是 MSB 优先:

| `depth` | 字节内布局(MSB → LSB) |
|:-------:|------------------------|
| 1 | `[P0][P1][P2][P3][P4][P5][P6][P7]` |
| 2 | `[P0:2b][P1:2b][P2:2b][P3:2b]` |
| 4 | `[P0:4b][P1:4b]` |
| 8 | `[P0:8b]` |

行末不足整字节的部分**用 0 填充**(写入端已用 `memset(dst, 0, stride)` 初始化)。

### 8.3 位移公式

对像素索引 `i ∈ [0, w)`:

```
pixels_per_byte = 8 / depth
byte_idx        = i / pixels_per_byte
bit_shift       = (pixels_per_byte - 1 - (i % pixels_per_byte)) × depth
src_mask        = (1 << depth) - 1
```

- **打包**(写入):`dst[byte_idx] |= (src[i] & src_mask) << bit_shift`
- **解包**(读取):`dst[i] = (src[byte_idx] >> bit_shift) & src_mask`

### 8.4 完整位打包示例

#### 示例 A:`depth=1`,`window_w=10`,行 stride = 2 字节

源像素(10 个):
```
P0 P1 P2 P3 P4 P5 P6 P7 P8 P9
```

打包结果(2 字节):
```
byte0: P0 P1 P2 P3 P4 P5 P6 P7    (P0 在 bit7, P7 在 bit0)
byte1: P8 P9  0  0  0  0  0  0    (P8 在 bit7, P9 在 bit6, 低位填 0)
```

#### 示例 B:`depth=2`,`window_w=3`,行 stride = 1 字节

源像素:`P0 P1 P2`(每个 2 bit)

打包结果(1 字节):
```
byte0: [P0:2b][P1:2b][P2:2b][00:2b]
       bit7..6   bit5..4   bit3..2   bit1..0
```

#### 示例 C:`depth=4`,`window_w=5`,行 stride = 3 字节

源像素:`P0 P1 P2 P3 P4`(每个 4 bit)

打包结果(3 字节):
```
byte0: [P0:4b][P1:4b]      (P0 高 4 位,P1 低 4 位)
byte1: [P2:4b][P3:4b]
byte2: [P4:4b][0000]        (P4 高 4 位,低 4 位填 0)
```

### 8.5 隐式钳位

解包时通过掩码 `(1 << depth) - 1` 仅保留低 `depth` 位,**越界的高位会被静默截断**。解码器在写入画布前可选择自行钳位(`val &= mask`)以保持显式。

### 8.6 16bpp 帧数据(RGB 输出模式,`depth = 16`)

> 仅当 `color_mode = 2` 时适用。**16bpp 不做位打包**:每像素固定占 2 字节。

#### 8.6.1 行布局

每像素 2 字节,**小端**(低字节在前,与 HEADER 其余多字节字段一致):

```
byte[0] = word & 0xFF          // 低字节
byte[1] = (word >> 8) & 0xFF   // 高字节
```

`stride(w, 16) = w × 2`(最大 512 字节/行)。

#### 8.6.2 位域(`flags` 的 `RGB_FMT` / `RGB555_LAYOUT` 决定)

| 格式 | R | G | B | 空闲位 |
|------|---|---|---|---|
| **RGB565** | **bit15..11**(5bit) | **bit10..5**(6bit) | **bit4..0**(5bit) | 无(16 位全用满) |
| **RGB555 MSB-first** | bit15..11(5bit) | bit10..6(5bit) | bit5..1(5bit) | bit0 = 0 |
| **RGB555 Windows DIB** | bit14..10(5bit) | bit9..5(5bit) | bit4..0(5bit) | bit15 = 0 |

> 📌 **RGB565 的 G 是 6bit 且独占 bit10..5** —— 与 RGB555 的 G(bit10..6)只差**一个 bit 的偏移**,写解码器时极易照抄 RGB555 那行 → **整体色偏且不报错**。
> 权威出处:`Rgb565.cpp:96` 打包 `(r5 << 11) | (g6 << 5) | b5`;`:113-115` 解包 `(w >> 11) & 0x1F` / `(w >> 5) & 0x3F` / `w & 0x1F`。
> 示例:纯蓝 = `0x001F`、纯绿 = `0x07E0`、纯红 = `0xF800`、纯白 = `0xFFFF`。
>
> ⚠️ **RGB555 有两套业界位域**,选错会整体色偏甚至红蓝互换且**不报错** →
> 必须按 `flags` 的 Bit2/Bit3 判定,不可隐式约定。

#### 8.6.3 量化规则(**格式的一部分**)

源像素是**调色板索引**,落盘时按下列步骤转成 16 位字:

```
1. color = palette[index]                      # BGRA 字节序,取 R/G/B
2. 量化到目标网格(四舍五入到最近):
     5bit: idx = (v * 31 + 127) / 255 ; out = (idx<<3) | (idx>>2)
     6bit: idx = (v * 63 + 127) / 255 ; out = (idx<<2) | (idx>>4)
   (RGB565:R/B 用 5bit,G 用 6bit;RGB555:三通道全 5bit)
3. 按 §8.6.2 的位域组装 16 位字
```

- **四舍五入而非截断**:截断误差上界 7/255 且恒偏暗;四舍五入为 4/255 且无偏。
- **网格点不动**:网格上的色量化误差恒为 0。
- 量化**后**才打包 → 16bpp 文件中的颜色一定落在目标网格上。

#### 8.6.4 读回(还原索引)

16 位字 → 颜色 → 调色板槽位:

```
1. 按 §8.6.2 解出 16 位字对应的 RGB(结果必为网格色)
2. 在 PALETTE 中查找"量化后等于该网格色"的槽位
3. 找不到时退到"量化后颜色最近"的槽位(外来文件 / 调色板未钳网格)
```

> 📌 **索引 100% 精确可还原的充要条件**:调色板全部落在目标网格上 **且** 无重复项。
> 调色板未钳到网格时,颜色仍精确一致,但**索引**可能落到同色槽位的另一个位置。

---

## 9. 合成播放语义

### 9.1 标准播放算法

```
canvas := zeroed buffer of size CANVAS_STRIDE × CANVAS_H (= 256 × 256 = 65536 字节;索引模式 uint8_t,索引 = 调色板项)
         # ⚠️ 行跨度恒为 CANVAS_STRIDE(256),不是 canvas_w —— 与编辑器一致
for i = 0 .. frame_count-1:
    desc := read FRAME_DESCRIPTOR(i)

    if desc.composite_mode & 0x01 == 1:        # ERASE
        memset(canvas, 0, canvas_bytes)

    op := (desc.composite_mode >> 1) & 0x03    # 0=overwrite, 1=xor, 2=and, 3=or

    pixels := unpack(FRAME_DATA(i), desc.window_w, desc.window_h, depth)

    for y = 0 .. desc.window_h-1:
        for x = 0 .. desc.window_w-1:
            cx := desc.window_x + x
            cy := desc.window_y + y
            src := pixels[y × desc.window_w + x]
            dst := &canvas[cy × CANVAS_STRIDE + cx]
            switch op:
                case 0: *dst = src;      break      # overwrite
                case 1: *dst ^= src;     break      # xor
                case 2: *dst &= src;     break      # and
                case 3: *dst |= src;     break      # or

    display(canvas)                               # 持续 delay_ms 毫秒
```

### 9.2 关键行为

- **画布起始为零**,不依赖首帧的 `ERASE` 位。📌 **当前实现也不会给首帧置 `ERASE=1`** —— 新建文档与 GIF 导入的帧一律是 `0x00`(覆盖)(编辑器实现、`:83`;编辑器实现、`:2869`),只有用户显式「清屏」才会置 `ERASE`。→ **解码器/播放器都应自己把画布清零**,不要依赖文件里的 `ERASE` 位。
- **窗口外区域保留上一帧内容**,仅修改 `(window_x..window_x+window_w) × (window_y..window_y+window_h)` 范围。
- **像素值含义**:
  - Indexed:索引值,经调色板查表得到 RGBA。
  - Grayscale:亮度值,通常直接使用 `R = G = B = value`,A = 255。
  - RGB(`depth = 16`):帧数据是 16bpp **颜色**,见 §8.6;合成到画布时需按 §8.6.4 还原为索引。

### 9.3 显示与调色板映射

最终颜色:

```
if color_mode == 1:    # Indexed
    color = palette[canvas_pixel_index]    # 4 字节,RGBA,见 §5
elif color_mode == 2:  # RGB (depth = 16)
    color = word_to_rgb(canvas_pixel_word, flags)   # 见 §8.6.2
else:                 # Grayscale
    color = (canvas_pixel_index, canvas_pixel_index, canvas_pixel_index, 255)
```

---

## 10. 验证规则清单(解码器)

> 实现完整解码器时,**至少**执行以下校验。建议在打开文件第一步完成头部校验,读完头部后再校验帧级一致性。

### 10.1 头部校验

> 下表是**推荐给解码器**的校验项;**编辑器自己执行到哪几条**见 **§10.3**。

| 项 | 规则 | 失败处理 |
|----|------|---------|
| `magic` | `== 0x50584130` | 拒绝 |
| `version >> 8` | `== 0x01` | 拒绝 |
| `canvas_w/h` | `∈ [8, 256]` | 拒绝 |
| `depth` | `∈ {1, 2, 4, 8, 16}` | 拒绝 |
| `color_mode` | `∈ {0, 1, 2}` | 拒绝 |
| `depth` 与 `color_mode` | `(depth==16) ⟺ (color_mode==2)` | 拒绝 |
| `frame_count` | `≥ 1`(写盘侧另有上限 `≤ 0xFFFF`) | 拒绝 |
| 文件总长 | `≥ 32 + (color_mode ∈ {1,2} ? 1024 : 0) + 4×frame_count + 1` | 拒绝 ⚠️ **编辑器不校验这条** |
| `reserved` | (忽略) | — |

### 10.2 帧校验(建议)

| 项 | 规则 | 失败处理 |
|----|------|---------|
| `window_w` / `window_h` | `≤ canvas_w` / `≤ canvas_h` | **该帧失败**(编辑器行为,见 §10.3) |
| `window_x + window_w` | `≤ canvas_w` | **该帧失败** |
| `window_y + window_h` | `≤ canvas_h` | **该帧失败** |
| `frame_data_size` | `= pxaStride(window_w, depth) × window_h` | 拒绝(若 fread 短读) |
| `padding` | `∈ {0, 1, 2, 3}` 且全 0 | 接受(忽略内容) |
| `composite_mode` 高 5 位 | `== 0` | 警告 / 忽略 |

> 📌 **窗口越界是硬失败**:编辑器**写**盘时任一窗口越界 → **整个保存失败**;**读**盘时某帧窗口越界 → **该帧读取失败**(文件本身仍能打开)。
> 📌 **输出缓冲区必须按 `CANVAS_STRIDE`(恒 = 256)行跨度分配,不是 `canvas_w`** —— 编辑器用 `outBuffer + (window_y + y) * CANVAS_STRIDE + window_x` 寻址。按 `canvas_w × canvas_h` 分配会在画布 < 256 时**堆越界写**。

### 10.3 校验执行现状（**编辑器实际行为，供参考**）

> ⚠️ 本节是**当前实现的事实**,不是建议。第三方解码器若照 §10.1 / §10.2 全量校验,**会比编辑器更早拒文件** —— 两条路都合法,但行为会不一致,需知情。

| 校验项 | 编辑器是否执行 | 触发时的实际行为 |
|---|:--:|---|
| `magic` | ✅ 执行 | 拒绝打开文件 |
| `version >> 8` | ✅ 执行 | 拒绝打开文件 |
| `canvas_w/h ∈ [8,256]` | ✅ 执行 | 拒绝打开文件 |
| `depth ∈ {1,2,4,8,16}` | ✅ 执行 | 拒绝打开文件 |
| `color_mode ≤ 2` | ✅ 执行 | 拒绝打开文件 |
| `(depth==16) ⟺ (color_mode==2)` | ✅ 执行 | 拒绝打开文件 |
| `frame_count ≥ 1` | ✅ 执行 | 拒绝打开文件 |
| **文件总长** | ❌ **不执行** | ⚠️ 截断文件**不会在打开阶段被拒**;后果推迟到读帧时表现为短读失败 |
| `reserved` 内容 | 不校验 | 按「忽略」处理 |
| **`flags` 的未定义位**(Bit1、Bit4..7) | ❌ **不校验** | 未知位被静默忽略,不会报错 |
| 帧窗口 4 条边界 | ✅ 执行 | **该帧**读取失败;写盘时则**整个保存失败** |
| 帧像素短读 | ✅ 执行 | 该帧读取失败 |
| `padding` 内容 | ❌ 不校验 | 跳过,不比对是否为 0 |
| `composite_mode` 高 5 位 | ❌ 不校验 | 保留值,播放时高位被忽略 |

> 📌 **写盘侧的额外约束**(只影响编辑器,不约束第三方读):`frame_count` 上限 `0xFFFF`、`depth`/`color_mode` 合法值同上、窗口必须落在画布内。

---

## 11. 完整读取算法(伪代码)

```
function load_pxa(path) -> (info, frames[frame_count]):
    fp = open(path, "rb")
    if not fp: return None

    hdr = read(fp, 32)
    if hdr.magic != 0x50584130: return None
    if (hdr.version >> 8) != 0x01: return None
    if not (8 <= hdr.canvas_w <= 256): return None
    if not (8 <= hdr.canvas_h <= 256): return None
    if hdr.depth not in {1,2,4,8,16}: return None
    if hdr.color_mode not in {0,1,2}: return None
    if (hdr.depth == 16) != (hdr.color_mode == 2): return None
    if hdr.frame_count < 1: return None

    info.canvas_w = hdr.canvas_w
    info.canvas_h = hdr.canvas_h
    info.depth     = hdr.depth
    info.color_mode = hdr.color_mode
    info.frame_count = hdr.frame_count
    info.loop      = (hdr.flags & 0x01) != 0
    info.rgb_fmt    = (hdr.flags & 0x04) ? RGB555 : RGB565
    info.rgb_layout = (hdr.flags & 0x08) ? WIN_DIB : MSB_FIRST

    # PALETTE
    if hdr.color_mode in (1, 2):
        for i in 0..255:
            entry = read(fp, 4)
            info.palette[i] = entry[0] | (entry[1]<<8) | (entry[2]<<16) | (0xFF<<24)
            # 内存布局 = 0xAARRGGBB(byte0=B, byte1=G, byte2=R, byte3=A=0xFF)
            # ⚠️ 不是 0xBBGGRR —— 字节序是 BGR,红蓝别读反
    else:
        memset(info.palette, 0, sizeof info.palette)

    # FRAME_OFFSET_TABLE
    offsets = read_uint32_array(fp, hdr.frame_count)

    # FRAMES
    frames = []
    for i in 0 .. hdr.frame_count-1:
        seek(fp, offsets[i], SEEK_SET)
        fhdr = read(fp, 11)
        desc.delay_ms       = uint16_le(fhdr[0..1])
        desc.window_x       = uint16_le(fhdr[2..3])
        desc.window_y       = uint16_le(fhdr[4..5])
        desc.window_w       = uint16_le(fhdr[6..7])
        desc.window_h       = uint16_le(fhdr[8..9])
        desc.composite_mode = fhdr[10]

        stride = pxa_stride(desc.window_w, hdr.depth)
        # ⚠️ 缓冲区按 **CANVAS_STRIDE(恒 256)** 行跨度分配,**不是 canvas_w**
        #    (编辑器的帧缓冲恒为 256×256 = 65536 字节,见 §5 / `appconfig.h`)
        frame_buf = alloc_buffer(CANVAS_STRIDE * CANVAS_H)
        memset(frame_buf, 0, CANVAS_STRIDE * CANVAS_H)               # 窗口外清零

        for y in 0 .. desc.window_h-1:
            packed_row = read(fp, stride)
            if hdr.depth == 16:
                # 16bpp:16 位字 → 颜色 → 调色板槽位(见 §8.6.4)
                unpack_row16(packed_row, desc.window_w, info.rgb_fmt, info.rgb_layout,
                              info.palette, frame_buf + (desc.window_y + y) * CANVAS_STRIDE + desc.window_x)
            else:
                unpack_row(packed_row, desc.window_w, hdr.depth, frame_buf + (desc.window_y + y) * CANVAS_STRIDE + desc.window_x)

        # 跳过 padding(0..3 字节)
        cur = ftell(fp)
        next_offset = offsets[i+1] if i+1 < frame_count else (file_size)
        pad = (next_offset - cur)
        if pad > 0: seek(fp, pad, SEEK_CUR)

        frames.append((frame_buf, desc))

    close(fp)
    return (info, frames)
```

### 11.1 位解包函数(`unpack_row`)

```
function unpack_row(src, w, depth, dst_row):
    if depth == 8:
        memcpy(dst_row, src, w)
        return

    pixels_per_byte = 8 / depth
    mask = (1 << depth) - 1

    for i in 0 .. w-1:
        byte_idx = i / pixels_per_byte
        bit_shift = (pixels_per_byte - 1 - (i % pixels_per_byte)) * depth
        dst_row[i] = (src[byte_idx] >> bit_shift) & mask
```

---

## 12. 完整写入算法(伪代码)

```
function save_pxa(path, info, frames, descs) -> bool:
    fp = open(path, "wb")

    # 1. HEADER
    hdr.magic        = 0x50584130
    hdr.version      = 0x0100
    hdr.canvas_w     = info.canvas_w
    hdr.canvas_h     = info.canvas_h
    hdr.depth        = info.depth
    hdr.color_mode   = info.color_mode
    hdr.frame_count  = info.frame_count
    hdr.flags        = info.loop ? 0x01 : 0x00
    if info.color_mode == 2:                       # RGB 输出格式
        if info.rgb_fmt == RGB555: hdr.flags |= 0x04
        if info.rgb_fmt == RGB555 and info.rgb_layout == WIN_DIB: hdr.flags |= 0x08
    memset(hdr.reserved, 0, 17)
    write(fp, hdr, 32)

    # 2. PALETTE (Indexed 与 RGB)
    if info.color_mode in (1, 2):
        for i in 0..255:
            c = info.palette[i]
            entry = [c & 0xFF, (c>>8)&0xFF, (c>>16)&0xFF, 0]
            write(fp, entry, 4)

    # 3. 占位偏移表
    offset_table_pos = ftell(fp)
    for i in 0 .. info.frame_count-1:
        write(fp, uint32(0), 4)

    # 4. 帧数据
    offsets = []
    for i in 0 .. info.frame_count-1:
        offsets.append(ftell(fp))
        write_frame(fp, descs[i], frames[i], info.depth)
        # padding 自动由 write_frame 写入

    # 5. 回填偏移表
    seek(fp, offset_table_pos, SEEK_SET)
    for i in 0 .. info.frame_count-1:
        write(fp, uint32_le(offsets[i]), 4)

    close(fp)
    return true


function write_frame(fp, desc, src_canvas, depth):
    fhdr = struct.pack('<HHHHHB',
        desc.delay_ms,
        desc.window_x, desc.window_y,
        desc.window_w, desc.window_h,
        desc.composite_mode)
    write(fp, fhdr, 11)

    stride = pxa_stride(desc.window_w, depth)
    for y in 0 .. desc.window_h-1:
        row_src = src_canvas + (desc.window_y + y) * CANVAS_STRIDE + desc.window_x
        packed = pack_row(row_src, desc.window_w, depth)        # 返回 stride 字节
        write(fp, packed, stride)

    frame_size = 11 + stride * desc.window_h
    pad = (4 - frame_size % 4) % 4
    if pad > 0: write(fp, [0]*pad, pad)


function pack_row(src_row, w, depth) -> bytes:
    if depth == 8:
        return src_row[:w]

    pixels_per_byte = 8 / depth
    stride = (w + pixels_per_byte - 1) / pixels_per_byte
    dst = zero-filled buffer of stride bytes
    mask = (1 << depth) - 1

    for i in 0 .. w-1:
        byte_idx = i / pixels_per_byte
        bit_shift = (pixels_per_byte - 1 - (i % pixels_per_byte)) * depth
        dst[byte_idx] |= (src_row[i] & mask) << bit_shift
    return dst
```

---

## 13. 参考实现

### 13.1 C99 实现

```c
/* pxa_decode.h - 第三方解码器参考实现(C99) */
#ifndef PXA_DECODE_H
#define PXA_DECODE_H

#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define PXA_MIN_DIM 8
#define PXA_MAX_DIM 256
#define PXA_PALETTE_ENTRIES 256
#define PXA_HEADER_BYTES 32
#define PXA_PALETTE_BYTES 1024
#define PXA_FRAME_HEADER_BYTES 11

typedef struct {
    uint32_t magic;        /* 0x50584130 */
    uint16_t version;      /* 0x0100 */
    uint16_t canvas_w;
    uint16_t canvas_h;
    uint8_t  depth;        /* 1,2,4,8,16 */
    uint8_t  color_mode;   /* 0=Gray, 1=Indexed, 2=RGB(16bpp) */
    uint16_t frame_count;
    uint8_t  flags;
    uint8_t  reserved[17]; /* 占位到 32 字节 —— ⚠️ 不能省,否则 fread 会越界读 17 字节 */
} PxaHeader;               /* sizeof == 32 */

typedef struct {
    uint16_t delay_ms;
    uint16_t window_x, window_y, window_w, window_h;
    uint8_t  composite_mode;
} PxaFrameHeader;

typedef struct {
    uint32_t palette[256]; /* 0xAARRGGBB on memory (byte0=B, byte1=G, byte2=R, A=0xFF) */
    int      canvas_w, canvas_h;
    int      depth;
    int      color_mode;
    int      frame_count;
    int      loop;
} PxaInfo;

/* 行 stride:ceil(w / (8/depth));depth=16 时每像素 2 字节 */
static inline int pxa_stride(int w, int depth) {
    switch (depth) {
        case 1: return (w + 7) / 8;
        case 2: return (w + 3) / 4;
        case 4: return (w + 1) / 2;
        case 16: return w * 2;          /* 16bpp:每像素 2 字节(小端 16 位字) */
        default: return w;  /* depth == 8 */
    }
}

/* 解包一行到位图(画布坐标系,行跨度 = CANVAS_STRIDE,恒 256) */
static void pxa_unpack_row(const uint8_t* src, int w, int depth, uint8_t* dst) {
    if (depth == 8) {
        memcpy(dst, src, w);
        return;
    }
    int ppb = 8 / depth;
    uint8_t mask = (uint8_t)((1u << depth) - 1u);
    for (int i = 0; i < w; i++) {
        int byte_idx = i / ppb;
        int bit_shift = (ppb - 1 - (i % ppb)) * depth;
        dst[i] = (uint8_t)((src[byte_idx] >> bit_shift) & mask);
    }
}

/* 打开并校验头部,返回 0 成功;失败返回非 0 */
static int pxa_open(FILE* fp, PxaInfo* info) {
    PxaHeader h;
    if (fread(&h, 1, 32, fp) != 32) return -1;
    if (h.magic != 0x50584130u) return -1;
    if ((h.version >> 8) != 0x01) return -1;
    if (h.canvas_w < PXA_MIN_DIM || h.canvas_w > PXA_MAX_DIM) return -1;
    if (h.canvas_h < PXA_MIN_DIM || h.canvas_h > PXA_MAX_DIM) return -1;
    if (h.depth != 1 && h.depth != 2 && h.depth != 4 && h.depth != 8 && h.depth != 16) return -1;
    if (h.color_mode > 2) return -1;
    if (((h.depth == 16) != (h.color_mode == 2))) return -1;  /* depth=16 ⟺ color_mode=2 */
    if (h.frame_count < 1) return -1;

    info->canvas_w = h.canvas_w;
    info->canvas_h = h.canvas_h;
    info->depth    = h.depth;
    info->color_mode = h.color_mode;
    info->frame_count = h.frame_count;
    info->loop = (h.flags & 0x01) != 0;

    if (h.color_mode >= 1) {   /* Indexed(1) 与 RGB(2) 都落调色板;仅 Grayscale(0) 不落 */
        for (int i = 0; i < 256; i++) {
            uint8_t e[4];
            if (fread(e, 1, 4, fp) != 4) return -1;
            /* 文件顺序 B,G,R,_pad */
            info->palette[i] = (uint32_t)e[0]
                             | ((uint32_t)e[1] << 8)
                             | ((uint32_t)e[2] << 16)
                             | (0xFFu << 24);
        }
    } else {
        memset(info->palette, 0, sizeof(info->palette));
    }

    /* 偏移表 */
    /* 此处省略将偏移表读入数组;调用方按需读取 */
    return 0;
}

/* 读取第 i 帧到 out_canvas(大小 = canvas_w × canvas_h 字节) */
static int pxa_read_frame(FILE* fp, const PxaInfo* info,
                          const uint32_t* offsets, int i,
                          uint8_t* out_canvas,
                          PxaFrameHeader* out_desc) {
    if (i < 0 || i >= info->frame_count) return -1;
    if (fseek(fp, (long)offsets[i], SEEK_SET) != 0) return -1;

    uint8_t fh[11];
    if (fread(fh, 1, 11, fp) != 11) return -1;
    out_desc->delay_ms       = (uint16_t)(fh[0] | (fh[1] << 8));
    out_desc->window_x       = (uint16_t)(fh[2] | (fh[3] << 8));
    out_desc->window_y       = (uint16_t)(fh[4] | (fh[5] << 8));
    out_desc->window_w       = (uint16_t)(fh[6] | (fh[7] << 8));
    out_desc->window_h       = (uint16_t)(fh[8] | (fh[9] << 8));
    out_desc->composite_mode = fh[10];

    int stride = pxa_stride(out_desc->window_w, info->depth);
    for (int y = 0; y < out_desc->window_h; y++) {
        uint8_t packed[256];
        if (fread(packed, 1, stride, fp) != (size_t)stride) return -1;
        uint8_t* dst = out_canvas
                     + (out_desc->window_y + y) * info->canvas_w
                     + out_desc->window_x;
        pxa_unpack_row(packed, out_desc->window_w, info->depth, dst);
    }
    return 0;
}

#endif /* PXA_DECODE_H */
```

### 13.2 Python 3 实现

```python
"""
pxa_decode.py - .pxa 第三方解码器参考实现(Python 3)
"""
import struct
from dataclasses import dataclass, field
from typing import List, Tuple, Optional


@dataclass
class PxaFrameHeader:
    delay_ms: int
    window_x: int
    window_y: int
    window_w: int
    window_h: int
    composite_mode: int


@dataclass
class PxaInfo:
    canvas_w: int
    canvas_h: int
    depth: int          # 1, 2, 4, 8
    color_mode: int     # 0=Gray, 1=Indexed
    frame_count: int
    loop: bool
    palette: List[Tuple[int, int, int, int]] = field(default_factory=list)  # RGBA


def pxa_stride(w: int, depth: int) -> int:
    if depth == 1:  return (w + 7) // 8
    if depth == 2:  return (w + 3) // 4
    if depth == 4:  return (w + 1) // 2
    if depth == 16: return w * 2      # 16bpp:每像素 2 字节(小端 16 位字)
    return w


def pxa_unpack_row(src: bytes, w: int, depth: int) -> bytearray:
    dst = bytearray(w)
    if depth == 8:
        dst[:w] = src[:w]
        return dst
    ppb = 8 // depth
    mask = (1 << depth) - 1
    for i in range(w):
        byte_idx = i // ppb
        bit_shift = (ppb - 1 - (i % ppb)) * depth
        dst[i] = (src[byte_idx] >> bit_shift) & mask
    return dst


def load_pxa(path: str) -> Tuple[PxaInfo, List[Tuple[bytes, PxaFrameHeader]]]:
    with open(path, "rb") as f:
        # 1) HEADER (32 bytes, little-endian)
        #    格式串:I(uint32) + H(uint16)×3 + B(uint8)×2 + H(uint16) + B(uint8) + 17s(reserved)
        magic, version, canvas_w, canvas_h, depth, color_mode, frame_count, flags, _reserved \
            = struct.unpack_from("<IHHHBBHB17s", f.read(32), 0)
        if magic != 0x50584130:
            raise ValueError("bad magic")
        if (version >> 8) != 1:
            raise ValueError("bad major version")
        if not (8 <= canvas_w <= 256 and 8 <= canvas_h <= 256):
            raise ValueError("canvas size out of range")
        if depth not in (1, 2, 4, 8, 16):
            raise ValueError("bad depth")
        if color_mode not in (0, 1, 2):
            raise ValueError("bad color_mode")
        if (depth == 16) != (color_mode == 2):
            raise ValueError("depth=16 must pair with color_mode=2")
        if frame_count < 1:
            raise ValueError("bad frame_count")

        info = PxaInfo(
            canvas_w=canvas_w, canvas_h=canvas_h, depth=depth,
            color_mode=color_mode, frame_count=frame_count,
            loop=bool(flags & 0x01), palette=[]
        )

        # 2) PALETTE (Indexed 与 RGB 都落盘,256 × 4 字节 BGRA;仅 Grayscale 不落)
        if color_mode in (1, 2):
            for _ in range(256):
                b, g, r, _pad = f.read(4)
                info.palette.append((r, g, b, 255))   # RGBA, A always 255
        else:
            info.palette = [(0, 0, 0, 255)] * 256

        # 3) FRAME_OFFSET_TABLE
        offsets = struct.unpack(f"<{frame_count}I", f.read(4 * frame_count))

        # 4) FRAMES
        frames: List[Tuple[bytes, PxaFrameHeader]] = []
        for idx in range(frame_count):
            f.seek(offsets[idx])
            fhdr_raw = f.read(11)
            (delay_ms, wx, wy, ww, wh, cm) = struct.unpack("<HHHHHB", fhdr_raw)
            desc = PxaFrameHeader(delay_ms, wx, wy, ww, wh, cm)

            stride = pxa_stride(ww, depth)
            # ⚠️ 帧缓冲按 CANVAS_STRIDE(恒 256)行跨度分配,不是 canvas_w
            canvas = bytearray(CANVAS_STRIDE * CANVAS_H)   # 全 0
            for y in range(wh):
                packed = f.read(stride)
                if len(packed) < stride:
                    raise ValueError("short read on frame data")
                row = pxa_unpack_row(packed, ww, depth)
                # 写入画布(wy+y, wx..wx+ww)
                base = (wy + y) * CANVAS_STRIDE + wx
                canvas[base:base + ww] = row
            frames.append((bytes(canvas), desc))

    return info, frames


def composite(canvas: bytearray, frame_pixels: bytes, desc: PxaFrameHeader,
              canvas_w: int) -> None:
    """标准合成:应用到 canvas(原地修改)。"""
    cm = desc.composite_mode
    if cm & 0x01:                  # ERASE
        for k in range(len(canvas)):
            canvas[k] = 0
    op = (cm >> 1) & 0x03
    for y in range(desc.window_h):
        base = (desc.window_y + y) * CANVAS_STRIDE + desc.window_x
        for x in range(desc.window_w):
            s = frame_pixels[base + x]      # frame_pixels 也是 CANVAS_STRIDE(256) 步长
            d_idx = base + x
            d = canvas[d_idx]
            if   op == 0: canvas[d_idx] = s
            elif op == 1: canvas[d_idx] = d ^ s
            elif op == 2: canvas[d_idx] = d & s
            elif op == 3: canvas[d_idx] = d | s
```

> ⚠️ **效率说明**:Python 参考实现的 `composite` 是逐元素循环,性能较差。生产代码应使用 `bytearray.translate` 或 NumPy 切片 + `np.bitwise_xor/and/or`,或将其编译为 C 扩展。

---

## 14. 版本兼容性与扩展策略

### 14.1 兼容性矩阵

| 主版本 | 次版本 | 解码器行为 | 说明 |
|:------:|:------:|------------|------|
| `0x01` | `0x00` | **完整支持**(本文档范围) | 当前唯一发布版 |
| `0x01` | `0x01..0xFF` | 完整支持,次版本差异应保持向后兼容 | 字段新增而非语义变更 |
| `0x02..` | 任意 | **拒绝加载** | 主版本不兼容时,不应尝试解码 |

### 14.2 未来扩展位

| 字段 | 当前用途 | 扩展方向 |
|------|---------|---------|
| `reserved[17]` | 全 0 | 引入新全局属性(如 ICC profile 偏移、压缩标志等) |
| `flags` 高 7 位 | 未定义 | 播放选项(暂停、淡入淡出、反向播放等) |
| `composite_mode` 高 5 位 | 未定义 | 扩展合成操作(乘法、α 混合、抖动等) |

> 解码器实现应**严格遵守**"主版本一致 → 全字段信任;主版本不一致 → 拒绝加载"策略;不要做宽松的回退。

---

## 15. 速查卡(Quick Reference)

### 15.1 文件字节序 / 大小端

- **全部字段小端**(LSB 在前)。
- 例:`canvas_w = 0x0100 (256)` → 文件字节 `00 01`。
- 例:`magic = 0x50584130` → 文件字节 `30 41 58 50` → ASCII `"0AXP"`。

### 15.2 关键常量

| 常量 | 值 |
|------|---|
| `MAGIC` | `0x50584130` |
| `VERSION` | `0x0100` |
| 画布宽高范围 | `[8, 256]` |
| 深度合法值 | `{1, 2, 4, 8, 16}`(`16` ⟺ `color_mode=2`) |
| `color_mode` 合法值 | `{0, 1, 2}` |
| `flags` 位 | Bit0 `LOOP` / Bit2 `RGB_FMT` / Bit3 `RGB555_LAYOUT` |
| 调色板落盘条件 | `color_mode ∈ {1, 2}`(Grayscale 不落) |
| RGB565 位域 | R=bit15..11, G=bit10..5, B=bit4..0 |
| 帧缓冲行跨度 | `CANVAS_STRIDE` = **256**(恒定,**不是** `canvas_w`) |
| 帧缓冲总字节 | `256 × 256` = **65536** |
| 调色板项数 | `256` |
| 单调色板字节数 | `4`(磁盘序 **B, G, R, _pad**;内存 `0xAARRGGBB`) |
| 调色板总字节 | `1024` |
| HEADER 字节数 | `32` |
| FRAME_HEADER 字节数 | `11` |
| 帧 4 字节对齐 padding | `0..3` |

### 15.3 偏移速查

| 字段 | 偏移 | 大小 |
|------|------|-----|
| `magic` | `0x00` | 4 |
| `version` | `0x04` | 2 |
| `canvas_w` | `0x06` | 2 |
| `canvas_h` | `0x08` | 2 |
| `depth` | `0x0A` | 1 |
| `color_mode` | `0x0B` | 1 |
| `frame_count` | `0x0C` | 2 |
| `flags` | `0x0E` | 1 |
| `reserved` | `0x0F..0x1F` | 17 |
| PALETTE (Indexed) | `0x20..0x41F` | 1024 |
| FRAME_OFFSET_TABLE (Indexed) | `0x420` | `4 × frame_count` |
| FRAME_OFFSET_TABLE (Gray) | `0x20` | `4 × frame_count` |
| FRAME_n 描述符 | `offsets[n]` | 11 |
| FRAME_n 像素数据 | `offsets[n] + 11` | `stride × window_h` |
| FRAME_n padding | 像素数据后 | `0..3` |

### 15.4 帧描述符内偏移

| 字段 | 偏移 | 大小 |
|------|------|-----|
| `delay_ms` | `0x00` | 2 |
| `window_x` | `0x02` | 2 |
| `window_y` | `0x04` | 2 |
| `window_w` | `0x06` | 2 |
| `window_h` | `0x08` | 2 |
| `composite_mode` | `0x0A` | 1 |
| 像素数据 | `0x0B` | `stride × window_h` |

---

## 16. 附录 A:16×16 灰阶示例文件 hex dump

> 一个最小有效 `.pxa` 文件:
>
> - 画布:16 × 16,深度 8 bit,Grayscale
> - 1 帧,覆盖全画布,delay 100 ms,ERASE+OVERWRITE

```
偏移     字节序列                          说明
──────────────────────────────────────────────────────────────────────
0x00     30 41 58 50                       magic = 0x50584130 (小端)
0x04     00 01                             version = 0x0100
0x06     10 00                             canvas_w = 16
0x08     10 00                             canvas_h = 16
0x0A     08                                depth = 8
0x0B     00                                color_mode = 0 (Grayscale)
0x0C     01 00                             frame_count = 1
0x0E     01                                flags = LOOP
0x0F..0x1F   17 × 00                       reserved
0x20     04 00 00 00                       offsets[0] = 0x24
0x24     64 00                             delay_ms = 100
0x26     00 00                             window_x = 0
0x28     00 00                             window_y = 0
0x2A     10 00                             window_w = 16
0x2C     10 00                             window_h = 16
0x2E     01                                composite_mode = ERASE|OVERWRITE
0x2F..0x12E  256 字节像素数据 (16×16)      stride = 16,需 1 字节 padding(267 mod 4 = 3 → padding = 1)
0x12F    00                                padding
──────────────────────────────────────────────────────────────────────
```

> 文件总长 = `0x130` = 304 字节。

---

## 17. 附录 B:索引模式 4 位深度示例

> - 画布:16 × 8,深度 4,Indexed
> - 调色板:索引 0 = 透明黑(0,0,0,255),索引 1 = 白(255,255,255,255)
> - 1 帧:左上 4×4 窗口,delay 50 ms,ERASE+OVERWRITE

```
偏移       字节序列                                  说明
────────────────────────────────────────────────────────────────────────────
0x00..0x1F (32 字节 HEADER)                          magic, version, canvas_w=16, canvas_h=8, depth=4, color_mode=1, frame_count=1, flags=01, reserved=17*00

0x20..0x41F  1024 字节 PALETTE                       256 × BGR+pad;此处:
                                                  entry[0] = 00 00 00 00  (索引 0)
                                                  entry[1] = FF FF FF 00  (索引 1)
                                                  其余 254 项 = 00 00 00 00

0x420..0x423   24 04 00 00                            offsets[0] = 0x0424

0x424      32 00                                      delay_ms = 50
0x426      00 00                                      window_x = 0
0x428      00 00                                      window_y = 0
0x42A      04 00                                      window_w = 4
0x42C      04 00                                      window_h = 4
0x42E      01                                         composite_mode = ERASE|OVERWRITE

# 像素数据:每行 4 像素 × 4 bit = 2 字节,共 4 行 = 8 字节
# 行 0..3:全索引 1(白色)→ 每字节 = 0x11 → 字节序列 11 11 11 11 11 11 11 11
0x42F..0x436  11 11 11 11 11 11 11 11

# padding:frame_size = 11 + 8 = 19,19 mod 4 = 3 → 1 字节 padding
0x437      00                                         padding
────────────────────────────────────────────────────────────────────────────
```

---

## 18. 附录 C:术语速查

| 英文 | 中文 |
|------|------|
| Big-endian | 大端 |
| Little-endian | 小端 |
| Bit-packing | 位打包 |
| Canvasing | 画布化 |
| Composite | 合成 |
| Depth (bit depth) | 位深 |
| Eager loading | 全量加载 |
| Frame | 帧 |
| Indexed color | 索引色 |
| Grayscale | 灰阶 |
| Loop | 循环 |
| Magic number | 魔数 |
| MSB-first | 高位优先 |
| Offset table | 偏移表 |
| Palette | 调色板 |
| Padding | 填充 |
| Pixel | 像素 |
| Random access | 随机访问 |
| RGBA | 红绿蓝 alpha |
| Stride | 行步长 |
| Window | 窗口(子区域) |

---

