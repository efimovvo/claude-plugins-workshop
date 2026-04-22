# Contributing a Skill

Anyone can contribute a skill to this workshop marketplace by opening a pull request.

## Steps

### 1. Fork & clone

Fork this repo on GitHub, then clone your fork:

```bash
git clone https://github.com/<your-username>/claude-plugins-workshop.git
cd claude-plugins-workshop
```

### 2. Create your skill directory

```
skills/
  your-skill-name/
    SKILL.md
```

Minimum `SKILL.md`:

```markdown
---
name: your-skill-name
description: What this skill does and when to use it (1-3 sentences)
---

# Your Skill Name

[Instructions for Claude here]
```

### 3. Register in marketplace.json

Add your skill to `.claude-plugin/marketplace.json` under `plugins`:

```json
{
  "name": "your-plugin-name",
  "description": "Short description",
  "source": "./",
  "strict": false,
  "skills": ["./skills/your-skill-name"]
}
```

### 4. Open a PR

Push your branch and open a pull request against `main`.

## Skill naming

Use kebab-case. Workshop participants are encouraged to add a personal prefix (e.g., `alice-code-reviewer`, `bob-data-analyst`) to avoid conflicts.

## Resources

- [Skill Writing Guide](https://github.com/anthropics/skills/blob/main/template/SKILL.md)
- [Agent Skills spec](https://github.com/anthropics/skills/tree/main/spec)
