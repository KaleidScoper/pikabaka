# Pika Windows 本地调试运行手册

本手册覆盖从零开始 —— `git clone` 刚完成，到项目成功在 Windows 上以开发模式运行的全流程。

---

## 目录

1. [前置环境安装](#1-前置环境安装)
2. [克隆 & 安装依赖](#2-克隆--安装依赖)
3. [配置环境变量](#3-配置环境变量)
4. [构建原生模块](#4-构建原生模块)
5. [启动开发模式](#5-启动开发模式)
6. [常见问题排查](#6-常见问题排查)

---

## 1. 前置环境安装

Pika 的技术栈为 **Electron + React + Vite + TypeScript + TailwindCSS**，音频捕获部分使用 **Rust (napi-rs)** 编写。需要以下基础环境：

### 1.1 Node.js (v20+ 推荐)

```powershell
# 推荐使用 fnm 管理 Node 版本 (Windows 下用 Scoop/Chocolatey 安装)
scoop install fnm
fnm install 20
fnm use 20

# 或直接从官网下载安装包
# https://nodejs.org/en/download
```

验证：
```powershell
node --version   # 应 ≥ v20
npm --version
```

### 1.2 pnpm

```powershell
npm install -g pnpm
# 或
scoop install pnpm
```

验证：
```powershell
pnpm --version
```

### 1.3 Rust (用于编译原生音频模块)

下载并运行 [Rust 官方安装器](https://rustup.rs/)，选择 **MSVC 工具链**（默认）：

```powershell
# 安装后验证
rustc --version
cargo --version

# 确认工具链为 MSVC（Windows 默认）
rustup show active-toolchain
# 应显示: stable-x86_64-pc-windows-msvc

# 如果没有 MSVC 工具链，手动安装
rustup toolchain install stable-x86_64-pc-windows-msvc
rustup default stable-x86_64-pc-windows-msvc
```

### 1.4 Visual Studio Build Tools（关键！）

Rust 的 MSVC 工具链和 Node 原生模块（better-sqlite3, sharp, keytar 等）都需要 C++ 编译工具链。

**任选其一：**

**方案 A — 安装 Visual Studio Build Tools（轻量，推荐）**
1. 下载 [Visual Studio Build Tools](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022)
2. 运行安装器，勾选 **"Desktop development with C++"** 工作负载
3. 安装

**方案 B — 安装完整 Visual Studio 2022 Community**
1. 下载 [VS 2022 Community](https://visualstudio.microsoft.com/downloads/)
2. 安装时勾选 **"Desktop development with C++"**

### 1.5 Python（node-gyp 需要）

部分 npm 原生模块通过 node-gyp 编译，依赖 Python。

```powershell
# 推荐通过 Microsoft Store 安装 Python 3.11/3.12
# 或
scoop install python
```

```powershell
python --version   # 应 ≥ 3.10
```

如果 Python 版本 > 3.12 且遇到 node-gyp 兼容问题，可安装 3.11：

```powershell
scoop install python311
```

### 1.6 Git

```powershell
scoop install git
# 或从 https://git-scm.com/download/win 下载
```

---

## 2. 克隆 & 安装依赖

### 2.1 克隆仓库

```powershell
git clone https://github.com/royisme/pikabaka.git
cd pikabaka
```

### 2.2 安装依赖

```powershell
pnpm install
```

`postinstall` 脚本会自动执行：
1. 为 `better-sqlite3` 和 `sqlite-vec` 执行 `electron-rebuild`（重编译原生模块匹配 Electron 的 Node ABI）
2. 下载两个本地 ML 模型（用于离线 RAG）：
   - `Xenova/all-MiniLM-L6-v2`（Embedding）
   - `Xenova/mobilebert-uncased-mnli`（意图分类）

如果 `pnpm install` 在此步骤失败，**不要慌**，参考 [第 6 节常见问题](#6-常见问题排查)。

---

## 3. 配置环境变量

Pika 支持多种 LLM 和 STT 提供商。在项目根目录创建 `.env` 文件：

### 3.1 最快上手：使用 Ollama（完全离线、免费）

如果你已安装 [Ollama](https://ollama.com/download/windows)，无需任何 API Key：

```env
# .env
VITE_STT_PROVIDER=ollama
VITE_LLM_PROVIDER=ollama
```

### 3.2 使用云端 LLM（需要 API Key）

```env
# .env —— 按需填写，最少配一个 LLM 提供商

# OpenAI
VITE_OPENAI_API_KEY=sk-xxxxxxxx

# Anthropic Claude
VITE_ANTHROPIC_API_KEY=sk-ant-xxxxxxxx

# Google Gemini
VITE_GOOGLE_API_KEY=xxxxxxxx

# Groq
VITE_GROQ_API_KEY=xxxxxxxx
```

> `VITE_` 前缀的变量会被 Vite 注入前端。Electron 主进程中也可能使用不带 `VITE_` 前缀的同名变量，视实际代码而定。

### 3.3 完整环境变量参考

| 变量 | 说明 |
|------|------|
| `OPENAI_API_KEY` / `VITE_OPENAI_API_KEY` | OpenAI API 密钥 |
| `ANTHROPIC_API_KEY` / `VITE_ANTHROPIC_API_KEY` | Anthropic Claude API 密钥 |
| `GOOGLE_API_KEY` / `VITE_GOOGLE_API_KEY` | Google Gemini API 密钥 |
| `GROQ_API_KEY` / `VITE_GROQ_API_KEY` | Groq API 密钥 |
| `VITE_STT_PROVIDER` | STT 提供商选择 |
| `VITE_LLM_PROVIDER` | LLM 提供商选择 |

---

## 4. 构建原生模块

Pika 的音频捕获使用了 Rust 编写的 `natively-audio` 模块。在首次运行前需要编译：

```powershell
pnpm run build:native:raw
```

此命令等价于：
```powershell
node scripts/build-native.js
```

脚本在非 macOS 系统上执行：
```powershell
npx napi build --platform --release
```

编译成功后，会在 `native-module/` 下生成 `index.win32-x64-msvc.node`（如果你是 x64 机器）。

验证：
```powershell
dir native-module\index.win32-*.node
# 应看到一个 .node 文件
```

---

## 5. 启动开发模式

```powershell
pnpm start
```

等价于：
```
pnpm app:dev
```

这条命令会同时启动两个进程：
1. **Vite 开发服务器** — `localhost:5180`，提供 React 前端热更新
2. **Electron 窗口** — 加载 Vite 开发服务器页面，显示 Pika 应用

### 如果你想跳过 lint + test 的验证步骤

正式 `pnpm start` 会走 `pnpm verify`（eslint + test）。本地调试可以手动分步启动：

**终端 1 — Vite 开发服务器：**
```powershell
pnpm run dev
```

**终端 2 — Electron（等 Vite 启动后）：**
```powershell
# 先编译 Electron TypeScript
pnpm run build:electron:raw

# 再启动 Electron
cross-env NODE_ENV=development electron .
```

---

## 6. 常见问题排查

### 6.1 `pnpm install` 失败：electron-rebuild 报错

```
✖ Rebuild Failed
```

**原因**：原生模块（better-sqlite3, sharp, keytar）编译失败，通常是因为缺少 MSVC 工具链或 Python。

**解决**：
1. 确认已安装 VS Build Tools 的 "Desktop development with C++"
2. 确认 Python 在 PATH 中：`python --version`
3. 手动重试重建：`npx electron-rebuild -f -w better-sqlite3,sqlite-vec`

### 6.2 `pnpm install` 失败：模型下载超时

```
Error downloading model: ...
```

**原因**：HuggingFace 在国内网络访问不稳定。

**解决**：
1. 设置 HuggingFace 镜像：
   ```powershell
   $env:HF_ENDPOINT="https://hf-mirror.com"
   pnpm run postinstall
   ```
2. 或挂代理后重试

### 6.3 `build:native:raw` 失败：找不到 napi

```
'napi' is not recognized
```

**解决**：
```powershell
npm install -g @napi-rs/cli
```

### 6.4 `build:native:raw` 失败：Rust 编译报错

**错误示例**：`error: linker 'link.exe' not found`

**原因**：Rust 工具链为 `gnu` 而非 `msvc`，或缺少 MSVC linker。

**解决**：
```powershell
# 切换到 MSVC 工具链
rustup default stable-x86_64-pc-windows-msvc

# 确认 VS Build Tools 已安装
# 或在 PowerShell (管理员) 中安装 Windows SDK
```

### 6.5 Electron 窗口白屏 / 无法加载

**可能原因**：
- Vite 开发服务器未启动（检查 `localhost:5180` 是否可访问）
- Windows 防火墙拦截了 localhost 连接

**解决**：
```powershell
# 先单独启动 Vite
pnpm run dev

# 浏览器访问 http://localhost:5180 ，确认页面可加载
```

### 6.6 keytar 编译失败

keytar 是凭证存储模块，在 Windows 上依赖 Windows API。

如果确实不需要凭证存储功能，可以暂时绕过：
1. 从 `package.json` 的 `dependencies` 中注释掉 `keytar`
2. 重新 `pnpm install`
3. 应用运行后，API Key 将不会加密存储到系统密钥链（不影响功能使用）

### 6.7 端口 5180 已被占用

**解决**：
```powershell
# 查找占用进程
netstat -ano | findstr :5180

# 终止占用进程 (替换 <PID>)
taskkill /PID <PID> /F
```

---

## 速查清单

在报告问题前，确认以下所有条件：

- [ ] Node.js ≥ v20
- [ ] pnpm 已全局安装
- [ ] Rust + MSVC 工具链就绪 (`rustup show` 显示 msvc)
- [ ] VS Build Tools 已安装（含 C++ 工作负载）
- [ ] Python ≥ 3.10 在 PATH 中
- [ ] `.env` 文件已创建（至少配了一个 LLM）
- [ ] `native-module/index.win32-*.node` 存在
- [ ] `pnpm run dev` 单独启动 Vite 可正常访问 `http://localhost:5180`
