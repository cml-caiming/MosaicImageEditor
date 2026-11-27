# MCP 接入指南

> 适用版本：Mosaic Image Editor **1.1.0.5** ｜ 面向 **任意支持 MCP（Model Context Protocol）的 AI 客户端**（Claude Desktop / Claude Code / pi / Cursor / Cline / Windsurf / 自研客户端）
> 本文件只讲「**怎么连上**」；每个工具的入参、返回值与错误码见 [`MCP像素图像编辑器接口手册.md`](./MCP像素图像编辑器接口手册.md)。

---

## 1. 整体结构

```text
AI 客户端 ──stdio（换行分隔的 JSON-RPC 2.0）──▶ mcp\mcp_mosaic.exe ──命名管道──▶ MI_Editor.exe
                                              （协议层：握手/工具表/参数校验）      （唯一持有文档，执行在 UI 线程）
```

| 环节 | 说明 |
|---|---|
| 客户端 → `mcp_mosaic.exe` | **stdio**，**换行分隔的 JSON-RPC 2.0**（不是 `Content-Length` 头格式） |
| `mcp_mosaic.exe` → 编辑器 | 命名管道 `\\.\pipe\MosaicMcp-<pid>-<salt>`（编辑器启动时生成，每次不同） |
| 谁干活 | 协议层**不做任何绘制**；像素操作全部由**编辑器进程**执行 → **编辑器必须开着** |
| 协议层怎么找到编辑器 | 读定位文件 `%LOCALAPPDATA%\MosaicImageEditor\mcp-locator.json`，连上后校验 PID |

---

## 2. 前置条件

1. 编辑器已启动（`MI_Editor.exe` 或 `MI_Editor_x86.exe`）。
2. 菜单 **Settings → Enable MCP** 已勾选（**默认开启**，切换**热生效**）。
3. 客户端与编辑器是**同一个 Windows 用户 / 同一会话**（跨用户、跨会话不支持）。

任一不满足 → 所有工具返回 `E_NO_GUI`（`retryable: true`）。编辑器状态栏右下角会显示 **「MCP 可用 · 13 工具」**；
被停用时显示 **「MCP 已停用 / MCP disabled」**（此时也是 `E_NO_GUI`，与「编辑器没开」同一个码，**无法区分**）。

---

## 3. 工具面：单档 13 个工具

| 项 | 值 |
|---|---|
| 产物 | **只有一个**：`mcp\mcp_mosaic.exe` |
| 工具数 | **13** |
| `serverInfo` | `variant = "full"`、`toolCount = 13` |

工具名单：`mosaic_view` / `mosaic_draw` / `mosaic_fill` / `mosaic_pixel_get` / `mosaic_doc` /
`mosaic_open` / `mosaic_save` / `mosaic_new` / `mosaic_palette` / `mosaic_undo` / `mosaic_frame` /
`mosaic_region_get` / `mosaic_batch`。

> 📌 **不需要按用途选 exe**：绘图、读写工程、导出 GIF/PNG/BMP/PXA、帧操作全在这一个进程里。
> 📌 调用不存在的工具会得到 JSON-RPC 级 **`-32602`**，消息里写明当前工具面：
> `Unknown tool: X (this build exposes 13 tools, variant=full)`。

---

## 4. 客户端配置

把命令指向 **`<解压目录>\mcp\mcp_mosaic.exe`**，`args` 留空。

### 4.1 通用 `mcpServers` 配置（Claude Desktop / Cursor / Cline / Windsurf 等）

```json
{
  "mcpServers": {
    "mosaic": {
      "command": "<解压目录>\\mcp\\mcp_mosaic.exe",
      "args": []
    }
  }
}
```

> Windows 路径在 JSON 里写 `\\` 或 `/`。放好后**重启客户端**（多数客户端不会热加载 MCP 配置）。

### 4.2 pi

项目内 `.pi/mcp.json`（或全局 `~/.pi/agent/mcp.json`）：

```json
{
  "mcpServers": {
    "mosaic": {
      "type": "stdio",
      "command": "C:\\Tools\\MosaicImageEditor\\mcp\\mcp_mosaic.exe",
      "cwd": "C:\\Tools\\MosaicImageEditor",
      "timeout": 60
    }
  }
}
```

验证：`pi mcp list` → 应显示 `mosaic: connected, 13 tools`。

### 4.3 Claude Code

```powershell
claude mcp add mosaic -- "C:\Tools\MosaicImageEditor\mcp\mcp_mosaic.exe"
claude mcp list                      # 应显示 mosaic: connected
```

- 也可写项目级 `.mcp.json`（**仅该项目有效**），格式同 §4.1。
- 首次添加后需要在会话里**批准一次**该服务器（`claude mcp list` 可能把已有服务器一起显示为 pending，属于客户端行为）。
- 需要放行工具权限时，在 `.claude/settings.json` 的 `permissions.allow` 里加 `mcp__mosaic__*`。

### 4.4 自研客户端

按 MCP 规范走 stdio：`initialize` → `tools/list` → `tools/call`。
请先读 `initialize` 响应里的 `serverInfo`（`variant` / `toolCount`），不要按工具名硬编码能力判断。

---

## 5. 协议层怎么找到编辑器

- 编辑器启动时写 **`%LOCALAPPDATA%\MosaicImageEditor\mcp-locator.json`**（含管道名 + pid）；
  多开时**每个实例**再写一份侧车 `mcp-locator-<pid>.json`，退出只删自己那份。
- `mcp_mosaic.exe` 按「主文件 → 侧车（新→旧）」逐个试连，第一个用完整超时，其余快速试探。
- 多开做 A/B 对照时，用环境变量指定目标实例：

```powershell
$env:MOSAIC_MCP_TARGET_PID = 12345      # 只连 pid=12345 的那个编辑器
```

- 传输层错误带 `error.data.detail`（试过哪些管道 / 哪个 pid / Win32 错误码），据此判断「编辑器没开」还是「连错实例」。

---

## 6. 快速验证（30 秒）

1. 启动编辑器 → 状态栏应显示 **「MCP 可用 · 13 工具」**。
2. 客户端里调 `mosaic_doc`（只读）：

```json
{ "name": "mosaic_doc", "arguments": {} }
```

3. 返回里应看到 `"pid"`（编辑器进程）、`"canvasActive"`、`"frameCount"`、`"toolCount"` 由 `serverInfo` 给出。
4. 再调 `mosaic_view` → 能拿到 PNG 就说明链路通了。

> 开工前建议固定顺序：`mosaic_doc`（确认连的是哪个窗口 + 有没有画布 + 有没有人在操作界面）→ 再画。

---

## 7. 两个容易被忽略的行为

1. **一次工具调用 = 一次撤销**（等价人类的一次 Ctrl+Z）。`mosaic_batch` 整批算一次。
2. **人类正在操作界面时写类工具会被拒**：返回 `E_BUSY`（`retryable: true`）→ **稍后重试**，不要当失败放弃；
   读类工具（`mosaic_view` / `mosaic_pixel_get` / `mosaic_region_get`）照常执行。
   播放中写类工具返回 `E_PLAYING`；没有活动画布返回 `E_NO_DOC`。

完整错误码与重试策略见接口手册 §3。

---

## 8. 相关文档

| 想知道 | 看 |
|---|---|
| 每个工具的参数 / 返回 / 错误码 | [`MCP像素图像编辑器接口手册.md`](./MCP像素图像编辑器接口手册.md) |
| 编辑器本身的安装与使用 | 包根 [`readme.md`](../readme.md) |
| 结构断言与常见故障 | 本文件 §6 / §7 + 接口手册 §5 |
