# 你好，我是 xzyj50609 👋

考研在读，业余折腾 AI 工具链和各种自动化。日常环境是 **Windows 11 + PowerShell**，
喜欢把踩过的坑写成"排查路径"而不是"标准答案"——因为下次遇到的现象大概率不一样，但收敛思路是通用的。

- 🧪 在做的：考研真题分类自动化、PDF 智能书签生成
- 🛠️ 常用：Claude Code / CC Switch / Python / 一堆自写脚本
- 📓 习惯：把疑难杂症记成笔记，重点记**怎么排查**

下面置顶一篇我自己排了大半天的真实排障记录，希望能帮到遇到同类问题的人。

---

# 终于查明白了：为什么 CC Switch 打开的 Claude Code 永远不是完全访问模式

> *Fix CC Switch launching Claude Code without bypassPermissions on Windows*

**日期**：2026-06-13　|　**平台**：Windows + PowerShell / CMD / Windows Terminal　|　**相关工具**：CC Switch、Claude Code CLI 2.1.177

明明手动输入 `claude --dangerously-skip-permissions` 可以成功，但从 CC Switch 打开却一直不是 `bypassPermissions`。
真正的问题不在 Claude Code，而在**启动链路、临时 settings 和 npm shim**。

## 目录

- [一句话总结](#一句话总结)
- [初始症状](#初始症状)
- [关键背景](#关键背景)
- [排查过程](#排查过程)
- [关键反转：where claude 骗了我](#关键反转where-claude-骗了我)
- [最终修复](#最终修复)
- [最终验证](#最终验证)
- [如何判断是否复发](#如何判断是否复发)
- [回滚方法](#回滚方法)
- [这次排查的几个教训](#这次排查的几个教训)
- [参考链接](#参考链接)
- [☕ 如果帮到你](#-如果帮到你)

---

## 一句话总结

CC Switch 打开的 Claude Code 终端看起来能启动，但实际不是完全访问模式。

最终根因不是 Claude Code 本身不支持 `--dangerously-skip-permissions`，而是 **CC Switch 生成的临时启动脚本固定执行 `claude --settings "<临时配置.json>"`，启动 Claude 进程时走的是 CC Switch 自己继承到的旧 PATH**。

我们一开始只把 wrapper 放进了 `%USERPROFILE%\.cc-switch\bin`，进入 Claude 后 `where claude` 确实能看到 wrapper 第一，但这已经是 Claude 内部工具环境的 PATH 了，不代表"启动 Claude 进程那一步"也经过了 wrapper。

最终通过接管 npm 的 `claude.cmd` shim 解决：

```text
%USERPROFILE%\AppData\Roaming\npm\claude.cmd
```

原始 npm shim 备份为：

```text
%USERPROFILE%\AppData\Roaming\npm\claude-real.cmd
```

现在即使 CC Switch 的旧 PATH 先命中 npm shim，也会被 wrapper 拦住，并自动变成：

```cmd
claude-real.cmd --dangerously-skip-permissions --settings "<临时配置.json>"
```

同时 wrapper 会 patch 这个临时 settings 文件，补上：

```json
{
  "skipDangerousModePermissionPrompt": true,
  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}
```

这篇笔记的价值在于：问题表面看起来像"Claude Code 参数没传进去"，但真正难点是 **你在 Claude 里面看到的 PATH，不等于 CC Switch 启动 Claude 那一刻用的 PATH**。这也是整个排查里最容易被误导的地方。

---

## 初始症状

在 CC Switch 里点击 Claude 供应商的"终端"按钮后，终端能打开，Claude Code 也能启动，但状态不是完全访问模式。

手动在普通终端输入 `claude --dangerously-skip-permissions` 可以进入完全访问模式，但从 CC Switch 打开的窗口不行。

后来在新窗口里确认 `where claude`，输出第一项已经是：

```text
%USERPROFILE%\.cc-switch\bin\claude.cmd
```

但 Claude 会话日志仍显示：

```text
permissionMode=default
```

这说明问题不在"Claude 会话内部能不能找到 wrapper"，而在"CC Switch 启动 Claude 进程时有没有经过 wrapper"。

问题分成了两层：

1. 第一层是 CC Switch 打开的终端找不到 `claude`。
2. 第二层是终端能打开以后，Claude Code 仍然不是完全访问模式。

第一层看起来像 PATH 问题，第二层看起来像 Claude Code 参数或 settings 问题。真正麻烦的是，第一层修好以后，第二层仍然存在。

---

## 关键背景

**CC Switch 用户手册说明：**

- Claude Code 供应商切换主要通过写 `~/.claude/settings.json` 生效。
- 终端设置只负责选择 Windows CMD、PowerShell、Windows Terminal。
- 手册没有提供"给打开的 Claude 终端追加 CLI 参数"的官方开关。

**Claude Code 官方文档说明：**

- `--dangerously-skip-permissions` 等同于 `--permission-mode bypassPermissions`。
- `skipDangerousModePermissionPrompt` 只负责跳过进入 bypass mode 的确认提示。
- `permissions.disableBypassPermissionsMode = "disable"` 会禁用 bypass。
- `permissions.defaultMode = "bypassPermissions"` 可以作为 settings 里的默认权限模式。

本机没有发现 managed settings 禁用 bypass（`C:\Program Files\ClaudeCode\managed-settings.json` 不存在）。

---

## 排查过程

### 1. 先确认 `claude` 命令是否可用

一开始 CC Switch 打开的终端报 `'claude' 不是内部或外部命令`。本机直接 `where claude` 能找到 npm 的 `claude.cmd` 和 WinGet 的 `claude.exe`，`claude --version` 返回 `2.1.177 (Claude Code)`。

所以不是 Claude Code 没装，而是 **CC Switch 启动环境里的 PATH 不完整**。

### 2. 发现 CC Switch provider env 可能覆盖 PATH

CC Switch 会为每个 provider 写临时配置：

```text
%USERPROFILE%\AppData\Local\Temp\claude_<provider-id>_<pid>.json
```

检查 CC Switch 的 SQLite 数据库 `%USERPROFILE%\.cc-switch\cc-switch.db`，发现 provider 配置在 `providers.settings_config` 和 `settings.common_config_claude`。当 provider 的 `env` 被当作进程环境使用时，如果里面没有 `PATH/PATHEXT`，就会导致 `claude` 找不到。

于是先给所有 Claude provider 和公共配置补了 `PATH` / `PATHEXT`。**这一步解决了"打不开 / 找不到 claude"的问题**——第一阶段的阶段性成功：能启动了，但还没解决"是不是完全访问模式"。

### 3. 第一版 wrapper：放到 `.cc-switch\bin`

创建 `%USERPROFILE%\.cc-switch\bin\claude.cmd`，目标是让 CC Switch 执行 `claude --settings ...` 时自动补 `--dangerously-skip-permissions`，并把用户 PATH 前面加上 `.cc-switch\bin`。

验证新进程里 `where claude` 第一项确实变成了 wrapper，**但问题仍然存在**。

> 当时最迷惑的地方就在这里：`where claude` 明明已经显示 wrapper 排第一，但实际权限模式没有变化。这说明 `where claude` 这个证据本身需要重新解释。

### 4. 克隆 CC Switch 源码，确认它的真实启动模板

为了不再猜，浅克隆 CC Switch 源码并搜索启动逻辑，关键文件 `src-tauri\src\commands\misc.rs`，Windows 逻辑大意：

```rust
let bat_file = temp_dir.join(format!("cc_switch_claude_{}.bat", std::process::id()));

let content = format!(
    "@echo off
{cwd_command}
echo Using provider-specific claude config:
echo {}
claude --settings \"{}\"
del \"{}\" >nul 2>&1
del \"%~f0\" >nul 2>&1
",
    config_path_for_batch, config_path_for_batch, config_path_for_batch,
);
```

也就是说，CC Switch 安装版实际生成的 `.bat` 里就是裸的 `claude --settings "<临时配置.json>"`，**没有任何 `--dangerously-skip-permissions`**。

这一步把问题从"猜测"推进到了"源码证据"：想让它进入完全访问模式，就必须在 `claude` 这个命令被解析时做文章，或者修改 CC Switch 源码重新构建。

### 5. 发现 `--settings` 临时配置覆盖了用户 settings 的关键字段

用户 `~/.claude/settings.json` 里有 `skipDangerousModePermissionPrompt: true` 和 `allowDangerouslySkipPermissions: true`，但 CC Switch 生成的临时 settings 只有 `env`（AUTH_TOKEN / BASE_URL / PATH / PATHEXT），**没有** `skipDangerousModePermissionPrompt`，也**没有** `permissions.defaultMode`。

这解释了为什么单独改 `~/.claude/settings.json` 不稳定：CC Switch 通过 `--settings <临时json>` 启动时，这个临时 settings 可能成为本次会话的关键 settings 来源。

### 6. 给 wrapper 增加 patcher

新增 `%USERPROFILE%\.cc-switch\bin\patch_ccswitch_claude_settings.py`，职责：从参数里找到 `--settings <path>` → 读取临时 JSON → 保留原有 `env` → 写入 `skipDangerousModePermissionPrompt: true` 和 `permissions.defaultMode: bypassPermissions`。

---

## 关键反转：where claude 骗了我

在 wrapper 中加入日志：

```cmd
echo %DATE% %TIME% wrapper-invoked add_skip=%ADD_SKIP% >> "%USERPROFILE%\.cc-switch\logs\claude-wrapper.log"
```

用户从 CC Switch 重新打开后，日志**只有测试记录**，没有真实启动的 `wrapper-invoked`。同时最新临时配置仍然只有 `{"env": {}}`，没有被 patch。再查最新 Claude 会话仍是 `permissionMode=default`。

所以最终确认：

> 进入 Claude 后 `where claude` 显示 `.cc-switch\bin\claude.cmd` 第一，并不能证明启动 Claude 时也用了它。
> 因为那是 Claude 运行后的工具环境 PATH，而不是 CC Switch 启动 `.bat` 时的 PATH。

CC Switch 自己进程仍然继承旧 PATH，启动时更早命中：

```text
%USERPROFILE%\AppData\Roaming\npm\claude.cmd
```

**这是整个排查的转折点**。此前的判断一直围绕 `.cc-switch\bin\claude.cmd`，但日志证明它没有被真实启动命中。问题不在 wrapper 写得不对，而在 wrapper 放的位置不够靠前。

---

## 最终修复

### 1. 备份 npm 原始 shim

把 `%USERPROFILE%\AppData\Roaming\npm\claude.cmd` 备份为 `claude-real.cmd`。原始内容大致是：

```cmd
@ECHO off
GOTO start
:find_dp0
SET dp0=%~dp0
EXIT /b
:start
SETLOCAL
CALL :find_dp0
"%dp0%\node_modules\@anthropic-ai\claude-code\bin\claude.exe"   %*
```

### 2. 用 wrapper 覆盖 npm shim

覆盖 `%USERPROFILE%\AppData\Roaming\npm\claude.cmd`，同时保留 `.cc-switch\bin\claude.cmd`。两个 wrapper 都指向真实入口 `claude-real.cmd`，避免递归：

```text
✅ 正确：wrapper -> claude-real.cmd -> node_modules\@anthropic-ai\claude-code\bin\claude.exe
❌ 错误：wrapper -> claude.cmd -> wrapper -> claude.cmd -> ...（无限递归）
```

### 3. wrapper 的关键行为

当参数包含 `--settings <临时配置.json>`，wrapper 会：

1. 记录日志到 `%USERPROFILE%\.cc-switch\logs\claude-wrapper.log`
2. patch 临时 settings（写入 `skipDangerousModePermissionPrompt` + `permissions.defaultMode: bypassPermissions`）
3. 用真实 Claude 启动：`claude-real.cmd --dangerously-skip-permissions --settings "<临时配置.json>"`

普通命令不乱加危险模式，例如 `claude --version` 仍然只是 `claude-real.cmd --version`。

---

## 最终验证

**版本不递归：** `claude --version` → `2.1.177 (Claude Code)`

**npm shim 会被接管**（dry-run 调用 npm shim）：

```text
"%USERPROFILE%\AppData\Roaming\npm\claude-real.cmd" --dangerously-skip-permissions --settings <临时配置.json>
```

**用户最终验证：** 从 CC Switch 重新打开 Claude 终端后，完全访问模式终于成功。这一刻才算真正闭环——不是"看起来应该成功"，而是启动链路、临时 settings、wrapper 日志和 Claude 会话权限模式都能互相印证。

---

## 如何判断是否复发

1. **看 `where claude` 解析顺序** —— 但注意这只能说明当前终端环境，不一定说明 CC Switch 启动 Claude 那一步。
2. **看 wrapper 日志** `%USERPROFILE%\.cc-switch\logs\claude-wrapper.log` —— 正常应有 `wrapper-invoked add_skip=1` + `patched-settings path=...claude_<provider-id>_<pid>.json`。如果只有测试记录，说明 CC Switch 启动没经过 wrapper。
3. **看最新 Claude 会话权限模式** —— 正常应是 `{"type":"permission-mode","permissionMode":"bypassPermissions"}`；如果是 `permissionMode=default` 就不是完全访问模式。
4. **看最新 CC Switch 临时 settings 是否被 patch** —— 正常应含 `skipDangerousModePermissionPrompt` + `permissions.defaultMode`；如果只有 `{"env": {}}` 说明 patcher 没跑。

---

## 回滚方法

如果以后 Claude Code 或 CC Switch 更新后不需要这个 workaround：

```cmd
:: 1. 恢复 npm shim
copy /Y "%USERPROFILE%\AppData\Roaming\npm\claude-real.cmd" "%USERPROFILE%\AppData\Roaming\npm\claude.cmd"

:: 2. 删除 CC Switch wrapper（可选）
del "%USERPROFILE%\.cc-switch\bin\claude.cmd"
del "%USERPROFILE%\.cc-switch\bin\patch_ccswitch_claude_settings.py"
```

3. 如不再需要，从用户环境变量 PATH 里移除 `%USERPROFILE%\.cc-switch\bin`。操作前建议先备份 PATH。

---

## 这次排查的几个教训

1. **`where claude` 要分层看** —— 在 Claude 会话里跑 `where claude` 只能说明 Claude 内部工具环境的 PATH，不能证明 CC Switch 启动 Claude 进程时也用了它。这次正是这个点误导了排查。
2. **`--settings` 不是普通 env 注入** —— CC Switch 执行的是 `claude --settings "<临时配置.json>"`，所以 `~/.claude/settings.json` 里的某些字段可能不会按预期参与本次启动。要让本次启动稳定进入 bypass，需要 patch 这个临时 settings。
3. **wrapper 放得太靠后不够** —— 放进 `.cc-switch\bin` 并写进 provider 的 `env.PATH`，只能影响 Claude 启动后的工具环境；CC Switch 进程自己继承旧 PATH，仍可能先找到 npm 的 `claude.cmd`。最终必须接管 npm shim。
4. **要用日志证明 wrapper 真的执行了** —— 这次真正收束问题的是日志。没有 `wrapper-invoked` 就说明 wrapper 没被启动，这比反复看 `where claude` 更可靠。

---

## 参考链接

- CC Switch 用户手册：<https://github.com/farion1231/cc-switch/blob/main/docs/user-manual/zh/README.md>
- CC Switch 切换供应商：<https://github.com/farion1231/cc-switch/blob/main/docs/user-manual/zh/2-providers/2.2-switch.md>
- CC Switch 终端设置：<https://github.com/farion1231/cc-switch/blob/main/docs/user-manual/zh/1-getting-started/1.5-settings.md>
- Claude Code CLI 参考：<https://code.claude.com/docs/zh-CN/cli-reference>
- Claude Code 权限模式：<https://code.claude.com/docs/en/permission-modes>

---

## ☕ 如果帮到你

这篇排障记录免费公开。如果它刚好帮你省下了几个小时，欢迎请我喝杯咖啡 ☕

| 微信支付 | 支付宝 |
|:---:|:---:|
| <img src="./assets/wechat-pay.jpg" width="240" alt="微信支付"> | <img src="./assets/alipay.jpg" width="240" alt="支付宝"> |

> 完全自愿，不打赏也欢迎你把这篇转给同样被 CC Switch 卡住的人。
