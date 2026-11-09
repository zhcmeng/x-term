# MFA 倒计时对齐与文件传输开关 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 MFA 倒计时与真实 TOTP 窗口对齐；给每个 MCP 连接加一个独立的文件传输自动执行开关。

**Architecture:** 前端只用后端已经返回的 `remainingSeconds` 反推窗口起点；后端在 `McpInstanceConfig` 加 `fileTransferAuto` 字段，用与 `RUN_MODES` 相同的运行时 map 保存，工具调用判定里把文件工具从「永远人工确认」改为「看该连接的开关」。

**Tech Stack:** Rust（Tauri 2）、Vue 3 + TypeScript、Element Plus。

**Spec:** `docs/superpowers/specs/2026-11-09-mfa-countdown-and-file-transfer-auto-design.md`

## Global Constraints

- 仓库根：`C:\Study\x-term`，remote `origin` 已指向 `https://github.com/zhcmeng/x-term`，`upstream` 指向原仓库。
- 新字段默认值必须是 `false`（升级后行为不变）。
- 配置字段走 serde camelCase：JSON 里是 `fileTransferAuto`。
- 前端没有测试框架（无 vitest / jest），前端的验证方式是 `pnpm build`（含 `vue-tsc --noEmit`）加手动验证。
- 后端测试用 `cargo test`，工作目录 `src-tauri`。
- 提交信息用中文，结尾带 `Co-Authored-By: Claude <noreply@anthropic.com>`。

## Review Focus

- **旧配置兼容**：升级后第一次启动，`mcp.json` 里没有 `fileTransferAuto` 字段，必须回退为 `false`，不能报解析错误。
- **周期不是 30 的条目**：`period=60` 的 TOTP 条目，倒计时应显示 60 以内的真实剩余秒数。
- **窗口边界刷新**：在还剩 0 到 1 秒时点刷新，不能把旧窗口的码标成新窗口（这正是不能直接用 `floor(now/period)*period` 的原因）。
- **`runMode=auto` 但文件开关关闭**：文件工具仍须弹人工确认，其它工具直接执行。
- **`runMode=manual` 但文件开关打开**：文件工具直接执行，其它工具仍须确认。
- **服务运行中切换开关**：保存配置后下一次调用就按新值执行，不需要重启 MCP 服务。

---

## 前置任务：安装 Rust 工具链

**Files:** 无（只装环境）

**Interfaces:**
- Produces: 可用的 `cargo` / `rustc` 命令，后续所有 Rust 任务依赖。

- [ ] **Step 1: 下载 rustup 安装器**

```powershell
Invoke-WebRequest -Uri https://win.rustup.rs/x86_64 -OutFile "$env:TEMP\rustup-init.exe"
```

- [ ] **Step 2: 用 MSVC 工具链静默安装**

```powershell
& "$env:TEMP\rustup-init.exe" -y --default-host x86_64-pc-windows-msvc --profile minimal
```

预期输出结尾：`Rust is installed now. Great!`

- [ ] **Step 3: 刷新 PATH 并验证**

```powershell
$env:Path = "$env:USERPROFILE\.cargo\bin;" + $env:Path
cargo --version
rustc --version
```

预期：两条命令都打印版本号，不报 `command not found`。

---

## Task 1: MFA 倒计时对齐

**Files:**
- Modify: `src/views/MfaView.vue:62-66`（`refreshAllCodes` 内）
- Modify: `src/views/MfaView.vue:294-298`（添加条目后的立即生成）

**Interfaces:**
- Consumes: `totpApi.totpGenerate(id)` 返回的 `TotpCode`，其中 `remainingSeconds: number` 是后端按 `period - (now % period)` 算出的真实剩余秒数（`src-tauri/src/totp.rs:149-155`）。
- Produces: `codes` map 里每条记录的 `codeTs` 语义改为「该码所属 TOTP 窗口的起点（Unix 秒）」。后续 `displayCode` / `progressPercent` / `remaining` / tick 判断都基于这个新语义，函数本身不用改。

- [ ] **Step 1: 改第一处（列表刷新）**

把 `src/views/MfaView.vue` 里的：

