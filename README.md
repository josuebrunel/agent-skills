# agent-skills

Personal [Claude Code](https://docs.claude.com/en/docs/claude-code) skills — reusable instructions Claude loads automatically when a task matches their trigger conditions.

## Skills

| Skill | Description |
|---|---|
| [`dev-principles`](dev-principles/SKILL.md) | Cross-stack development principles — idiomatic code, security, DRY, testing, structured logging, error handling, config management, and standing up local dependencies via Docker Compose. Applies to any language, not just Go. |
| [`go-stack`](go-stack/SKILL.md) | Default backend stack for new Go services and features: Echo v5, Templ, EzAuth, Goose (isolated `NewProvider`), RiverQueue, HTMX, and DaisyUI v5. Builds on `dev-principles` and requires a Makefile + docker-compose.yml. |
| [`web-design`](web-design/SKILL.md) | Visual design system and new-app workflow — mesh gradients, glassmorphism, WCAG 2.1 AA accessibility, and a two-phase (aesthetic HTML demo → real build) process. |

## Usage

Drop a skill directory into `~/.claude/skills/` (user-level) or `<project>/.claude/skills/` (project-level) and Claude Code will pick it up automatically based on its `SKILL.md` frontmatter description.
