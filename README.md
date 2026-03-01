# Claude Skills

A collection of custom skills for [Claude Code](https://claude.ai/code).

## Skills

### disk-clean - Disk Deep Clean Assistant

A smart disk cleanup skill that works on both **Windows** and **macOS**.

**Trigger phrases** (say any of these in Claude Code):

> 清理磁盘 / 清理C盘 / 磁盘空间不够 / 空间不足 / 硬盘满了 / disk clean / 盘快满了

**What it does:**

- Auto-detects your OS (Windows / macOS), no questions asked
- Dynamically scans your machine for cleanable caches
- Covers: system temp, browsers, IM apps (WeChat, QQ, DingTalk, Slack, Teams...), dev tools (npm, pip, gradle...), and more
- Shows a categorized, size-sorted list before touching anything
- Merges all admin/UAC prompts into one single pop-up
- Never touches: chat history, received files, bookmarks, passwords, Desktop, Documents

After each run, the skill logs cleanup history and learns your preferences for next time.

---

## Installation

Copy any `.md` file from this repo into your Claude Code commands directory:

**Windows:**
```
C:\Users\<YourName>\.claude\commands\
```

**macOS / Linux:**
```
~/.claude/commands/
```

Then restart Claude Code. The skill is ready to use.

---

## Usage Example

```
You:     磁盘快满了帮我清理一下
Claude:  正在扫描，请稍候…
         ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         📊 扫描完成 | C盘剩余 28 GB / 共 256 GB
            可清理：约 4.7 GB
         ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         A1. 系统临时文件     1.2 GB
         B1. Chrome缓存       800 MB
         C1. 微信缓存         600 MB
         ...
         [一次确认 → 自动清理 → 汇报结果]
```

---

## License

MIT
