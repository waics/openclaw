# OpenClaw 代理指南

> 本文档包含 AI 编程代理在 OpenClaw 项目上工作所需的基本信息。
> 在进行任何更改之前，请先阅读本文档。

## 项目概述

**OpenClaw** 是一个在您自己的设备上运行的个人 AI 助手。它可以在您已经使用的各种渠道上回复您（WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage 等 20 多个平台）。它可以在 macOS/iOS/Android 上说话和聆听，并可以渲染一个您可以控制的实时画布。

- **仓库**: https://github.com/openclaw/openclaw
- **网站**: https://openclaw.ai
- **文档**: https://docs.openclaw.ai
- **许可证**: MIT

### 核心特性

- **本地优先网关**: 为会话、渠道、工具和事件提供单一控制平面
- **多渠道收件箱**: 支持 25+ 个消息平台
- **多代理路由**: 将入站渠道/账户/对端路由到隔离的代理
- **语音功能**: 在 macOS/iOS 上支持语音唤醒，在 Android 上支持连续语音
- **实时画布**: 由代理驱动的可视化工作空间，使用 A2UI
- **可扩展性**: 基于插件的架构，支持渠道、内存和工具扩展


## 技术栈

### 核心运行时

| 组件 | 技术 |
|-----------|------------|
| **语言** | TypeScript (ESM, 严格模式) |
| **运行时** | Node.js ≥22.12.0 |
| **包管理器** | pnpm 10.23.0 (也支持 Bun 用于开发) |
| **构建工具** | tsdown (TypeScript 打包器) |
| **测试框架** | Vitest with V8 coverage |
| **代码检查** | Oxlint (类型感知) |
| **代码格式化** | Oxfmt |

### 移动应用

| 平台 | 技术 |
|----------|------------|
| **iOS** | Swift, SwiftUI (Observation 框架), XcodeGen |
| **Android** | Kotlin, Gradle |
| **macOS** | Swift, SwiftUI, AppKit |
| **共享组件** | OpenClawKit (共享 Swift 包) |

### 基础设施

- **容器**: Docker/Podman 多阶段构建 (Node 22 bookworm)
- **进程管理**: launchd (macOS), systemd (Linux)
- **文档**: Mintlify
- **CI/CD**: GitHub Actions (Blacksmith runners)

## 项目结构

```
openclaw/
├── src/                    # 核心 TypeScript 源代码
│   ├── cli/               # CLI 连接和 UI 组件
│   ├── commands/          # CLI 命令实现
│   ├── gateway/           # 网关服务器 (WebSocket 控制平面)
│   ├── agents/            # 代理运行时, Pi 集成, 工具
│   ├── channels/          # 渠道路由和共享逻辑
│   ├── telegram/          # Telegram 渠道 (grammY)
│   ├── discord/           # Discord 渠道 (discord.js)
│   ├── slack/             # Slack 渠道 (Bolt)
│   ├── signal/            # Signal 渠道 (signal-cli)
│   ├── imessage/          # iMessage 渠道 (legacy)
│   ├── web/               # WhatsApp Web (Baileys)
│   ├── webchat/           # WebChat UI
│   ├── browser/           # 浏览器自动化 (Playwright)
│   ├── media/             # 媒体管道 (图像/音频/视频)
│   ├── memory/            # 内存系统
│   ├── config/            # 配置管理
│   ├── infra/             # 基础设施工具
│   ├── plugin-sdk/        # 插件 SDK 导出
│   └── ...
├── extensions/            # 渠道/扩展插件 (工作区包)
│   ├── telegram/          # Telegram 扩展
│   ├── discord/           # Discord 扩展
│   ├── slack/             # Slack 扩展
│   ├── whatsapp/          # WhatsApp 扩展
│   ├── matrix/            # Matrix 扩展
│   ├── msteams/           # Microsoft Teams 扩展
│   ├── voice-call/        # 语音通话扩展
│   └── ... (33 个扩展)
├── apps/                  # 原生应用
│   ├── macos/             # macOS 菜单栏应用 (SwiftUI)
│   ├── ios/               # iOS 应用 (SwiftUI)
│   ├── android/           # Android 应用 (Kotlin)
│   └── shared/            # 共享 OpenClawKit Swift 包
├── skills/                # 捆绑技能 (代理工具)
├── ui/                    # Web UI (基于 Lit)
├── docs/                  # 文档 (Mintlify)
├── scripts/               # 构建和实用脚本
├── test/                  # 测试夹具和设置
└── packages/              # 内部包
```


