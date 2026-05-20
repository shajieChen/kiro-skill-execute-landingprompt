# Execute-LandingPrompt

Kiro Agent Skill — 执行单个 LandingPrompt 文件并自动回流结果到 `status.yaml`（PST 回流）。

## 功能

- 读取 LandingPrompt 目录 README.md 获取项目上下文（source_root、coding standards、LP 序列）
- 按步骤执行指定 LandingPrompt 中的实施任务
- 执行完成后自动同步状态到 `status/status.yaml`
- **Result 持久化** — Handoff Markdown 自动归档至 `pst_root/Result/`，保留完整执行历史
- 项目无关设计，支持任意 PST 管理的项目

## 调用方式

```text
Skill Execute-LandingPrompt + <path-to-landing-prompt>.md
```

## 前置条件

- LandingPrompt 目录必须包含 README.md（含 YAML front-matter 配置 `source_root`）
- 目标项目需遵循 project-state-tracker 工作流结构

## 安装

将本文件夹放置于 `~/.kiro/skills/Execute-LandingPrompt/`。

## 许可证

MIT
