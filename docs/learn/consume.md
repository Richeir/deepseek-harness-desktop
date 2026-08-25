# DSH Desktop 学习文档（小白入门）

> 本文是给第一次接触本仓库的人准备的「上手地图」。目标不是把所有细节都讲一遍，而是让你：能看懂这个仓库到底装了什么、怎么跑起来、动手时该检查什么。
> 想深究某一层的原理，请跟着文末的链接去看对应的正式文档；本文只做导航和 Checklist。

---

## 0. 一句话理解这个项目

**DSH Desktop = 「官方 DeepSeek Harness（一个可组合的智能体运行时）」+「一层薄的 Electron 桌面壳」。**

它**没有重新实现**上游 Harness，而是把同一个运行时装进一个能双击启动、有窗口/托盘/更新、符合操作系统习惯的原生应用里。核心思想是 **「一切皆插件」**：Desktop 的窗口、托盘、终端、profile 管理本身也按 DSH 插件的方式接入，和第三方插件走同一套组合机制。

记住这张图（来自 `docs/architecture.md`）：

```
用户 → Electron main/托盘/窗口 → Profile launcher → Host Cordis generation
                                                          ├─→ loopback HTTP + WebSocket → 沙箱 Web renderer
                                                          ├─→ Upstream DSH services（agent/模型/工具/会话/Web UI）
                                                          └─→ Desktop-owned / Third-party plugins
```

> 关键：Desktop 只负责「壳」，真正的 agent、模型、工具、会话语义仍归上游 Harness。不要以为这里重新造了一套 Web。

---

## 1. 仓库顶层结构（先认路）

| 路径 | 是什么 | 你会什么时候进去 |
| --- | --- | --- |
| `dsh-plugin-desktop/` | **桌面产品主体**：Cordis Host/Client、Electron bootstrap、打包与发布测试都在这 | 改窗口/托盘/profile/终端/更新逻辑时 |
| `deepseek-harness/` | **固定版本的上游子模块**（原样运行的官方 Harness checkout） | ⚠️ 只读，不要在桌面 feature 分支里改它 |
| `dsh-community-fabric/` | 社区互操作 RFC（manifest/capability 草案），当前是私有文档脚手架 | 想参与统一插件 contract 讨论时 |
| `dsh-community-market/` | 插件市场壳：发现/详情/安装与管理，以开放方式接各种数据源 | 看插件市场产品与安全设计时 |
| `.agents/notes/**` | 日期化的维护者决策记录（architecture / process） | 追溯「为什么当初这么定」时 |
| `patches/`、`.yarn/patches` | 对上游 npm 包的 patch（Yarn resolution 指向这里） | 排查依赖行为差异时 |
| `scripts/` | 仓库级校验脚本（headless，见下面 Checklist C3） | 跑 `check` 前想明白它在查什么 |

> ⚠️ **最容易踩的坑**：`deepseek-harness/` 是上游子模块，有自己的 pnpm workspace。**不要**在桌面 feature 分支里直接改它里面的文件；要动上游能力时走 root 的 `upstream:*` 脚本（例如 `corepack yarn upstream:build`），并且把「更新 submodule pin」和「桌面行为变更」分成两个独立提交。

---

## Checklist C1 · 动手前环境准备

- [ ] Node 版本满足要求：`^22.19.0 || >=24.0.0`（见根 `package.json` 的 `engines`）
- [ ] 用 **Corepack** 驱动 Yarn，不要直接 `yarn install`：仓库固定 `yarn@4.18.0`、`nodeLinker: node-modules`
- [ ] 初始化固定的上游子模块（一次即可，或 submodule 有更新时）：

  ```sh
  git submodule update --init --recursive
  corepack yarn install --immutable   # 用 --immutable，不手写改 lockfile
  ```

- [ ] 确认包管理器分工没被破坏：**外层仓库 = Yarn**；`deepseek-harness/` **内部保持它自己的 pnpm**。二者不能混用。

> 为什么强调 `--immutable`：CI 和文档都要求可复现安装，手改 lockfile 会让后续校验失败。

---

## Checklist C2 · 跑起来（开发流程）

在仓库根目录执行：

```sh
corepack yarn dev      # = 先 build community-market，再起 dsh-plugin-desktop dev
```