```ts
const c = await totpApi.totpGenerate(e.id);
next.set(e.id, {
  code: c.code,
  codeTs: Math.floor(Date.now() / 1000),
  period: e.period,
});
```

改为：

```ts
const c = await totpApi.totpGenerate(e.id);
const nowS = Math.floor(Date.now() / 1000);
next.set(e.id, {
  code: c.code,
  // codeTs 记窗口起点而不是取到码的时刻：用后端返回的剩余秒数反推，
  // 否则每次刷新倒计时都会从 period 重新开始，显示的时间比码的真实寿命长。
  codeTs: nowS - (e.period - c.remainingSeconds),
  period: e.period,
});
```

- [ ] **Step 2: 改第二处（添加后立即生成）**

把：

```ts
const c = await totpApi.totpGenerate(entry.id);
codes.value.set(entry.id, {
  code: c.code,
  codeTs: Math.floor(Date.now() / 1000),
  period: entry.period,
});
```

改为：

```ts
const c = await totpApi.totpGenerate(entry.id);
const nowS = Math.floor(Date.now() / 1000);
codes.value.set(entry.id, {
  code: c.code,
  codeTs: nowS - (entry.period - c.remainingSeconds),
  period: entry.period,
});
```

- [ ] **Step 3: 安装前端依赖并做类型检查**

```powershell
pnpm --dir "C:/Study/x-term" install
pnpm --dir "C:/Study/x-term" build
```

预期：`vue-tsc --noEmit` 无报错，vite 构建成功。

- [ ] **Step 4: 手动验证倒计时**

```powershell
pnpm --dir "C:/Study/x-term" tauri:dev
```

在应用里打开 MFA 页面，观察任意条目：

- 倒计时从当前窗口的真实剩余秒数开始，不是固定 30。
- 在显示 3 到 5 秒时点刷新按钮，倒计时仍显示 3 到 5 秒，不会跳回 30。
- 倒计时归零后自动换成新码。

- [ ] **Step 5: Commit**

```bash
git -C "C:/Study/x-term" add src/views/MfaView.vue
git -C "C:/Study/x-term" commit -F - <<'EOF'
fix(mfa): 倒计时按 TOTP 窗口起点计算

用后端返回的 remainingSeconds 反推窗口起点，不再把「取到码的时刻」
当成起点。修复刷新后倒计时总从 period 重新开始、码失效后仍继续显示的问题。

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
```

---

## Task 2: 后端配置字段与运行时存储

**Files:**
- Modify: `src-tauri/src/commands/mcp.rs`（`McpInstanceConfig` 结构、`default_for`、新增测试模块）
- Modify: `src-tauri/src/mcp/server.rs:88-112`（`RUN_MODES` 旁边加 `FILE_TRANSFER_AUTO`）

**Interfaces:**
- Produces:
  - `McpInstanceConfig.file_transfer_auto: bool`（serde 名 `fileTransferAuto`，默认 `false`）
  - `crate::mcp::server::set_file_transfer_auto(kind: McpKind, auto: bool)`
  - `crate::mcp::server::file_transfer_auto_of(kind: McpKind) -> bool`

- [ ] **Step 1: 写失败的运行时测试**

在 `src-tauri/src/mcp/server.rs` 的 `mod tests`（约 1918 行）里加：

```rust
/// 文件传输开关默认关闭；设置后查询返回设置值。
#[test]
fn file_transfer_auto_default_and_toggle() {
    assert!(!file_transfer_auto_of(McpKind::Ssh));
    set_file_transfer_auto(McpKind::Ssh, true);
    assert!(file_transfer_auto_of(McpKind::Ssh));
    set_file_transfer_auto(McpKind::Ssh, false);
    assert!(!file_transfer_auto_of(McpKind::Ssh));
}
```

- [ ] **Step 2: 跑测试确认失败**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml" file_transfer_auto_default_and_toggle
```

预期：编译失败，提示 `cannot find function file_transfer_auto_of` / `set_file_transfer_auto`。

- [ ] **Step 3: 实现运行时存储**

在 `src-tauri/src/mcp/server.rs` 的 `RUN_MODES` 定义后面加：

```rust
/// 运行时文件传输开关（per-kind）：true 表示该连接的文件工具跳过人工确认。
/// `mcp_save_config` 更新此值，改后立即生效无需重启。
static FILE_TRANSFER_AUTO: Lazy<Mutex<HashMap<McpKind, bool>>> =
    Lazy::new(|| Mutex::new(HashMap::new()));

