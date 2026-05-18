# Execute-LandingPrompt

Kiro Agent Skill — 执行单个 LandingPrompt 文件并自动回流结果到 status.yaml。

## 功能

- 读取 LandingPrompt 目录的 README.md 获取项目上下文（source_root、coding standards、LP 序列）
- 执行指定的 LandingPrompt 文件中的实施步骤
- 自动将执行结果同步回项目的 status/status.yaml（PST 回流）
- 支持任意项目路径，项目无关设计

## 调用方式

`
Skill Execute-LandingPrompt + <path-to-landing-prompt>.md
`

## 前置条件

- LandingPrompt 目录必须包含 README.md（含 YAML front-matter 配置 source_root）
- 目标项目需遵循 project-state-tracker 工作流结构

## 安装

将本文件夹放置于 ~/.kiro/skills/Execute-LandingPrompt/。

## 许可证

MIT