其它常用入口（都从 root `package.json` 转发到各 workspace）：

| 命令 | 作用 |
| --- | --- |
| `corepack yarn start` | 直接起 `dsh-plugin-desktop`（已 build 过的快速启动） |
| `corepack yarn build` | 依次 build community-market → dsh-plugin-desktop |
| `corepack yarn package:dir` / `dist:mac` / `dist:win` | 打 Electron Builder 包到目录；出 macOS DMG / Windows NSIS |

> ⚠️ **图形界面启动必须是显式的**（AGENTS.md 规定）。build、typecheck、单测、Loader smoke 必须保持 headless-safe——不要为了「方便」把 `yarn dev` 塞进会弹窗口/阻塞 CI 的路径。
>
> 想验证「装出来的东西能起来」而不是「源码能编译」，用 `corepack yarn dist:mac-smoke`（或对应平台 smoke），它才是真·打包产物冒烟。

---

## Checklist C3 · headless 校验门槛

改完代码后用它确认没把门禁搞挂：

```sh
corepack yarn check      # = check:layout + 三个 workspace 的 check，全 headless-safe
```

拆开看它在查什么（都在根 `package.json`）：

- [ ] **bilingual-docs**：`test:bilingual-docs` / `check:bilingual-docs` → 校验中英文文档一致（`.md` + `.en.md` 成对、hash 对齐）。**你改了一份，另一份要跟着改。**
- [ ] **architecture gate**：`test:architecture-gates` → 校验依赖方向（market-dependency-direction），防止包之间乱引。
- [ ] **layout / verify-* `.mjs`**：`scripts/verify-layout.mjs`、`scripts/bilingual-docs.test.mjs`、`scripts/market-dependency-direction.*` 等，是仓库级脚本。

其它独立门槛：

```sh
corepack yarn typecheck   # TS 类型检查（dsh-plugin-desktop + dsh-community-market）
corepack yarn test        # 单测（同上两个 workspace）
```

> ⚠️ **常见顺序坑**：`typecheck` / `test` / `build` 都只覆盖 `dsh-plugin-desktop` 和 `dsh-community-market`，**不含** `dsh-community-fabric`。fabric 目前是私有文档脚手架（见它自己的 README），不要以为它有可跑入口。

---

## Checklist C4 · 「一切皆插件」边界（改代码前先确认）

- [ ] **第三方插件只依赖公开 contract。** Renderer（Web/浏览器界面）**不能**直接读 `desktopProfiles` / `desktopPnpm`；有 UI 的插件仍走普通 DSH Web routes、RPC、client metadata、service、slot。
- [ ] **公开 service 只有两个**：`dsh-plugin-desktop/profile-service` 与 `dsh-plugin-desktop/pnpm`（即 `desktopProfiles` + `desktopPnpm`）。其它都是 Desktop 私有内部实现，不要当 API 用。
- [ ] **兼容模式不覆盖上游布局。** Client face 校验环境即可；扩展窗口保留官方 layout、只通过 overlay slot 加 Desktop 操作栏；增强模式才装 Desktop-owned layout/frame（见 `docs/architecture.md`）。
- [ ] **generation 之间不能缓存跨代资源**：profile/模式一切换就 dispose 当前 generation 再起新的。service reference、窗口对象、subprocess handle 都不能跨 generation 复用或单独销毁。

> 想给插件加能力？先读 `docs/plugin-development.md`，再看 `dsh-plugin-desktop/docs/plugin-services.md`（稳定 contract + TS 示例）。别直接去翻 Electron 内部对象。

---

## Checklist C5 · profile / pnpm service 的语义

- [ ] **profile 名字与绝对目录只能来自 `desktopProfiles.current`**，不要从 argv、settings、URL 猜；`list()` 是只读发现，`select()` 记 pending target、靠重启才生效。
- [ ] **两种 pnpm 入口别混**：
  - `desktopPnpm.run()` → 直接跑内置 pnpm（作用于 Desktop 自己创建的进程）；
  - `runPlugin()` → 通过打包的 DSH CLI，维持 profile 初始化、相对 source 与 bundle reconcile。
- [ ] **两者都属当前 generation**，由 subprocess service 管理整棵进程树；不要假设它改了用户全局 PATH（内置 pnpm 只影响 Desktop 自己的进程）。

