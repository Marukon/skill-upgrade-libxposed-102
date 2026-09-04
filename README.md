# Skill: Upgrade to LibXposed API 102 (`upgrade-libxposed-102`)

[![API](https://img.shields.io/badge/LibXposed-API%20102-brightgreen)](https://github.com/libxposed/api)
[![Target](https://img.shields.io/badge/Target-LSPosed%201.9.3%2B-blue)](https://github.com/LSPosed/LSPosed)
[![License](https://img.shields.io/badge/License-Apache%202.0-orange)](LICENSE)
[![Type](https://img.shields.io/badge/Agent%20Skill-Antigravity%20%7C%20Claude%20%7C%20Cursor-purple)](#)

这是一个面向 AI 编程智能体（AI Coding Agents / Harnesses，如 Google Antigravity, Claude Code, Cursor, Cline 等）的专业技能包（Skill）。

当智能体接入本技能后，只需接收到类似 **“将该 Xposed 模块升级到 102”**、**“迁移到现代 LSPosed API”** 等指令，便能自动按规范执行全流程代码重构、元数据生成与性能优化，无需人工每次提供 API 规范链接或文档。

---

## 📦 技能能力一览

- **自动依赖迁移**：将旧版 `de.robv.android.xposed:api`（54-93）迁移至 `io.github.libxposed:api:102.0.0`。
- **现代元数据生成**：规范化生成 `META-INF/xposed/` 目录下的 `module.prop`、`java_init.list` 与 `scope.list`。
- **Hook 模型重构**：将老旧反射及 `XC_MethodHook` 改写为现代链式拦截器（`Hooker` / `Chain`），并配置热重载（Hot Reload）。
- **性能深度优化**：自动对 `isEnabled` 等系统高频调用注入优化模版（Android 14+ `StackWalker` 惰性遍历、底层零开销直通嗅探与动态栈帧定位），彻底杜绝 GC 卡顿掉帧。
- **构建环境自适应**：自动修复 Java 17/21 与旧版 Gradle 冲突（如 `major version 61` 异常），适配 Android SDK 34。

---

## 🚀 安装指南 (如何让 AI 智能体识别并安装)

### 方式 1：全局安装（所有项目通用，推荐）

#### Windows (PowerShell):
```powershell
# 克隆至 Antigravity / Gemini CLI 全局配置目录
git clone https://github.com/<your-username>/skill-upgrade-libxposed-102.git "$HOME\.gemini\config\skills\upgrade-libxposed-102"
```

#### Linux / macOS (Bash):
```bash
# 克隆至全局配置目录
git clone https://github.com/<your-username>/skill-upgrade-libxposed-102.git "$HOME/.gemini/config/skills/upgrade-libxposed-102"
```

---

### 方式 2：项目本地安装（随项目 Git 仓库共享给团队）

在你的 Android 模块工程根目录下执行：

```bash
# 作为 Git 子模块引入到项目 agents 目录
git submodule add https://github.com/<your-username>/skill-upgrade-libxposed-102.git .agents/skills/upgrade-libxposed-102
```

---

### 方式 3：Claude Code / 通用 Agent 安装

```bash
# 安装到通用 agent skills 目录
mkdir -p .claude/skills && git clone https://github.com/<your-username>/skill-upgrade-libxposed-102.git .claude/skills/upgrade-libxposed-102
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
