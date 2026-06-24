# SF Skills — Claude Code Skills

Claude Code skill for Salesforce development workflows.

---

## Skills

| Skill | Purpose |
|---|---|
| `org-refresh-to-local` | Force-sync Salesforce metadata from a live org to local when the SF CLI retrieve silently skips changed files |

---

## Prerequisites

| Requirement | Details |
|---|---|
| Claude Code | CLI installed (`claude --version` to verify) |
| Salesforce CLI (`sf`) | v2.x or later (`sf --version` to verify) |
| Authenticated org | Run `sf org list` to verify |

---

## Install

```bash
git clone https://github.com/rsrinivasan33/sf-skills.git
cp -r sf-skills/skills/* ~/.claude/skills/
```

---

## Usage

```
/org-refresh-to-local
```

Claude will diagnose the retrieve issue, clear corrupt source tracking indexes if needed, and fall back to the SOAP Metadata API to force-sync the component to disk.