/// 设置指定 kind 的文件传输开关。
pub fn set_file_transfer_auto(kind: McpKind, auto: bool) {
    FILE_TRANSFER_AUTO.lock().insert(kind, auto);
}

/// 查询指定 kind 的文件传输开关（默认 false，保持人工确认）。
pub fn file_transfer_auto_of(kind: McpKind) -> bool {
    FILE_TRANSFER_AUTO.lock().get(&kind).copied().unwrap_or(false)
}
```

- [ ] **Step 4: 跑测试确认通过**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml" file_transfer_auto_default_and_toggle
```

预期：`test result: ok. 1 passed`。

- [ ] **Step 5: 写失败的配置字段测试**

在 `src-tauri/src/commands/mcp.rs` 文件末尾加测试模块：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::mcp::approval::McpKind;

    /// 旧配置没有 fileTransferAuto 字段时，解析后必须是 false（保持人工确认）。
    #[test]
    fn file_transfer_auto_defaults_to_false_when_absent() {
        let json = r#"{
            "enabled": false,
            "host": "127.0.0.1",
            "port": 8765,
            "resourceMode": "bound",
            "boundSource": "config",
            "runMode": "manual",
            "enableLog": true,
            "idleTimeoutMinutes": 15,
            "baseIdleMinutes": 30
        }"#;
        let c: McpInstanceConfig = serde_json::from_str(json).unwrap();
        assert!(!c.file_transfer_auto);
    }

    /// 打开开关后能序列化并原样读回。
    #[test]
    fn file_transfer_auto_round_trips() {
        let mut c = McpInstanceConfig::default_for(McpKind::Ssh);
        c.file_transfer_auto = true;
        let json = serde_json::to_string(&c).unwrap();
        assert!(json.contains("\"fileTransferAuto\":true"));
        let back: McpInstanceConfig = serde_json::from_str(&json).unwrap();
        assert!(back.file_transfer_auto);
    }
}
```

- [ ] **Step 6: 跑测试确认失败**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml" file_transfer_auto
```

预期：编译失败，提示 `no field file_transfer_auto`。

- [ ] **Step 7: 实现配置字段**

在 `src-tauri/src/commands/mcp.rs` 的 `McpInstanceConfig` 结构里，`run_mode` 字段后加：

```rust
/// 文件传输工具（upload_file / download_file / bastion_upload_file）
/// 是否跳过人工确认。默认 false，保持强制确认。
#[serde(default = "default_file_transfer_auto")]
pub file_transfer_auto: bool,
```

在 `default_run_mode()` 旁边加：

```rust
/// 文件传输开关默认值：关闭（保持人工确认）。
fn default_file_transfer_auto() -> bool {
    false
}
```

在 `McpInstanceConfig::default_for` 的 `Self { ... }` 里补一行：

```rust
file_transfer_auto: false,
```

- [ ] **Step 8: 跑全部后端测试**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml" file_transfer_auto
```

预期：`2 passed`，另有 `file_transfer_auto_default_and_toggle` 一起通过。

- [ ] **Step 9: Commit**

```bash
git -C "C:/Study/x-term" add src-tauri/src/commands/mcp.rs src-tauri/src/mcp/server.rs
git -C "C:/Study/x-term" commit -F - <<'EOF'
feat(mcp): 增加文件传输自动执行配置字段与运行时开关