## 构建和开发命令

### 设置

```bash
# 安装依赖
pnpm install

# 构建所有内容 (运行前必需)
pnpm build

# 仅构建 UI
pnpm ui:build
```

### 开发

```bash
# 在开发模式下运行 CLI (通过 tsx 使用 TypeScript)
pnpm openclaw <command>
pnpm dev                    # 同上

# 网关开发 (自动重载)
pnpm gateway:watch

# TUI 模式
pnpm tui
pnpm tui:dev               # 带配置文件的开发模式
```

### 代码质量

```bash
# 完整检查 (lint + 格式化 + 类型检查)
pnpm check

# 单独检查
pnpm lint                  # Oxlint 类型感知
pnpm lint:fix              # 自动修复 lint 问题
pnpm format:check          # 检查格式化
pnpm format:fix            # 修复格式化
pnpm tsgo                  # 仅 TypeScript 检查
```

### 测试

```bash
# 运行所有测试
pnpm test

# 特定测试套件
pnpm test:fast             # 仅单元测试
pnpm test:coverage         # 带覆盖率报告
pnpm test:e2e              # E2E 测试
pnpm test:live             # 实时测试 (需要 API 密钥)
pnpm test:channels         # 渠道特定测试
pnpm test:gateway          # 网关测试

# 内存受限测试
OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test
```

### 移动开发

```bash
# iOS
pnpm ios:gen               # 生成 Xcode 项目
pnpm ios:build             # 构建 iOS 应用
pnpm ios:run               # 在模拟器上构建并运行
pnpm ios:open              # 在 Xcode 中打开

# Android
pnpm android:assemble      # 构建 debug APK
pnpm android:install       # 安装到设备
pnpm android:run           # 安装并启动
pnpm android:test          # 运行单元测试
pnpm android:lint          # Kotlin lint

# macOS
pnpm mac:package           # 打包 macOS 应用
pnpm mac:restart           # 重启网关
```


## 代码风格指南

### TypeScript

- **严格类型**: 避免使用 `any`。需要时使用 `unknown` 和类型守卫。
- **仅 ESM**: 所有代码都是 ES 模块 (`"type": "module"`)
- **文件大小**: 目标文件大小在 500-700 行左右；可行时拆分
- **命名**: 
  - 产品/应用/文档标题使用 **OpenClaw**
  - CLI 命令、包名、路径、配置键使用 `openclaw`

### 代码模式

```typescript
// 优先使用显式继承而不是原型修改
// ❌ 不要这样做
applyPrototypeMixins(MyClass, [Mixin1, Mixin2]);

// ✅ 这样做
class MyClass extends BaseClass { }

// 动态导入: 不要为同一模块混用静态和动态导入
// ❌ 不要这样做
import { foo } from "module";
const { bar } = await import("module");

// ✅ 这样做 - 创建 *.runtime.ts 边界
// module.runtime.ts
export { foo, bar } from "module";
// consumer.ts
const { foo, bar } = await import("./module.runtime.ts");
```

### 测试

- **命名**: 源文件匹配 `*.test.ts`；E2E 测试使用 `*.e2e.test.ts`
- **同位**: 测试与源文件放在一起
- **覆盖率**: 70% 行/函数/语句，55% 分支
- **工作进程**: 最多 16 个工作进程 (在 vitest.config.ts 中配置)

### Lint/Format 规则

- 永远不要添加 `@ts-nocheck`
- 不要禁用 `no-explicit-any` - 修复根本原因
- 提交前运行 `pnpm check`
- 使用 `src/terminal/palette.ts` 中的共享 CLI 调色板 (不要使用硬编码颜色)

