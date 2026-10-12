# skills

个人维护的 [Agent Skills](https://agentskills.io) 集合。每个技能是一个包含 `SKILL.md` 的目录，供编码 Agent 在匹配场景时自动加载。

支持 OpenCode、Claude Code、Codex、Cursor 等近 80 个 Agent，通过 [`skills` CLI](https://github.com/vercel-labs/skills) 一条命令安装。

## 技能列表

| 技能 | 说明 |
| --- | --- |
| [writing-interface-docs](writing-interface-docs/SKILL.md) | 编写对外接口文档。面向看不到本模块源码的对接方，只写契约不写实现。触发场景：写接口文档、对接文档、报文文档、API 文档、"让别的模块能接上"。 |

## 安装

在项目根目录（或任意目录）执行：

```bash
npx skills add zzlcoding/skills
```

CLI 会自动检测本机已安装的 Agent 并交互式询问装到哪些。不带参数时按**项目级**安装，装到当前目录。

### 常用变体

```bash
# 先看看仓库里有哪些技能，不安装
npx skills add zzlcoding/skills --list

# 只装指定技能
npx skills add zzlcoding/skills --skill writing-interface-docs

# 装到指定 Agent（可写多个）
npx skills add zzlcoding/skills -a opencode -a claude-code

# 全局安装：所有项目都能用
npx skills add zzlcoding/skills --skill writing-interface-docs -g

# 非交互安装，适合 CI / 脚本
npx skills add zzlcoding/skills --skill writing-interface-docs -g -a opencode -y

# 复制而非软链（文件系统不支持符号链接时）
npx skills add zzlcoding/skills --copy
```

## 安装位置

| 范围 | 参数 | OpenCode 路径 | 适用场景 |
| --- | --- | --- | --- |
| 项目级 | 默认 | `<项目>/.agents/skills/` | 随项目提交，团队共享 |
| 全局级 | `-g` | `~/.config/opencode/skills/` | 跨所有项目可用 |

其他 Agent 的目录不同，例如 Claude Code 为 `.claude/skills/`（项目）与 `~/.claude/skills/`（全局）。

默认用**软链**方式安装：CLI 维护一份源文件，各 Agent 目录里只有软链。单一来源、便于更新。若软链不可用，加 `--copy` 改为独立副本。

## 后续维护

```bash
# 查看已安装的技能
npx skills list

# 更新到最新版本（-y 自动识别项目级/全局级）
npx skills update writing-interface-docs

# 卸载
npx skills remove writing-interface-docs
```