McpInstanceConfig 新增 fileTransferAuto（默认 false），
并在 server 模块加一组 per-kind 的运行时读写函数。

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
```

---

## Task 3: 工具调用的自动执行判定

**Files:**
- Modify: `src-tauri/src/mcp/server.rs:1186-1222`（`handle_tool_call` 的模式判定）
- Test: `src-tauri/src/mcp/server.rs` 的 `mod tests`

**Interfaces:**
- Consumes: `file_transfer_auto_of(kind) -> bool`（Task 2）
- Produces:
  - `fn is_file_tool_name(name: &str) -> bool`
  - `fn auto_run_decision(is_file_tool: bool, mode: &str, whitelist_hit: bool, file_auto: bool) -> bool`

- [ ] **Step 1: 写失败的判定测试**

在 `src-tauri/src/mcp/server.rs` 的 `mod tests` 里加：

```rust
/// 自动执行判定真值表：文件工具只看文件开关，其它工具看运行模式。
#[test]
fn auto_run_decision_truth_table() {
    // 非文件工具：auto 全放行；manual 全确认；whitelist 看是否命中。
    assert!(auto_run_decision(false, "auto", false, false));
    assert!(!auto_run_decision(false, "manual", false, true));
    assert!(auto_run_decision(false, "whitelist", true, false));
    assert!(!auto_run_decision(false, "whitelist", false, false));
    // 文件工具：只受文件开关控制，run_mode 与白名单都不参与。
    assert!(auto_run_decision(true, "manual", false, true));
    assert!(!auto_run_decision(true, "auto", false, false));
    assert!(!auto_run_decision(true, "whitelist", true, false));
    assert!(auto_run_decision(true, "whitelist", true, true));
}

/// 三个文件工具都被识别；exec_ssh / exec_sql / list_* 不算。
#[test]
fn is_file_tool_name_matches_only_transfer_tools() {
    assert!(is_file_tool_name("upload_file"));
    assert!(is_file_tool_name("download_file"));
    assert!(is_file_tool_name("bastion_upload_file"));
    assert!(!is_file_tool_name("exec_ssh"));
    assert!(!is_file_tool_name("exec_sql"));
    assert!(!is_file_tool_name("list_files"));
    assert!(!is_file_tool_name("bastion_session_exec"));
}
```

- [ ] **Step 2: 跑测试确认失败**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml" auto_run_decision is_file_tool_name
```

预期：编译失败，`cannot find function auto_run_decision`。

- [ ] **Step 3: 实现两个判定函数**

在 `src-tauri/src/mcp/server.rs` 的 `run_mode_label` 附近加：

```rust
/// 是否为文件传输工具（能读写本机任意路径）。
pub(crate) fn is_file_tool_name(name: &str) -> bool {
    matches!(
        name,
        "upload_file" | "download_file" | "bastion_upload_file"
    )
}

/// 本次调用是否跳过人工确认。
///
/// 文件工具只受该连接的文件传输开关控制；其它工具按运行模式与白名单判定。
pub(crate) fn auto_run_decision(
    is_file_tool: bool,
    mode: &str,
    whitelist_hit: bool,
    file_auto: bool,
) -> bool {
    if is_file_tool {
        file_auto
    } else {
        mode == "auto" || whitelist_hit
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml" auto_run_decision is_file_tool_name
```

预期：`2 passed`。

- [ ] **Step 5: 改 `handle_tool_call` 使用新判定**

把 `server.rs:1186-1202` 这一段：

```rust
let mode = run_mode_of(shared.kind);
let is_file_tool = matches!(
    name,
    "upload_file" | "download_file" | "bastion_upload_file"
);
let whitelist_hit = mode == "whitelist"
    && !is_file_tool
    && tools::is_whitelist_auto(&shared.state, name, arguments);
let auto_run = (mode == "auto" || whitelist_hit) && !is_file_tool;
```

改为：

```rust
let mode = run_mode_of(shared.kind);
let is_file_tool = is_file_tool_name(name);
let whitelist_hit =
    mode == "whitelist" && tools::is_whitelist_auto(&shared.state, name, arguments);
let auto_run = auto_run_decision(
    is_file_tool,
    &mode,
    whitelist_hit,
    file_transfer_auto_of(shared.kind),
);
```

同时把上方注释（1186-1193 行）里「无论何种模式都必须人工确认」一段，改为说明新的行为：

```rust
// 文件传输工具（upload_file / download_file / bastion_upload_file）能读写
// 本机任意路径，默认仍强制人工确认；打开该连接的「文件传输」开关后不再确认。
// 其它工具按运行模式判定：manual 人工确认、auto 全部自动、whitelist 白名单内自动。
```

- [ ] **Step 6: 改自动放行的日志标签**

把：

```rust
let tag = if whitelist_hit { "白名单放行" } else { "自动放行" };
```

