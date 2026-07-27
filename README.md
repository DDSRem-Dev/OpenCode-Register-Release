# OpenCode 注册机

个人学习场景下的 OpenCode 多账号管理工具。自动化注册流程中的重复步骤，支付和安全验证由用户完成。

> 仅用于个人批量管理已授权的 OpenCode 账号，不用于绕过付费或批量滥用。

## 功能

- Temp-Mail 浏览器临时邮箱（当前唯一支持 `https://temp-mail.org/en/`）
- GitHub 自动注册（邮件验证码自动读取，CAPTCHA 与风控人工介入）
- 显式启用的已遮罩人工介入截图（默认关闭、有限留存）
- OpenCode 登录与支付页自动跳转
- API Key 自动读取与本地加密存储
- 多账号号池配置，由 OMO 在受支持错误下按 fallback 链自动切换
- 账号加密导出/导入
- OpenCode Go 后台浏览器额度检测与额度预警
- GitHub 失效账号精确确认后自动删除及本地号池安全清理

项目功能已完整交付。PyInstaller/Tauri sidecar 二进制分发说明见「构建分发包」。
代码签名与 macOS 公证尚未交付，分发产物为未签名状态。

## 技术栈

- 桌面端：Tauri + React
- 自动化：Python + CloakBrowser
- 通信：Python 本地 HTTP / WebSocket 服务
- 存储：SQLite + 字段级 AES 加密
- 调度：应用生命周期内定时刷新 OpenCode Go 额度

## 架构说明

详细设计见 [docs/architecture.md](docs/architecture.md)。

核心要点：
- 桌面端：Tauri + React
- 自动化后端：Python + CloakBrowser
- 号池：由 Oh My OpenAgent (OMO) 的 `model_fallback` + `runtime_fallback` 自动切换
- 工具负责：创建账号、读取 API Key，并按界面开关写入 `auth.json` / `opencode.json` / `oh-my-openagent.json`
- 号池配置：用 OpenCode Go 官方模型 ID 与 Models.dev 结构化 SDK 元数据安全同步，自动编号并追加到 OMO 链尾
- 额度检测：后台 CloakBrowser 登录已验证 workspace，并直接读取 Go 仪表盘额度节点
- 失效清理：用户输入精确 GitHub 用户名授权删除，程序自动提交已验证表单，远端验证后再清理 SQLite 与号池配置
- 导入恢复：完整校验加密包后按目标机器自动配置设置重建配置；数据库提交失败会回滚本次配置写入
- 分发：PyInstaller 将 Python 后端打包为 Tauri sidecar 二进制

## 核心流程

```
临时邮箱 → GitHub 注册 → OpenCode 登录 → 跳转支付页 → 手动支付 → 记录 API Key → 加入号池
```

## 已交付能力

| 能力 | 内容 |
|------|------|
| 桌面运行时 | Tauri 壳、Python 本地服务与前后端通信 |
| 注册流程 | Temp-Mail 浏览器 provider、GitHub 注册与人工介入面板 |
| OpenCode 接入 | OpenCode 登录、支付跳转与 API Key 读取 |
| 本地存储 | SQLite 加密与号池配置写入（auth.json / opencode.json / oh-my-openagent.json） |
| 账号管理 | 账号列表、导出与导入 |
| 账号维护 | 额度检测与失效清理 |
| 应用分发 | PyInstaller + Tauri sidecar 二进制打包与文档 |

## 本地开发

### 环境要求

