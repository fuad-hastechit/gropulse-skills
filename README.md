# gropulse-skills

Internal Claude Code plugin — reusable implementation skills for Gropulse Shopify apps.

## Skills

| Skill | Slash command | What it does |
|-------|--------------|-------------|
| `translation` | `/translation` | Add full multilanguage support (i18next, DB-persisted language, auto-detection, language switcher) |
| `appointment-booking` | `/appointment-booking` | Add free Growth Strategy Call booking to any Gropulse app |

## Install

### Via skills.sh (recommended)

```bash
npx skills add https://github.com/fuad-hastechit/gropulse-skills
```

Install a specific skill only:

```bash
npx skills add https://github.com/fuad-hastechit/gropulse-skills --skill appointment-booking
npx skills add https://github.com/fuad-hastechit/gropulse-skills --skill translation
```

### Via Claude Code plugin marketplace

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
