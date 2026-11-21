# Mosaic Image Editor 使用说明

> 版本 **1.1.0.4** ｜ 绿色版（portable）：**解压即用**，不写注册表、不装服务、不需要卸载。
> 256×256 以内的索引色像素画 / 2D 帧动画编辑器（Windows 桌面）。

---


## 1. 快速开始

1. 把压缩包**解压到一个可写目录**（例如 `<解压目录>`；⚠️ **不要**放在 `C:\Program Files` 这类只读位置，见 §4）。
2. 双击 **`MI_Editor.exe`** —— 这是 **x64 主程序，绝大多数机器选它**。
3. 首次**启动并正常退出**后，会在 exe 同目录生成 `MI_Editor.ini`（设置文件）。
4. **打开文件**：菜单 **File → Open…**（`Ctrl+O`），或**直接把 `.pxa` / `.pxp` 拖到窗口上**（拖到画布区、工具栏、状态栏都行）。
   ⚠️ **一次只能拖一个文件** —— 同时拖多个会被整体忽略（什么都不做）；拖入不支持的格式（如 `.gif`）也只会忽略。

> 🌐 **界面默认是英文**：首次运行时还没有设置文件，程序按「默认英文」启动。
> 想要中文请在 **菜单 → 语言 → 中文** 里切换一次，之后会记住。
> （两个语言包都在包内；若某个语言包缺失，对应菜单项会置灰，界面回落到英文，见 §7。）

| 可执行文件 | 架构 | 什么时候用 |
|---|---|---|
| `MI_Editor.exe` | **x64** | 64 位 Windows 10 / 11 —— **默认推荐** |
| `MI_Editor_x86.exe` | **x86** | 32 位 Windows，或希望**开箱即用显卡加速**时（见 §3） |

> 两个 exe **共用**同一棵 `supra\` 资产树和同一份 `Language.*.ini` —— 不需要复制两份，也不要删掉其中一个。
> 两个 exe 的**设置文件是同一个**（`MI_Editor.ini`），所以先后运行不会互相覆盖偏好。

---

## 2. 前置依赖：VC++ 运行库

AI 引擎（`supra\SupraImgEngine.dll`）依赖 **Microsoft Visual C++ 2015-2022 Redistributable**。
Windows 10/11 通常已经装了；**没有装过**的干净机器会在**第一次用文生图**时报「无法加载引擎」。

| 主程序 | 需要安装 |
|---|---|
| `MI_Editor.exe`（x64） | [vc_redist.x64.exe](https://aka.ms/vs/17/release/vc_redist.x64.exe) |
| `MI_Editor_x86.exe`（x86） | [vc_redist.x86.exe](https://aka.ms/vs/17/release/vc_redist.x86.exe) |

x64 引擎需要三个文件：`MSVCP140.dll` / `VCRUNTIME140.dll` / `VCRUNTIME140_1.dll`
（位于 `C:\Windows\System32`）；x86 引擎只需要前两个（位于 `C:\Windows\SysWOW64`）。
**画图、编辑、导入导出 GIF 不需要这些运行库** —— 只有 AI 文生图需要。

> 主程序本身**不需要**额外的 VCL/RTL 运行库（已静态链接）。

---

## 3. 显卡加速（只影响文生图）

**结论：默认包里 x64 走 CPU，x86 自带显卡（DirectML）支持。**

| 运行时文件 | 内容 | 结果 |
|---|---|---|
| `supra\onnxruntime.dll` | x64 · **CPU 版**（无 DirectML 导出） | x64 + 勾「优先 GPU」→ **实际仍跑 CPU** |
| `supra\directml\onnxruntime.dll` | x64 · DirectML 版（体积更大） | 覆盖上去之后 x64 才能用显卡 |
| `supra\win32\onnxruntime.dll` | x86 · **自带 DirectML** | `MI_Editor_x86.exe` 可直接用显卡 |

想让 **x64** 用显卡（**部署操作，不是开关**）：

1. 关闭编辑器。
2. 备份原文件：把 `supra\onnxruntime.dll` 改名为 `onnxruntime.dll.cpu-backup`。
3. 把 `supra\directml\onnxruntime.dll` **复制**到 `supra\` 并**改名**为 `onnxruntime.dll`。
4. 重新启动编辑器 → 文生图里勾选「优先 GPU」并选中独立显卡。

> ⚠️ 混合显卡笔记本（核显 + 独显）**必须手动选独显** —— 不选默认用核显，速度只有独显的约 1/3 ~ 1/4。
> ✅ 界面会把「降级」说出来：文生图面板的 **Provider 标签**在勾选了「优先 GPU」但实际没生效时会显示 **`(未生效)`**。
> ⚠️ **具体原因只写在悬停提示里** —— 把鼠标停在 Provider 标签或显卡下拉框上才能看到（标签上只有 `未生效` 三个字，不会写原因）。

---

## 4. 设置文件与可写目录

| 项 | 位置 | 说明 |
|---|---|---|
| 设置文件 | `MI_Editor.ini` | 与 exe **同目录**，**正常退出**程序后自动生成（运行中不落盘） |
| 语言文件 | `Language.zh.ini` / `Language.en.ini` | 与 exe 同目录；缺失时回落到**内置英文**文案（不是中文） |

→ **程序目录必须可写**。放在只读目录（如 `C:\Program Files`）时：
画图功能照常，但**设置（语言、网格、线宽、MCP 开关等）无法保存**，下次启动回到默认值。
退出时程序会**弹窗告警**（标题「设置无法保存」）——按提示把整个目录换到有写权限的位置（例如用户文档目录）再运行一次即可。

**已知限制**：设置是**关闭程序前统一保存一次**的。
→ **异常退出**（崩溃 / 任务管理器强杀）会**丢失本次会话修改的设置**；**正常退出**则不会丢。

---

## 5. 目录结构

```text
<解压目录>/
├── MI_Editor.exe                    x64 主程序
├── MI_Editor_x86.exe                x86 主程序
├── Language.zh.ini                  界面中文
├── Language.en.ini                  界面英文
├── MI_Editor.ini                    设置（首次运行后生成，不属于安装包）
├── readme.md                        本文件
├── supra/
│   ├── SupraImgEngine.dll           x64 AI 引擎
│   ├── onnxruntime.dll              x64 推理运行时（CPU 版）
│   ├── directml/
│   │   └── onnxruntime.dll          x64 DirectML 版（按 §3 覆盖后启用显卡）
│   ├── win32/
│   │   ├── SupraImgEngine.dll       x86 AI 引擎
│   │   └── onnxruntime.dll          x86 推理运行时（自带 DirectML）
│   └── models/                      8 个模型文件，x64 与 x86 共用这一份（约 1.0 GB）
│       ├── t5_encoder.onnx          418.4 MB
│       ├── dit_step.onnx            397.7 MB
│       ├── vae_decoder.onnx         188.9 MB
│       ├── uncond_text.bin          0.4 MB
│       ├── uncond_mask.bin          0.0 MB
│       ├── spm_charmap.bin          0.2 MB
│       ├── spm_unicode.bin          0.0 MB
│       └── spm_vocab.bin            0.7 MB
├── mcp/
│   └── mcp_mosaic.exe               MCP 协议层 · 单档 13 个工具（含 open/save/doc）
└── docs/
    ├── readme.md                    本文件
    ├── MCP接入指南.md               让 AI 客户端连上编辑器
    └── MCP像素图像编辑器接口手册.md   13 个工具的接口语义（只含参数 / 返回 / 错误码）