- Node.js 20.19+（或 22.12+）
- Rust 1.77.2+
- Python 3.11+
- [uv](https://docs.astral.sh/uv/)

### 首次安装

```bash
npm install
uv sync --project backend --group dev
uv run --project backend python -m cloakbrowser install
uv run --project backend python scripts/build_backend.py --placeholder
```

最后一条在任何 Cargo 或 `npm run tauri` 命令之前必须执行一次。`tauri.conf.json` 声明了
`externalBin`，`tauri-build` 会在 build script 阶段校验 sidecar 文件存在，全新 clone 在补上
占位文件前连 `cargo check` 都无法通过。占位文件是空文件，Rust 宿主会按长度拒绝并回落到开发期
解释器，不会顶替真实后端。

CloakBrowser 首次启动时也会自动下载浏览器二进制；上面的显式安装命令便于提前完成约 200 MB 的下载。

### 启动桌面应用

```bash
npm run tauri dev
```

Tauri 启动时会自动使用 `backend/.venv` 中的 Python 解释器拉起本地服务。桌面宿主每次启动时
在 `127.0.0.1` 上分配一个随机可用端口，并通过类型化 IPC 通知前端；退出桌面应用时对应进程会一起停止。

也可以分别启动后端和浏览器版前端，便于调试：

```bash
uv run --project backend python backend/main.py
npm run dev
```

不希望修改系统中的真实 OpenCode 配置时，使用沙盒模式启动后端：

```bash
OPENCODE_REGISTER_SANDBOX_DIR=.opencode-register-sandbox uv run --project backend python backend/main.py
npm run dev
```

沙盒模式会把 SQLite、`auth.json`、`opencode.json` 和 `oh-my-openagent.json` 全部写入
`.opencode-register-sandbox/`，并在界面显示“沙盒测试模式”。该模式只隔离本地文件，不会模拟
GitHub、Temp-Mail、OpenCode 或支付页面；这些仍然是真实外部服务。
Tauri 启动器会在切换 Python 工作目录前将相对沙盒路径按项目根目录转换为绝对路径，避免误建到
`backend/.opencode-register-sandbox/`。

额度调度间隔通过 `OPENCODE_REGISTER_QUOTA_CHECK_INTERVAL_SECONDS` 设置，默认 3600 秒，允许范围为
60–86400 秒。周期任务、批量刷新和单账号刷新统一使用独立的无窗口 CloakBrowser；后台登录后只读取
已验证 `/workspace/{workspace_id}/go` 页面中的三个额度百分比和订阅入口，检查结束立即关闭隔离会话。
若遇到 CAPTCHA、设备验证或未知阻断，检查停止并保留旧额度，不尝试绕过安全验证。

人工介入截图默认关闭。设置 `OPENCODE_REGISTER_SCREENSHOTS_ENABLED=true` 后，仅在 GitHub 人工介入时
捕获遮罩后的 PNG；输入框、验证码/密钥区域、iframe、canvas 及已知身份文本都会被遮罩。截图位于应用
私有数据目录的 `screenshots/`，默认最多保留 24 小时、每流程 3 张，并在恢复、取消、失败或完成时删除。
可分别用 `OPENCODE_REGISTER_SCREENSHOT_RETENTION_HOURS`（1–168）和
`OPENCODE_REGISTER_SCREENSHOT_MAX_PER_FLOW`（1–10）收紧边界。付款页面永不截图。

### 构建分发包

```bash
npm run package
```

该命令先用 PyInstaller 把后端冻结为单文件可执行程序，按 `rustc -vV` 的宿主三元组写入
`src-tauri/binaries/backend-<target triple>`，再运行 `tauri build` 生成桌面安装包。
也可以分开执行：

```bash
uv run --project backend python scripts/build_backend.py
npm run tauri build
```

产物位于 `src-tauri/target/release/bundle/`：macOS 为 `.app` 与 `.dmg`，Windows 为 NSIS 安装包。

打包后的应用不再需要 Python、uv 或 Rust 工具链。CloakBrowser 的 Chromium 不随包分发，
仍在首次使用时下载到 `~/.cloakbrowser`。

PyInstaller 不支持交叉编译，每个架构必须在对应机器上构建；`.github/workflows/release.yml`
在 tag 推送时按 macOS arm64、macOS Intel 和 Windows x64 三个 runner 出包并附校验和，
全部构建成功后创建对应的 GitHub Release 并上传带目标平台后缀的发布文件。

> 当前产物**未签名也未公证**。macOS 上下载后会被 Gatekeeper 拦截，需要手动放行；
> Windows 可能触发 Defender 提示。

### 验证

```bash
uv run --project backend pytest
uv run --project backend ruff check backend
uv run --project backend ruff check --config backend/pyproject.toml scripts
uv run --project backend mypy --config-file backend/pyproject.toml scripts
npm test
npm run build
cargo check --manifest-path src-tauri/Cargo.toml
```

拆分调试时，后端运行后可通过 `http://127.0.0.1:17891/api/health` 检查通信状态，API 文档位于
`http://127.0.0.1:17891/api/docs`。Tauri 桌面模式使用宿主通过 IPC 返回的动态端口。

本地账号库默认写入当前平台的应用数据目录。仅设置 `OPENCODE_REGISTER_DATA_DIR` 不会隔离
OpenCode 配置；需要完整隔离时必须使用 `OPENCODE_REGISTER_SANDBOX_DIR`。

## 免责声明

- 支付环节仅自动跳转页面，需用户手动完成支付。
- GitHub 邮件验证码由 Temp-Mail 浏览器读取；CAPTCHA、手机号、设备验证和未知风控均由人工介入。
- GitHub 删除只在用户重新输入完整用户名后自动提交；验证码、二次验证和未知安全挑战仍由用户处理。
- 请遵守 GitHub、OpenCode 及相关服务商的服务条款。
- 本项目仅用于个人学习场景，开发者需对使用方式负责。
