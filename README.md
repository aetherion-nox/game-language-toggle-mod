# 游戏一键切换语言 Mod Skill

一个用于 Codex 等支持 `SKILL.md` 的智能体的技能，指导制作和修复通过热键在游戏内切换显示语言的 Mod，包括当前对话和历史记录的刷新。

技能关注可靠配对原始文本、只刷新显示层、字体与编码、输入处理以及存档和界面回归。本仓库提供跨游戏适配方法，不包含现成游戏补丁或游戏资源；即时切换的可行性取决于具体游戏。

## 安装

将本仓库完整克隆到 Codex 的技能目录（或将整个目录复制到该目录）：

```bash
git clone https://github.com/aetherion-nox/game-language-toggle-mod.git ~/.codex/skills/game-language-toggle-mod
```

在 Windows 上，该目录通常为 `%USERPROFILE%\.codex\skills\game-language-toggle-mod`。安装后重新启动 Codex，使技能列表刷新。

## 使用

在 Codex 中说明游戏、目标语言、需要切换的界面和热键偏好，并提出制作或修复语言切换 Mod 的需求；也可明确指定 `game-language-toggle-mod`。实际实施需要相应游戏文件、版本信息和必要授权。

已要求实施时，技能连续完成范围内的制作、安装与可执行验证；需要用户游玩、实体按键或目视检查的项目，完成后再集中确认。可以直接修改游戏文件，修改前备份受影响文件并提供回退方法；用户明确提出的隔离或原文件限制继续遵守。

## 文件

- [SKILL.md](SKILL.md)：技能入口与核心工作流程。
- [运行时接入](references/runtime-switching.md)：运行时切换与刷新。
- [安全与验收](references/safety-and-validation.md)：安装、回退及验证。
- [agents/openai.yaml](agents/openai.yaml)：Codex 技能显示信息。

本技能用于切换显示语言；两种语言同时显示的场景请使用 `game-bilingual-patch` 技能。