## 架构概述

### 网关架构

```
┌─────────────────────────────────────────────────────────────┐
│                        网关                               │
│                   (ws://127.0.0.1:18789)                    │
├─────────────────────────────────────────────────────────────┤
│  渠道  │  代理  │  工具  │  会话  │  WebSocket   │
├─────────────────────────────────────────────────────────────┤
│ Telegram  │  Pi RPC  │ 浏览器 │   内存   │  控制 UI  │
│  Discord   │          │ 画布 │   配置   │   WebChat    │
│   Slack    │          │ 节点  │   技能   │   协议   │
│  WhatsApp  │          │ 定时任务 │            │              │
│    ...     │          │  ...    │            │              │
└─────────────────────────────────────────────────────────────┘
```

### 关键模块

| 模块 | 用途 | 关键文件 |
|--------|---------|-----------|
| **网关** | WebSocket 控制平面, HTTP 服务器 | `src/gateway/server.ts` |
| **ACP** | 代理控制平面 - 会话管理 | `src/acp/` |
| **代理** | Pi 代理运行时, 模型管理 | `src/agents/` |
| **渠道** | 消息路由, 渠道抽象 | `src/channels/` |
| **自动回复** | 入站消息处理, 命令分发 | `src/auto-reply/` |
| **插件 SDK** | 扩展 API 和类型 | `src/plugin-sdk/` |


### 插件系统

扩展位于 `extensions/` 目录下，作为工作区包。每个扩展：

- 有自己的 `package.json`，带有 `"openclaw.extensions"` 入口点
- 可以扩展渠道、工具或内存
- 通过 jiti 动态加载
- 应将插件专用依赖项保留在扩展的 `package.json` 中

**重要**: 避免在扩展的 `dependencies` 中使用 `workspace:*`。将 `openclaw` 放在 `devDependencies` 或 `peerDependencies` 中。

### 渠道架构

添加渠道时，始终考虑所有内置 + 扩展渠道：

- **核心渠道**: Telegram, Discord, Slack, Signal, iMessage, WhatsApp Web, WebChat
- **扩展**: Matrix, MS Teams, Zalo, Feishu, LINE, Mattermost 等

添加渠道时更新 `.github/labeler.yml` 并创建匹配的 GitHub 标签。

## 测试策略

### 测试类型

| 类型 | 模式 | 命令 | 要求 |
|------|---------|---------|--------------|
| 单元测试 | `*.test.ts` | `pnpm test:fast` | 无 |
| E2E | `*.e2e.test.ts` | `pnpm test:e2e` | 已构建 dist |
| 实时测试 | `*.live.test.ts` | `pnpm test:live` | API 密钥 |
| Docker | - | `pnpm test:docker:*` | Docker |

### 测试配置

- **基础配置**: `vitest.config.ts`
- **单元测试**: `vitest.unit.config.ts`
- **E2E 测试**: `vitest.e2e.config.ts`
- **实时测试**: `vitest.live.config.ts`
- **网关测试**: `vitest.gateway.config.ts`
- **渠道测试**: `vitest.channels.config.ts`
- **扩展测试**: `vitest.extensions.config.ts`

### 覆盖率排除项

覆盖率有意排除：
- 入口点和 CLI 连接 (`src/cli/`, `src/commands/`, `src/entry.ts`)
- 集成表面 (`src/acp/`, `src/gateway/server.ts`)
- 渠道实现 (`src/telegram/`, `src/discord/` 等)
- 手动/E2E 验证代码 (TUI, 向导, 浏览器自动化)

## 安全注意事项

### 信任模型

OpenClaw **不**将单个网关建模为多租户：

- 经过身份验证的网关调用者被视为受信任的操作员
- 会话标识符是路由控制，不是授权边界
- 推荐: 每台机器/主机一个用户，每个用户一个网关
- 执行行为默认以主机优先 (`agents.defaults.sandbox.mode: off`)

