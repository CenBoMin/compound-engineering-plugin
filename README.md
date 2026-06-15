# Compound Engineering for Zed

[![Build Status](https://github.com/EveryInc/compound-engineering-plugin/actions/workflows/ci.yml/badge.svg)](https://github.com/EveryInc/compound-engineering-plugin/actions/workflows/ci.yml)
[![npm](https://img.shields.io/npm/v/@every-env/compound-plugin)](https://www.npmjs.com/package/@every-env/compound-plugin)

> 专为 [Zed](https://zed.dev) 编辑器打造的复合工程技能体系。让每一次工程工作都比上一次更轻松。

---

## 🚀 快速开始

**Zed 中启用 Compound Engineering：**

```bash
# 将技能目录复制到 Zed 的 agents 目录
cp -r .agents/skills/* ~/.agents/skills/
```

然后在任意项目的 Zed 对话中通过 `/ce-xxx` 斜杠命令调用。

### 安装检查

在任意项目中运行 `/ce-setup`——它会诊断环境、检查插件版本、引导项目配置。

---

## 🧩 技能总览

当前 `.agents/skills/`（Zed 原生）提供以下 11 个技能：

### 核心工作流

| 技能             | 用途                                                 |
| ---------------- | ---------------------------------------------------- |
| `/ce-brainstorm` | 交互式 Q&A，在编码前想透功能或问题，产出需求规格文档 |
| `/ce-plan`       | 将想法转化为多步骤实施计划，含自动信心检查           |
| `/ce-work`       | 按计划执行工作项——编码、验证、交付，走通质量门禁     |
| `/ce-compound`   | 记录已解决问题到 `docs/solutions/`，让下次迭代更聪明 |

核心闭环：

```
/ce-brainstorm → /ce-plan → /ce-work → /ce-compound
                                                      ↑
                                    (下次迭代更聪明) ←┘
```

### 质量保障

| 技能                | 用途                                                     |
| ------------------- | -------------------------------------------------------- |
| `/ce-code-review`   | 多视角代理代码评审——分级评审者、置信度门控、去重管道     |
| `/ce-debug`         | 系统性根因分析——构建因果链、形成可测试假设、测试先行修复 |
| `/ce-simplify-code` | 简化近期代码变更——并行审查（复用、质量、效率），行为验证 |

### Git 工作流

| 技能                 | 用途                                                             |
| -------------------- | ---------------------------------------------------------------- |
| `/ce-commit`         | 创建高质量的 git 提交——约定感知、敏感文件安全、逻辑拆分          |
| `/ce-commit-push-pr` | 从工作改动到 PR 全流程——三种模式（完整流程/描述更新/仅描述生成） |
| `/ce-worktree`       | 创建 git worktree 用于并行开发，自动处理 `.env` 和 gitignore     |

### 环境与设置

| 技能        | 用途                                             |
| ----------- | ------------------------------------------------ |
| `/ce-setup` | 诊断环境、安装缺失工具、引导项目配置——交互式开箱 |

---

### Zed 目录结构

```
.agents/skills/
├── ce-brainstorm/          # 需求探索与规格文档
│   ├── SKILL.md
│   └── references/
├── ce-code-review/         # 多智能体代码评审
│   ├── SKILL.md
│   └── references/
├── ce-commit/              # 高质量 git 提交
│   └── SKILL.md
├── ce-commit-push-pr/      # PR 全流程
│   ├── SKILL.md
│   └── references/
├── ce-compound/            # 知识沉淀与复用
│   ├── SKILL.md
│   ├── references/
│   ├── assets/
│   └── scripts/
├── ce-debug/               # 系统性根因分析
│   ├── SKILL.md
│   └── references/
├── ce-plan/                # 实施计划生成
│   ├── SKILL.md
│   └── references/
├── ce-setup/               # 环境诊断与配置
│   ├── SKILL.md
│   ├── references/
│   └── scripts/
├── ce-simplify-code/       # 代码简化
│   └── SKILL.md
├── ce-work/                # 工作项执行
│   ├── SKILL.md
│   └── references/
└── ce-worktree/            # 并行开发 worktree 管理
    ├── SKILL.md
    └── scripts/
```

---

## 📦 更多技能（其他平台）

`plugins/compound-engineering/skills/` 包含完整的 35+ 技能目录，涵盖：

- **战略与度量**：`ce-strategy`、`ce-product-pulse`、`ce-compound-refresh`
- **研究与上下文**：`ce-sessions`、`ce-slack-research`、`ce-riffrec-feedback-analysis`
- **开发框架**：`ce-agent-native-architecture`、`ce-dhh-rails-style`、`ce-frontend-design`、`ce-polish`
- **审查与质量**：`ce-doc-review`、`ce-optimize`
- **工作流工具**：`ce-demo-reel`、`ce-promote`、`ce-resolve-pr-feedback`、`ce-update`、`ce-release-notes`、`ce-report-bug`
- **浏览器与测试**：`ce-test-browser`、`ce-test-xcode`
- **自动化实验**：`lfg`、`ce-dogfood-beta`
- **Git 工具**：`ce-clean-gone-branches`
- **内容与协作**：`ce-proof`
- **图像生成**：`ce-gemini-imagegen`

这些技能通过 CLI 转换器（`compound-plugin convert --to <platform>`）分发到 Claude Code、Codex、Gemini CLI 等平台。Zed 端的适配按需进行——当前 `ce-strategy`、`ce-product-pulse`、`ce-sessions` 等高级技能正在陆续移植到 `.agents/skills/`。

详细技能目录参见 [`plugins/compound-engineering/README.md`](plugins/compound-engineering/README.md) 或 [`docs/skills/`](docs/skills/)。

---

## 🔧 仓库开发

```bash
bun install
bun test                    # 完整测试套件
bun run release:validate    # 检查插件/市场一致性
```

本项目包含：

- **`src/`** — Bun/TypeScript CLI，用于跨平台插件转换与安装
- **`plugins/compound-engineering/`** — 主插件（技能、代理、清单）
- **`plugins/coding-tutor/`** — 配套插件
- **`docs/`** — 需求、计划、解决方案和平台规范

详细贡献指南参见 [`AGENTS.md`](AGENTS.md)（此为仓库权威指令文件）。

---

## 🤝 贡献说明

_About Contributions:_ 本项目不接受外部贡献。我只是没有足够的脑力去审查任何东西。欢迎提交 issue 或 PR 来说明建议的修复，但我不会直接合并它们。特别欢迎 bug 报告。抱歉如果这冒犯了任何人，但这是我能以这个速度前进并保持理智的唯一方式。

## 📄 许可证

[MIT](LICENSE)

---

<p align="center">
  <strong>专为 Zed 编辑器用户打造</strong> · 让每一次工程工作都比上一次更轻松
</p>
