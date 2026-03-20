---
name: skill-distribution
description: |
  Use when publishing, sharing, or distributing Claude Code skills.
  Triggers: "put skills on github", "publish", "clawhub publish",
  "share skill", "set up repo for skills", "open source".
---

# Skill Distribution

Three paths for distributing Claude Code skills, from private to public.

## Decision Matrix

| Criterion | Local | GitHub | ClawHub |
|-----------|-------|--------|---------|
| Setup effort | None | Low | Medium |
| Reach | Only you | Anyone with link | Entire OpenClaw ecosystem |
| Updates | Manual copy | git pull | clawhub update |
| Security review | Your responsibility | Community | VirusTotal + community |
| Best for | Testing/developing | Sharing + portfolio | Maximum reach |

## Path 1: Local Only

- Skills in `~/.claude/skills/` or project `.claude/skills/`
- No extra files needed
- Backup via private git repo
- Sufficient for personal use

## Path 2: GitHub

Repo structure:
```
{skill-name}/
├── README.md
├── LICENSE (MIT standard)
├── .gitignore
├── CHANGELOG.md
├── {skill-folder}/
│   ├── SKILL.md
│   ├── references/
│   └── gotchas.md
```

Installation: copy skill folder to `~/.claude/skills/`.

## Path 3: ClawHub

- Push to github.com/openclaw/skills via PR
- Standard AgentSkills spec (SKILL.md with YAML frontmatter)
- VirusTotal scan runs automatically
- Install: `clawhub install {author}/{skill-name}`

## Language Strategy

| Content | Public Skills | Niche NL Market |
|---------|-------------|-----------------|
| SKILL.md description/triggers | English | English |
| SKILL.md body | English | Dutch OK |
| README | English | Bilingual |
| Gotchas/references | English | English |

**Rule:** Descriptions and triggers always in English for discoverability.

## .gitignore Template

```
credentials/
.env
node_modules/
*.db
__pycache__/
.DS_Store
```