### DM 安全默认值

- **默认**: 需要 DM 配对 (`dmPolicy="pairing"`)
- 未知发送者会收到配对码
- 使用以下命令批准: `openclaw pairing approve <channel> <code>`
- 选择公开 DM: 设置 `dmPolicy="open"` 并在允许列表中包含 `"*"`

### 安全检查清单

- 永远不要提交真实的电话号码、凭证或实时配置值
- 在文档中使用占位符如 `user@gateway-host`
- 运行 `openclaw doctor` 以显示有风险的配置
- 在进行分类/严重性决策之前阅读 `SECURITY.md`

### 常见误报

报告通常被关闭，不被视为漏洞：
- 没有边界绕过的提示注入
- 操作员意图的本地功能 (TUI `!` shell)
- 对允许列表会话的授权用户操作
- 没有认证/沙箱绕通的纯奇偶校验发现


## 部署

### Docker

```bash
# 构建镜像
docker build -t openclaw:local .

# 使用 docker-compose 运行
docker-compose up -d
```

构建参数：
- `OPENCLAW_EXTENSIONS`: 要包含的以空格分隔的扩展名称
- `OPENCLAW_VARIANT`: `default` 或 `slim`
- `OPENCLAW_INSTALL_BROWSER`: 为浏览器自动化安装 Chromium
- `OPENCLAW_INSTALL_DOCKER_CLI`: 为沙箱安装 Docker CLI

### NPM 包

```bash
# 全局安装
npm install -g openclaw@latest

# 入门向导
openclaw onboard --install-daemon
```

### 发布渠道

| 渠道 | 标签模式 | npm dist-tag | 说明 |
|---------|-------------|--------------|-------|
| stable | `vYYYY.M.D` | `latest` | 生产版本 |
| beta | `vYYYY.M.D-beta.N` | `beta` | 可能缺少 macOS 应用 |
| dev | `main` 分支 | - | 移动头部 |

切换: `openclaw update --channel stable|beta|dev`

## Git 和 PR 指南

### 提交规范

- 使用 `scripts/committer "<msg>" <file...>` 进行提交
- 格式: `<scope>: <action>` (例如 `CLI: add verbose flag to send`)
- 保持提交聚焦且原子化

### PR 指南

- 一个 PR = 一个议题/主题 (不要捆绑不相关的更改)
- 超过 ~5,000 行更改的 PR 是例外情况
- 为 UI/视觉更改包含截图
- 对于错误修复，在 `/landpr` 之前运行 `/reviewpr`
- 自行解决机器人审查对话

### 错误修复要求

在合并错误修复之前，需要：
1. 症状证据 (复现/日志/失败测试)
2. 代码中经过验证的根本原因，包含文件/行
3. 修复触及受影响的代码路径
4. 尽可能进行回归测试 (修复前失败/修复后通过)

### 自动关闭标签

应用这些标签进行自动处理：
- `r: skill` - 在 Clawhub 上发布技能
- `r: support` - 重定向到 Discord
- `r: third-party-extension` - 作为第三方插件发布
- `r: testflight` - 不可用，从源代码构建
- `invalid` - 无效项目

## 文档

### 文档结构

- **位置**: `docs/`
- **格式**: Markdown (Mintlify 风格)
- **托管**: Mintlify (docs.openclaw.ai)

### 链接规则

- 内部文档: 根相对路径，无 `.md`/`.mdx` (例如 `[Config](/configuration)`)
- 锚点: 使用根相对路径 (例如 `[Hooks](/configuration#hooks)`)
- README: 绝对 URL (`https://docs.openclaw.ai/...`)
- 避免在标题中使用 em 破折号和撇号 (会破坏 Mintlify 锚点)

### i18n (zh-CN)

- `docs/zh-CN/**` 是生成的 - 不要直接编辑
- 流程: 更新英文 → 调整词汇表 → 运行 `scripts/docs-i18n`
- 参见 `docs/.i18n/README.md`


## 配置文件参考

