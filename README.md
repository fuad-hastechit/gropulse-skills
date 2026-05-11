# gropulse-skills

Internal Claude Code plugin — reusable implementation skills for Gropulse Shopify apps.

## Skills

| Skill | Slash command | What it does |
|-------|--------------|-------------|
| `appointment-booking` | `/appointment-booking` | Add free Growth Strategy Call booking to any Gropulse app |

## Install

1. Add the marketplace source:
```
/plugin marketplace add fuad-hastechit/gropulse-skills
```

2. Install the plugin:
```
/plugin install gropulse-skills@gropulse
```

## Adding more skills

1. Create `skills/<skill-name>/SKILL.md` with `name` and `description` frontmatter
2. Create `.claude/commands/<skill-name>.md` for the slash command
3. Push to main — plugin updates on next Claude Code session