```

> `supra\models\` 里的 8 个文件是**随包分发的静态资产**（不是运行时生成的），程序**只读不写**；
> 首次运行**只**会在退出时新生成 `MI_Editor.ini` —— 这是唯一的运行期产物。

| 想做的事 | 看哪份文档 |
|---|---|
| 正常画图 / 导入导出 / 做帧动画 | 本文件 + 编辑器内菜单 |
| 让 AI 客户端（Claude Code / pi / 其它）控制编辑器画图 | `docs/MCP接入指南.md` |
| 查某个 MCP 工具的参数、返回值、错误码 | `docs/MCP像素图像编辑器接口手册.md` |

---

## 6. 用 MCP 让 AI 帮你画

MCP 是「AI 客户端 → `mcp\mcp_mosaic.exe` → 命名管道 → 编辑器」的桥。要点：

1. **编辑器必须开着**（MCP 服务随编辑器启动）；
2. 编辑器菜单 **设置 → 启用 MCP**（默认已勾选）；
3. 在客户端里把命令指向 `<解压目录>\mcp\mcp_mosaic.exe`（**单档 13 个工具**，含 `mosaic_open` / `mosaic_save` / `mosaic_doc`）。
   包内**只有这一个 MCP 可执行文件**（`mcp\mcp_mosaic.exe`）。

完整步骤（含客户端配置片段、验证方法、常见错误码）见 **`docs/MCP接入指南.md`**。

---

## 7. 常见问题

| 现象 | 原因与处置 |
|---|---|
| 文生图报「无法加载引擎」或提示缺少 `MSVCP140.dll` | 未安装 VC++ 运行库 → 见 §2 |
| 文生图很慢，显卡占用为 0 | x64 默认 CPU 运行时 → 见 §3 覆盖 `onnxruntime.dll`；混合显卡还需在界面里**选独显**；面板上的 Provider 标签显示 `(未生效)` 即为降级（悬停看原因） |
| 设置改了就变回默认 | 程序目录不可写（退出时会有「设置无法保存」弹窗）→ 见 §4，换到可写目录 |
| 改了设置，异常退出后没保存 | 设置是**退出时统一保存**的 → 崩溃 / 任务管理器强杀会丢本次改动；正常退出即可保存（见 §4） |
| 界面是英文，或「语言」菜单里中文是灰的 | `Language.zh.ini` 被删/被移动 → 放回 exe 同目录；缺包时程序回落**英文**界面（**不会**弹窗提示） |
| 双击后没反应 | 32 位系统请用 `MI_Editor_x86.exe`；x64 版在 32 位 Windows 上无法运行 |
| 导出的 GIF 一直循环播，停不下来 | 工具栏「循环」按钮**关掉**后再导出 GIF —— GIF **尊重**该标志（关闭 → 不写 Netscape 循环扩展 → 播一次即停）。`.pxa` 同样存该标志（`flags.bit0`） |
| 写的 `.pxp` / `.pxa` 打不开 | 确认是本程序保存的文件：文件头 4 字节是 magic（`.pxp` 为 `30 50 58 50`、`.pxa` 为 `30 41 58 50`，按文本方式看分别显示成 `0PXP` / `0AXP`）；被外部工具改过的文件不保证可读 |

---

## 8. 卸载

**删掉这个目录即可** —— 不写注册表、不装系统服务。
若还想彻底清掉「MCP 定位文件」等本地状态，再删除 `%LOCALAPPDATA%\MosaicImageEditor\` 目录。