| 文件 | 用途 |
|------|---------|
| `package.json` | npm 包配置、脚本、依赖 |
| `pnpm-workspace.yaml` | pnpm 工作区配置 |
| `tsconfig.json` | TypeScript 编译器选项 |
| `vitest.config.ts` | Vitest 测试配置 |
| `pyproject.toml` | Python 工具 (ruff, pytest for skills) |
| `Dockerfile` | 多阶段 Docker 构建 |
| `docker-compose.yml` | Docker Compose 服务 |
| `knip.config.ts` | 死代码检测配置 |
| `.oxlintrc.json` | Oxlint 配置 |
| `.swiftlint.yml` | Swift 代码检查规则 |
| `.swiftformat` | Swift 格式化规则 |

## 故障排除

### 常见问题

1. **缺少依赖**: 运行 `pnpm install` 然后重试命令
2. **类型错误**: 运行 `pnpm build` 检查 `[INEFFECTIVE_DYNAMIC_IMPORT]` 警告
3. **测试失败**: 对于内存受限的主机使用 `OPENCLAW_TEST_PROFILE=low`
4. **macOS 日志**: 使用 `./scripts/clawlog.sh` 查询统一日志

### 多代理安全

- 除非请求，否则不要创建/应用/删除 `git stash` 条目
- 不要创建/删除/修改 `git worktree` 检出
- 除非明确请求，否则不要切换分支
- 专注于您的编辑；仅在相关时注明"存在其他文件"

## 关键外部依赖

| 包 | 用途 |
|---------|---------|
| `@mariozechner/pi-*` | Pi 代理运行时核心 |
| `grammy` | Telegram 机器人框架 |
| `@slack/bolt` | Slack 应用框架 |
| `discord.js` | Discord API 客户端 |
| `@whiskeysockets/baileys` | WhatsApp Web API |
| `playwright-core` | 浏览器自动化 |
| `sharp` | 图像处理 |
| `zod` | Schema 验证 |
| `@sinclair/typebox` | JSON Schema 类型 |

## 仓库指南 (原始)

- 仓库: https://github.com/openclaw/openclaw
- 在聊天回复中，文件引用必须仅为仓库根相对路径 (例如: `extensions/bluebubbles/src/channel.ts:80`)；不要使用绝对路径或 `~/...`。
- GitHub issues/comments/PR 评论: 使用字面量多行字符串或 `-F - <<'EOF'` (或 $'...') 表示真实的换行；永远不要嵌入 "\\n"。
- GitHub 评论陷阱: 当正文包含反引号或 shell 字符时，永远不要使用 `gh issue/pr comment -b "..."`。始终使用单引号 heredoc (`-F - <<'EOF'`) 以避免命令替换/转义损坏。
- GitHub 链接陷阱: 当您想要自动链接时，不要像 `#24643` 这样的 issue/PR 引用用反引号包裹。使用普通 `#24643` (可选择添加完整 URL)。
- PR 落地评论: 始终使用完整的提交链接使提交 SHA 可点击 (包括落地 SHA + 源 SHA，如果存在)。
- PR 审查对话: 如果机器人在您的 PR 上留下审查对话，请自行解决并在修复后关闭这些对话。仅在需要审查者或维护者判断时才保留对话未解决；不要将机器人对话清理留给维护者。
- GitHub 搜索陷阱: 当想要搜索所有内容时，不要将自己限制在前 500 个 issue 或 PR。除非您应该查看最新内容，否则继续直到到达搜索的最后一页
- 安全公告分析: 在进行分类/严重性决策之前，阅读 `SECURITY.md` 以与 OpenClaw 的信任模型和设计边界保持一致。

## 资源

- **愿景**: `VISION.md`
- **贡献**: `CONTRIBUTING.md`
- **安全**: `SECURITY.md`
- **更新日志**: `CHANGELOG.md`
- **维护者**: 参见 `CONTRIBUTING.md` 获取完整列表

---

*最后更新: 2026-03-10*
*如有疑问，请在 Discord 中提问或参考 https://docs.openclaw.ai*