改为：

```rust
let tag = if is_file_tool {
    "文件传输自动放行"
} else if whitelist_hit {
    "白名单放行"
} else {
    "自动放行"
};
```

- [ ] **Step 7: 跑全部后端测试**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml"
```

预期：全部通过，无编译警告新增。

- [ ] **Step 8: Commit**

```bash
git -C "C:/Study/x-term" add src-tauri/src/mcp/server.rs
git -C "C:/Study/x-term" commit -F - <<'EOF'
feat(mcp): 文件传输工具按连接开关决定是否自动执行

抽出 is_file_tool_name / auto_run_decision 两个判定函数并加真值表测试；
文件工具不再无条件走人工确认，改由 per-kind 的开关控制。

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
```

---

## Task 4: 启动与保存时同步开关

**Files:**
- Modify: `src-tauri/src/mcp/server.rs`（`start_server` 参数与初始化）
- Modify: `src-tauri/src/commands/mcp.rs`（`mcp_save_config`、`mcp_start` 的调用参数）

**Interfaces:**
- Consumes: `set_file_transfer_auto(kind, bool)`（Task 2）
- Produces: `start_server(...)` 多一个 `file_transfer_auto: bool` 参数；`mcp_save_config` 保存后立即更新运行时开关。

- [ ] **Step 1: 给 `start_server` 加参数并在启动时初始化**

在 `src-tauri/src/mcp/server.rs` 的 `start_server` 参数列表里，`run_mode: String` 后面加：

```rust
file_transfer_auto: bool,
```

在 `set_run_mode(kind, &run_mode);` 后面加：

```rust
set_file_transfer_auto(kind, file_transfer_auto);
```

- [ ] **Step 2: 找到并修改 `start_server` 的调用处**

```powershell
rg -n "start_server\(" "C:/Study/x-term/src-tauri/src"
```

在每个调用处传入 `config.file_transfer_auto`（变量名可能是 `config` 或 `cfg`，按实际上下文取）。预期只有 `commands/mcp.rs` 的 `mcp_start` 一处。

- [ ] **Step 3: 保存配置时立即生效**

在 `src-tauri/src/commands/mcp.rs` 的 `mcp_save_config` 里，`set_run_mode` 调用后面加：

```rust
crate::mcp::server::set_file_transfer_auto(kind, config.file_transfer_auto);
```

- [ ] **Step 4: 编译验证**

```powershell
cargo build --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml"
```

预期：exit 0，无 `unused variable` / `missing argument` 报错。

- [ ] **Step 5: Commit**

```bash
git -C "C:/Study/x-term" add src-tauri/src/mcp/server.rs src-tauri/src/commands/mcp.rs
git -C "C:/Study/x-term" commit -F - <<'EOF'
feat(mcp): 启动与保存配置时同步文件传输开关

启动服务时按配置初始化运行时开关；保存配置后立即更新，无需重启服务。

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
```

---

## Task 5: 前端开关界面

**Files:**
- Modify: `src/api/mcp.ts`（`McpInstanceConfig` 加字段）
- Modify: `src/components/McpInstancePanel.vue`（运行模式下方加一行开关）

**Interfaces:**
- Consumes: 后端 `mcp_save_config` 接受并持久化 `fileTransferAuto`。
- Produces: 用户在界面切换开关后立即保存。

- [ ] **Step 1: 加前端类型字段**

在 `src/api/mcp.ts` 的 `McpInstanceConfig` 里，`runMode` 字段后面加：

```ts
/**
 * 文件传输工具（upload_file / download_file / bastion_upload_file）
 * 是否跳过人工确认。默认 false。仅 ssh / file 两种 kind 有意义。
 */
