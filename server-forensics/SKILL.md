---
name: server-forensics
description: Server forensics skill for analyzing disk images (.E01, .raw, .qcow2, etc.) and simulation environments via SSH-MCP. Use this skill whenever the user mentions server forensics, disk image analysis, forensic investigation of servers, CTF server challenges, incident response involving server images, analyzing compromised servers, recovering data from server disk images, or connecting to forensic simulation environments. Triggers on any mention of server disk images, forensic analysis of Linux/Windows servers, or connecting to forensic simulation environments.
---

# Server Forensics Analysis

## Core Principles

1. **Simulation AND disk image — always both, always parallel.** For every question, analyze the simulation (via SSH-MCP) and the disk image simultaneously. If one path is blocked, immediately switch to the other. Never spend more than 2-3 attempts on one approach before pivoting. Kali forensic tools (The Sleuth Kit, foremost, photorec, etc.) are available for direct disk image analysis.
2. **Plan first, act second.** After reading all questions, you MUST classify them into groups, draft a forensic strategy for each group (data sources, tools, cross-validation approach, fallback plans), and present the complete plan to the user. Do NOT start analysis until the user approves.
3. **Report per question, not per group.** After solving EACH individual question, immediately append the answer and a brief solution method to the forensic report. Never batch report updates.
4. **Ask for help, don't spin.** If stuck after switching between simulation and disk image 2-3 times, STOP and report to the user: what you're looking for, what you tried on both sources, and 2-3 proposed next steps. Wait for user input before continuing.

## Workflow

### Phase 0: Gather Essential Information

Before doing anything else, ask the user for these four items:

1. **Kali VM** — IP address, SSH port, username, password, and root password (this is where the disk image is stored)
2. **Simulation** — IP address, SSH port, username, and password (the live server being investigated)
3. **Disk image path** — the full path to the image file on the Kali VM (e.g., `/home/user/case.e01`)
4. **Questions file** — the path to the questions/case file (usually a .txt on the user's local machine)

Wait until all four items are provided. If any are missing, ask again.

### Phase 1: Connect and Mount

Connect to the Kali VM via SSH-MCP first, then mount the disk image read-only. Use `file <image>` to identify the format, then mount accordingly (.E01 → ewfmount, .raw/.dd → loop mount, .qcow2 → qemu-nbd/guestmount). Record the mount path.

Connect to the simulation via SSH-MCP. Verify with a simple command (`hostname && uname -a`). If the simulation is unavailable, report immediately and wait — do not proceed with only one source.

### Phase 2: Quick System Identification

From the simulation, capture ONLY what's needed to understand the server's role and tech stack. Run these in parallel:

- OS and kernel (`uname -a`, `cat /etc/os-release`)
- Hostname and IP
- Key technologies — check specifically for: Docker (`docker ps`), Kubernetes (`kubectl get all`), PVE (`pvecm status`), Node.js, Python, PHP, nginx/apache
- Listening services (`ss -tlnp`)
- Users with login shells and their home directories

From the disk image, cross-validate: check `/etc/` configs, service files, web roots, and home directories.

**Skip** hardware specs, disk capacity, memory, CPU, and installed package lists — these are noise for forensic analysis.

Briefly summarize: what is this server, what does it run, what's the tech stack.

### Phase 3: Classify, Plan, Present — MANDATORY STOP

1. Read the questions file provided by the user.
2. Classify every question into themed groups (e.g., system identity, user accounts, network services, intrusion traces, data recovery, website reconstruction, malware/persistence).
3. For EACH group, draft a forensic strategy:
   - Which data sources to use (simulation paths + disk image locations)
   - Key tools and techniques
   - How to cross-validate between simulation and disk image
   - Fallback plan if primary approach fails
4. Present the complete plan to the user — all groups, all questions, proposed order, strategies per group.
5. **WAIT for explicit user approval. Do not execute anything before approval.**

### Phase 4: Execute One Question at a Time

For each question (following the approved plan order):

1. **Attack from both sides simultaneously.** Start simulation commands AND disk image analysis at the same time. For the disk image, use TSK tools directly on the raw image if mounting is impractical — `mmls` for partition layout, `fls` for file listing, `icat` for file extraction.
2. **Cross-validate.** Compare findings from both sources. If they disagree, investigate and resolve the discrepancy.
3. **Pivot on failure.** If simulation returns nothing useful → go straight to the disk image binary data. If the disk image is hard to parse → leverage the live simulation. Don't wait, don't retry the same failed command.
4. **Record immediately.** After solving the question, append to the report:

```
### QN: [Question text]
**Answer**: [Concise answer with key evidence]
**Method**: [2-3 sentences: what was done, which source(s) provided the answer, key file/command]
```

5. Move to the next question. Report group completion to the user only after all questions in a group are done.

### Phase 5: Website Reconstruction

If the server hosts a website:
1. Identify the web stack from server configs (nginx/apache) and source files on the disk image.
2. Locate web root, extract all source files, find database files or dumps.
3. Reconstruct locally — start the web server and database.
4. Use **chrome-devtools** on the host machine to access `http://<server-ip>:<port>` and verify the site works.
5. Document the URL and key pages in the report.

## Handling Blockers

When stuck on a question:

1. **Switch source immediately.** Simulation not working? Read the disk image with TSK/fls/icat. Disk image can't mount? Use the running simulation. Try alternative tools for the same goal.
2. **After 2-3 failed attempts across both sources, STOP.** Do not keep trying variations of the same approach.
3. **Report to user:**
   - What question you're on and what you're trying to find
   - What you tried: simulation commands run + disk image analysis attempted
   - Specific errors or blockers encountered
   - 2-3 concrete next steps you propose
4. **Wait for user response.** Do not proceed until the user gives direction.

## Report Template

```markdown
# [Server Name] Forensics Report

## System Overview
- **OS**: ...
- **Hostname**: ...
- **IP**: ...
- **Key Technologies**: (Docker / K8s / PVE / Node.js / PHP / ...)
- **Services**: (nginx, mysql, ssh, ...)

## Findings

### Q1: [Question]
**Answer**: ...
**Method**: ...

### Q2: [Question]
**Answer**: ...
**Method**: ...
```

Update this file after EVERY question. Create it after Phase 2, then append findings as you solve each question.