> ⚠️ **常见误解**：Desktop 不是「把官方 profile 复制一份到另一个数据库」。兼容模式默认共享 DSH home 里的会话和设置，没有第二套存储。

---

## Checklist C6 · native shell generation / platform adapter

- [ ] **窗口、托盘、Electron listener、导航限制、外链处理、缩放快捷键**都由一个 `ElectronShellGeneration` module 完整拥有；释放走它幂等的 `release()`，调用方不能跨 generation 缓存或单独销毁这些资源。
- [ ] **平台差异集中在启动时选一次的 seam**：`ElectronPlatformStrategy`（Windows / macOS / Linux adapter）负责目录选择、shell 模式切换、更新下载能力、各自菜单/Dock/原生材质。**新加一个平台分支 → 进对应 adapter**，别塞到 generation/runtime 的共享生命周期里。
- [ ] **打包要区分 ASAR vs unpacked**：发布用 `app.asar`，但物理 unpack 依赖（pnpm、node-pty、Windows ACL/native 文件）放在 `app.asar.unpacked`；profile fallback 不能把符号链接指向 Node 解析不了的虚拟 ASAR 路径。

> 改窗口/托盘前先看 `.agents/notes/implemented/architecture/*native-shell-generation-and-platform-adapters*`，那是「为什么这样分层」的决策记录。

---

## Checklist C7 · 打包 / 发布（dist）

- [ ] **打包入口都在 root**：`package:dir`、`dist:mac`、`dist:win`（以及 `dist:mac-smoke`）。它们先 build community-market，再进 dsh-plugin-desktop。
- [ ] **Windows NSIS / macOS DMG 的平台交接**属于 desktop-owned client plugins；不要在 fabric/market 里声明可加载的 DSH 入口（那两个包当前是私有脚手架）。
- [ ] **发布产物要过 packaged-runtime gate**：检查 ASAR 入口 + 物理运行时入口都能解析。

---

## 常见「为什么」（速答）

| 你可能在想 | 答案要点 |
| --- | --- |
| 为什么不直接重新实现 Web UI？ | Desktop 只做壳；官方 Web client 走 loopback carrier，不暴露 Electron IPC 给页面（`docs/why-desktop.md`）。 |
| 为什么有 Yarn + pnpm 两套包管理？ | 外层仓库 = Yarn；固定上游子模块 `deepseek-harness/` **保留它自己的 pnpm workspace**——边界刻意保持，别统一。 |
| 为什么 `.en.md` / `.i18n.yaml` 成对出现？ | 文档是双语的（`.md` + `.en.md`），由 `check:bilingual-docs` 校验一致；改一份要同步另一份。 |
| 手机远程 / Channels 在哪？ | **Roadmap**，当前安装包还没提供这些产品入口，别当成已交付功能去查代码。 |

---

## 学习路径建议（从易到难）

1. **先跑起来**：过 Checklist C1 → `corepack yarn dev`，能看见窗口/托盘。
2. **再认路**：对着第 1 节表格里走一遍每个目录，知道「改 X 该去哪」。
3. **读架构**：`docs/architecture.md`（启动顺序 + Host/Client/native runtime 分工）。
4. **进插件层**：`docs/plugin-development.md` → `dsh-plugin-desktop/docs/plugin-services.md`。
5. **追决策**：`.agents/notes/implemented/**` 里对应日期的 architecture/process note，看「当初为什么这么定」。

> 维护者级别再补：`dsh-plugin-desktop/README.md`（完整 build/run/release/known-limitations）与 `CONTRIBUTING.md`。

---

## 本文覆盖不到、但值得去看的正式文档

- [`../user-guide.md`](../user-guide.md) — 普通用户视角的安装/profile/模式/终端
- [`../faq.md`](../faq.md) — 平台、边界、数据的直接回答
- [`../why-desktop.md`](../why-desktop.md) — Desktop 与官方 Harness 的边界
- [`../architecture.md`](../architecture.md) — 架构总览（本文第 0 节那张图的原版）
- [`../plugin-development.md`](../plugin-development.md) / `dsh-plugin-desktop/docs/plugin-services.md` — 插件怎么写、service contract

> 本学习文档是导航 + Checklist，不是规范来源；以对应正式文档为准。
