# Skill

自定义 Claude Code Skill 合集。每个 skill 是一个独立目录，内含描述触发规则和输出规范的 `SKILL.md` 文件。

将本仓库克隆到本地后，在 Claude Code 中通过 `/config` → "Skills" 页签 → "Add skill directory" 加载对应技能目录即可生效。

## 已有技能

| Skill | 用途 |
|-------|------|
| [agent-handoff](./agent-handoff) | 从对话上下文生成结构化工作交接文档，包含任务目标、进度、遇到的问题、失败方案和后续建议。支持中英文触发，也覆盖安全类工作场景（渗透测试、应急响应、取证分析等）。 |
| [server-forensics](./server-forensics) | 服务器取证分析技能，支持磁盘镜像（E01、raw、qcow2 等）分析和仿真环境联动。通过 SSH-MCP 同时接入仿真环境和 Kali 取证虚拟机，双路径交叉验证。适用场景：服务器取证、磁盘镜像分析、CTF 服务器挑战、入侵应急响应。 |
| [bruteforce-login](./bruteforce-login) | 登录爆破脚本生成器。通过 chrome-devtools 分析目标登录页（表单结构、验证码类型、JS 加密逻辑），自动生成带进度持久化、验证码识别、断点续跑的 Python 爆破脚本。支持 Playwright 和 DrissionPage 两种驱动。 |

## 技能结构

```
<skill-name>/
  SKILL.md          # 必需。YAML frontmatter + 指令正文
```

`SKILL.md` frontmatter 关键字段：
- `name` — skill 标识符（kebab-case）
- `description` — 触发匹配说明，Claude 用这段文字来判断何时调用该技能

## 贡献技能

需要新建或修改技能时，直接在 Claude Code 会话中说「帮我创建一个 skill，功能是…」等指令，Claude 会自动调用 `/skill-creator` 引导完成。
