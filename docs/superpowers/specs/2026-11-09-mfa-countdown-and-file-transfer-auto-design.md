# MFA 倒计时对齐与文件传输自动执行开关

日期：2026-11-09
仓库：zhcmeng/x-term（fork 自 qingfengCode/x-term）
基线提交：f2138f5

## 一、MFA 倒计时每次从 30 秒开始

### 现象

在 MFA 页面点刷新，倒计时每次都从 30 秒开始，和真实的 TOTP 窗口不同步。

### 原因

`src/views/MfaView.vue:64` 与 `:296` 把 `codeTs` 设成「取到验证码的那一刻」：

```ts
codeTs: Math.floor(Date.now() / 1000),
```

倒计时公式是 `period - (nowSec - codeTs)`（同文件 97-102 行）。起点永远是刷新时刻，所以永远从 30 开始。

TOTP 的窗口按整 30 秒对齐（0 秒、30 秒、60 秒各换一次）。后端 `src-tauri/src/totp.rs:149-151` 算得正确，并通过 `remaining_seconds` 返回了真实剩余秒数；前端 `src/api/totp.ts:20` 也定义了这个字段，但 `MfaView.vue` 一处都没有使用。

### 后果

界面会在码失效后继续显示它，最多 30 秒。用户按倒计时去输入时，可能输入的是已经作废的码。

### 改法

用后端返回的 `remainingSeconds` 反推窗口起点。

`src/views/MfaView.vue:62-66` 改为：

```ts
const c = await totpApi.totpGenerate(e.id);
const nowS = Math.floor(Date.now() / 1000);
next.set(e.id, {
  code: c.code,
  codeTs: nowS - (e.period - c.remainingSeconds),
  period: e.period,
});
```

`src/views/MfaView.vue:294-298` 同样处理（用 `entry.period`）。

不采用 `Math.floor(nowS / period) * period` 的原因：生成码和收到结果之间隔着一次 IPC。如果生成发生在窗口边界前、前端收到时已过边界，会把旧窗口的码标成新窗口，倒计时又从 30 开始。用后端返回的剩余秒数没有这个问题。

### 验收

- 窗口还剩 5 秒时点刷新，倒计时显示 5，不是 30。
- 倒计时归零时自动换新码。

## 二、文件传输自动执行开关

### 现状

MCP 实例有 `runMode` 三档（manual / whitelist / auto），但三个文件工具被硬编码排除在自动执行之外，见 `src-tauri/src/mcp/server.rs:1190-1202`：

```rust
let is_file_tool = matches!(
    name,
    "upload_file" | "download_file" | "bastion_upload_file"
);
let whitelist_hit = mode == "whitelist"
    && !is_file_tool
    && tools::is_whitelist_auto(&shared.state, name, arguments);
let auto_run = (mode == "auto" || whitelist_hit) && !is_file_tool;
```

注释写明的理由：这些工具的 `localPath` 不做沙箱限制，能读写本机任意路径，所以任何模式下都强制人工确认。

### 目标

每个 MCP 连接（`mcp.json` 里 ssh / db / file 各一份配置）各自有一个「文件传输」开关：

- 关闭（默认）：保持现状，每次上传、下载都弹人工确认。
- 打开：该连接的文件工具直接执行，不弹确认。

开关与 `runMode` 互相独立。可以只放开文件传输、命令执行仍保持确认；也可以反过来。

### 涉及的工具

| 连接 | 受开关控制的工具 |
|:---|:---|
| SSH MCP | `bastion_upload_file` |
| File MCP | `upload_file`、`download_file` |
| DB MCP | 没有文件工具，不显示开关 |

### 改动点

#### 1. 配置字段

`src-tauri/src/commands/mcp.rs` 的 `McpInstanceConfig` 加：

```rust
/// 文件传输工具是否跳过人工确认。默认 false（保持强制确认）。
#[serde(default = "default_file_transfer_auto")]
pub file_transfer_auto: bool,
```

`default_file_transfer_auto()` 返回 `false`。`Default for McpInstanceConfig`（同文件约 190-205 行）补上该字段的初始化。

#### 2. 运行时存储

`src-tauri/src/mcp/server.rs` 照 `RUN_MODES` 的写法加一份：

```rust
static FILE_TRANSFER_AUTO: Mutex<HashMap<McpKind, bool>>;

pub fn set_file_transfer_auto(kind: McpKind, auto: bool);
pub fn file_transfer_auto_of(kind: McpKind) -> bool;  // 未设置时 false
```

初始化位置：

- `start_server`（`server.rs:196` 的 `set_run_mode` 旁边）按配置设一次。
- `mcp_update_config`（`commands/mcp.rs:478-481`）保存配置时立即生效，不需要重启 MCP 服务。

#### 3. 判定逻辑

`server.rs` 的判断改为：

```rust
let is_file_tool = matches!(
    name,
    "upload_file" | "download_file" | "bastion_upload_file"
);
let auto_run = if is_file_tool {
    file_transfer_auto_of(shared.kind)
} else {
    mode == "auto"
        || (mode == "whitelist"
            && tools::is_whitelist_auto(&shared.state, name, arguments))
};
```

原来 `whitelist_hit` 里的 `!is_file_tool` 一并去掉，文件工具不再参与白名单判定。

#### 4. 前端

- `src/api/mcp.ts` 的 `McpInstanceConfig` 加 `fileTransferAuto: boolean`，注释写明默认 `false`。
- `src/components/McpInstancePanel.vue` 在「运行模式」下面加一行开关：
  - 标题：文件传输
  - 说明：打开后上传、下载不再逐次确认
  - 仅 `ssh` 与 `file` 两种 kind 显示（`db` 隐藏）
  - `@change` 复用现有保存函数，确保保存后立即生效

### 安全说明

- 默认关闭，升级后行为不变。
- 打开后，外部 MCP 客户端可以在没有任何提示的情况下读写本机任意路径的文件。界面上用说明文字写明这一点，由用户明确选择。
- 不做路径沙箱。用户选择的是「每个连接一个独立开关」，不是按目录放行。

### 验收

- 开关关闭时，`bastion_upload_file` 弹出确认卡片。
- 开关打开后，同一调用直接执行，执行日志里记录自动放行的标记。
- 切换开关不需要重启 MCP 服务，下一次调用就按新值执行。
- `runMode = manual` 且文件开关打开时，文件工具自动执行，其它工具仍然逐条确认。
