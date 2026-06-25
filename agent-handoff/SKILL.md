---
name: agent-handoff
description: Generate structured agent handoff documents in markdown format. Use this skill whenever the user asks to create a handoff document, 交接文档, 移交文档, work transfer doc, task handoff, or context transfer between agents. Also trigger when the user says things like "summarize what you've done for the next agent", "create a doc so another agent can continue", "write a handoff note", "帮我写个交接文档", "把这个任务转交给其他人", or any request involving preserving task state for another AI agent to continue the work. Also trigger for security work handoffs: 渗透测试交接, 漏洞挖掘报告移交, 应急响应交接, 取证分析移交, 代码审计报告, 逆向分析笔记, 红蓝对抗总结, IOC/攻击链/利用链 handoff. The output is always a .md file with 6 fixed sections.
---

# Agent Handoff Document Generator

Generate a handoff document from conversation context so the receiving agent can be productive immediately.

## Output template

```markdown
# [Task Name] - 工作交接文档

> 首次创建: YYYY-MM-DD HH:MM | 最近更新: YYYY-MM-DD HH:MM | 数据来源: 对话上下文

## 1. 任务全局状况和最终目标
## 2. 任务当前进度
## 3. 当前遇到的问题
## 4. 已经尝试过但失败的方案
## 5. 推荐尝试的方案
## 6. 下一步要干什么
```

If a section truly has no information, write "暂无" instead of inventing content.

**Sensitive content warning:** If the conversation involves vulnerability details, internal network info, credentials, PII, or forensic evidence, prepend this at the top of the document:
```
> ⚠️ 本文档包含敏感安全信息，请控制分发范围
```

## Section guide

**1. 任务全局状况和最终目标** — The big picture a newcomer needs.

General: project context, end goal, success criteria, stakeholders, deadlines.

Security contexts:
- Pentest: scope of authorization, target systems/networks, test type (black/grey/white box), rules of engagement
- Incident response: incident type (ransomware/intrusion/data exfil/etc.), blast radius, severity, timeline origin
- Forensics: evidence source (disk image/memory dump/network capture), hash values, legal/compliance constraints, chain of custody status
- Code audit: audit scope, codebase language/framework, audit standard (OWASP ASVS/CWE/etc.)

**2. 任务当前进度** — Completed items with file paths, in-progress work, completion estimate (e.g., 60%), key artifacts produced. Be file-aware.

Security contexts, additionally include:
- Vulnerabilities found with severity/CVSS/file locations
- Attack surface enumerated and entry points identified
- Evidence collected with file hashes
- Exploit chain status and confirmed exploitability

**3. 当前遇到的问题** — Split into blocking (with error messages, reproduction steps, code locations) and non-blocking.

Security-specific blocking examples:
- WAF/IDS bypass failures, EDR behavioral detection, anti-debug/anti-analysis blockers
- Unknown encryption schemes, undocumented protocols, anti-sandbox behavior in samples
- Missing forensic artifacts, encrypted evidence, access/permission restrictions

**4. 已经尝试过但失败的方案** — For each failed attempt, use this exact format:
```
- **方案名 (关键词)**: 一句话描述做了什么
  - 失败原因: 具体为什么不行
  - 关键发现: 从中学到了什么（避免下一个人踩坑）
```
This is the most valuable section — it prevents the receiver from wasting time on dead ends.

Security contexts may include: failed exploit payloads, ineffective bypass techniques, incorrect forensic hypotheses, tools that triggered detection.

**5. 推荐尝试的方案** — Priority-ranked (高/中/低), with rationale, expected challenges, and references.

Security contexts, additionally consider:
- Alternative attack vectors or exploitation paths
- Different toolchain combinations (e.g., swap Burp for Caido, swap impacket for crackmapexec)
- Privilege escalation or lateral movement paths to explore

**6. 下一步要干什么** — Three horizons: immediate (today/tomorrow), short-term (this week), items needing external confirmation.

Security contexts:
- Urgent: immediate fix/block/isolate actions (e.g., patch this vuln, isolate compromised host)
- Short-term: further recon/exploitation/evidence preservation
- External dependencies: waiting on threat intel, vendor patches, client authorization, legal sign-off

## How to generate

1. **Check for existing handoff documents first.** Use Glob to search for `handoff-*.md` in the current working directory. If one exists, read it and update it — preserve existing content, refresh the dynamic sections (progress, problems, next steps), and update the "最近更新" timestamp. If none exists, create a new file.

2. **Extract everything from conversation history.** Pull out goals, file paths, error messages, function names, approaches discussed, and next steps mentioned. Do not ask the user follow-up questions — just work from what's available.

3. **Be specific with actual details from context.** Use real file paths, error text, function names:
   - Bad: "修复登录 bug"
   - Good: "修复 `auth/login.ts:42` 中 token 刷新时抛出的 `JWTExpiredError`"
   - Bad: "发现了一个漏洞"
   - Good: "`api/auth.go:87` 中 JWT `none` 算法绕过漏洞（CVSS 7.5），可未授权访问任意用户数据"

4. **Adapt to the domain.** Detect the primary domain from conversation context — general dev, pentest, forensics, incident response, code audit, reverse engineering — and adjust section weighting accordingly. Security-heavy context gets more security detail; general dev stays general.

5. **Save as `handoff-<brief-slug>.md`** in the current working directory. Tell the user the file path and offer to adjust any section. If updating an existing document, use the same file path.
