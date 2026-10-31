# Skill: Upgrade to LibXposed API 102 (`upgrade-libxposed-102`)

[![API](https://img.shields.io/badge/LibXposed-API%20102-brightgreen)](https://github.com/libxposed/api)
[![Target](https://img.shields.io/badge/Target-LSPosed%201.9.3%2B-blue)](https://github.com/LSPosed/LSPosed)
[![Android](https://img.shields.io/badge/Android-9.0--16%20(ColorOS%2016)-orange)](https://developer.android.com)
[![License](https://img.shields.io/badge/License-Apache%202.0-orange)](LICENSE)
[![Type](https://img.shields.io/badge/Agent%20Skill-Antigravity%20%7C%20Claude%20%7C%20Cursor-purple)](#)

这是一个面向 AI 编程智能体（AI Coding Agents / Harnesses，如 Google Antigravity, Claude Code, Cursor, Cline 等）的专业技能包（Skill）。

当智能体接入本技能后，只需接收到类似 **“将该 Xposed 模块升级到 102”**、**“迁移到现代 LSPosed API”** 等指令，便能自动按规范执行全流程代码重构、元数据生成与性能优化，无需人工每次提供 API 规范链接或文档。

---

## 📦 技能能力一览

- **自动依赖迁移**：将旧版 `de.robv.android.xposed:api`（54-93）迁移至 `io.github.libxposed:api:102.0.0`，适配 Java 17 与 Android SDK 34+。
- **现代元数据生成**：规范化生成 `META-INF/xposed/` 目录下的 `module.prop`、`java_init.list` 与 `scope.list`。
- **系统服务级 Hook**：提供 `system_server` 专用入口 (`onSystemServerStarting`)、系统 Context 双重安全捕获与延迟初始化范式。
- **跨进程特权隔离防护**：提供针对 Android 14/15/16（包括 ColorOS 16 / HyperOS 等定制 ROM）下的 `Binder.clearCallingIdentity` 特权提升范式，根除系统级 Hook 常见权限拒绝异常（`Permission Denial`）。
- **零常驻配置热同步**：提供基于 `Settings.Global` 配合 `ContentObserver` 的免常驻、零功耗、抗杀后台的无感实时热重载方案，彻底替代在现代 Android 下失效的 `XSharedPreferences`。
- **模块状态自检测 (Self-Hooking)**：在模块加载生命周期中精准识别自身应用，通过优先级 Hook 实现优雅的模块激活状态自检测。
- **Hook 模型重构 & 反射工具库**：将老旧反射及 `XC_MethodHook` 改写为现代链式拦截器（`Hooker` / `Chain`），并提供轻量零依赖的反射辅助类（替代已废弃的 `XposedHelpers`）。
- **性能深度优化**：针对 `isEnabled`、视图绘制等系统高频调用，自动注入优化模板（Android 14+ `StackWalker` 惰性遍历、底层零开销直通嗅探与动态栈帧定位），彻底杜绝 GC 卡顿掉帧。
- **CI/CD 与官方分发仓库集成**：内置提交至 LSPosed 官方模块仓库（`Xposed-Modules-Repo`）的自动化 Actions 规范，解决跨仓库推送 403 权限覆盖陷阱、JKS 机密管理与文档免跑 CI 策略。

---

## 🚀 安装指南 (如何让 AI 智能体识别并安装)

### 方式 1：全局安装（所有项目通用，推荐）

#### Windows (PowerShell):
```powershell
# 克隆至 Antigravity / Gemini CLI 全局配置目录
git clone https://github.com/Marukon/skill-upgrade-libxposed-102.git "$HOME\.gemini\config\skills\upgrade-libxposed-102"
```

#### Linux / macOS (Bash):
```bash
# 克隆至全局配置目录
git clone https://github.com/Marukon/skill-upgrade-libxposed-102.git "$HOME/.gemini/config/skills/upgrade-libxposed-102"
```

---

### 方式 2：项目本地安装（随项目 Git 仓库共享给团队）

在你的 Android 模块工程根目录下执行：

```bash
# 作为 Git 子模块引入到项目 agents 目录
git submodule add https://github.com/Marukon/skill-upgrade-libxposed-102.git .agents/skills/upgrade-libxposed-102
```

---

### 方式 3：Claude Code / 通用 Agent 安装

```bash
# 安装到通用 agent skills 目录
mkdir -p .claude/skills && git clone https://github.com/Marukon/skill-upgrade-libxposed-102.git .claude/skills/upgrade-libxposed-102
```

---

## 💡 使用方法

安装完成后，在与 AI 智能体（如 Antigravity / agy / Claude 等）对话时，直接下达指令即可触发：

> “请阅读当前项目，将该 Xposed 模块升级到 LibXposed API 102 现代规范，并优化性能。”

智能体会自动检索到本技能的描述，动态激活并读取 `SKILL.md`，全自动完成迁移。

---

## 📂 仓库目录规范

```text
skill-upgrade-libxposed-102/
├── README.md               # 本项目说明
├── SKILL.md                # 根目录放置一份，保证各类 Agent 扫描器均可直接识别
├── skills/
│   └── upgrade-libxposed-102/
│       └── SKILL.md        # 标准 Antigravity / Claude 技能目录
└── LICENSE
```

---

## 📄 开源协议
Apache License 2.0
