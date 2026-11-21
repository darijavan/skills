# Skills

Agent skills for Claude Code and other agents that read `SKILL.md`.

| Skill | What it does |
|---|---|
| [`npm-first-publish`](skills/npm-first-publish/SKILL.md) | Gets a brand-new package onto npm without a token that bypasses 2FA, then sets up trusted publishing (OIDC) from GitHub Actions. It covers staging, `npm trust`, semantic-release pitfalls, recovering a half-finished release, and an error-to-cause table. |

## Install

```bash
npx skills add darijavan/skills --skill npm-first-publish
```

Or copy `skills/npm-first-publish/` into `~/.claude/skills/` (all projects) or `.claude/skills/` (one project).

## Licence

MIT