fileTransferAuto: boolean;
```

- [ ] **Step 2: 加保存函数**

在 `src/components/McpInstancePanel.vue` 的 `saveRunMode` 函数后面加：

```ts
/** 文件传输开关切换：保存配置（后端立即生效，无需重启）。 */
async function saveFileTransferAuto() {
  try {
    await mcp.saveConfig(props.kind);
    lastSaved = { ...config.value };
    ElMessage.success(
      config.value.fileTransferAuto
        ? "文件传输已改为自动执行"
        : "文件传输已改为逐次确认"
    );
  } catch (e) {
    rollbackConfig();
    ElMessage.error("保存失败：" + String(e));
  }
}
```

- [ ] **Step 3: 加开关 UI**

在 `McpInstancePanel.vue` 模板里「运行模式」那个 `switch-row` 之后、「记录执行日志」之前，插入：

```vue
<!-- 文件传输开关（仅 ssh / file 有文件工具） -->
<div v-if="kind === 'ssh' || kind === 'file'" class="switch-row">
  <div class="switch-info">
    <div class="switch-title">
      文件传输
      <HelpTip
        content="打开后，该连接的上传、下载不再逐次弹确认。文件工具能读写本机任意路径，请只在信任外部客户端时打开。"
      />
    </div>
    <div class="switch-desc">
      {{ config.fileTransferAuto ? "上传、下载直接执行，不弹确认" : "每次上传、下载都要人工确认" }}
    </div>
  </div>
  <el-switch v-model="config.fileTransferAuto" @change="saveFileTransferAuto" />
</div>
```

注意：`kind` 必须是组件里已有的变量。确认它的名字（可能是 `props.kind` 或 computed），按文件里的实际写法替换。

- [ ] **Step 4: 类型检查与构建**

```powershell
pnpm --dir "C:/Study/x-term" build
```

预期：`vue-tsc --noEmit` 无报错。

- [ ] **Step 5: Commit**

```bash
git -C "C:/Study/x-term" add src/api/mcp.ts src/components/McpInstancePanel.vue
git -C "C:/Study/x-term" commit -F - <<'EOF'
feat(mcp): 界面增加每个连接独立的文件传输开关

SSH 与 File 连接的配置面板各有一个开关，切换后立即保存并生效。

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
```

---

## Task 6: 端到端验证

**Files:** 无（只验证）

- [ ] **Step 1: 跑全部后端测试**

```powershell
cargo test --manifest-path "C:/Study/x-term/src-tauri/Cargo.toml"
```

预期：全部通过。

- [ ] **Step 2: 启动应用**

```powershell
pnpm --dir "C:/Study/x-term" tauri:dev
```

- [ ] **Step 3: 逐项验证开关行为**

| 场景 | 操作 | 预期 |
|:---|:---|:---|
| 默认关闭 | 不改开关，从 Claude Code 调 `bastion_upload_file` | 弹出确认卡片 |
| 打开后自动 | 打开文件传输开关，再调一次 | 不弹确认，直接执行，日志记「文件传输自动放行」 |
| 与运行模式独立 | 把运行模式设为「自动运行」、文件开关关闭，再调上传 | 上传仍弹确认 |
| 反向独立 | 运行模式保持「手动运行」、文件开关打开，调 `bastion_session_exec` | 命令仍弹确认 |
| 重启后保持 | 关掉开关，重启应用 | 开关仍是关闭状态 |
| 旧配置兼容 | 用未含 `fileTransferAuto` 的旧 `mcp.json` 启动 | 正常启动，开关显示为关闭 |

- [ ] **Step 4: 验证 MFA 倒计时**

在 MFA 页面按 Task 1 Step 4 的三条标准再验一次。

- [ ] **Step 5: 推送分支**

```bash
git -C "C:/Study/x-term" push -u origin main
```

预期：推送成功，`https://github.com/zhcmeng/x-term` 上能看到全部提交。

---

## Self-Review 结论

- 设计文档的两节各自对应 Task 1 与 Task 2 到 Task 5；Task 6 覆盖设计里的验收标准。
- 前端没有测试框架，Task 1 与 Task 5 用类型检查加手动验证替代自动化测试，已在 Global Constraints 里写明。
- 所有新函数名在 Task 2、3、4 之间一致：`set_file_transfer_auto`、`file_transfer_auto_of`、`is_file_tool_name`、`auto_run_decision`。
- Review Focus 的六条都在对应任务里有测试或手动验证项：旧配置兼容（Task 2 Step 5）、周期非 30 与窗口边界（Task 1 Step 4）、两种独立组合（Task 6 Step 3）、运行中切换（Task 4 Step 3 + Task 6 Step 3）。
