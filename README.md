# Claude Plugins Workshop Marketplace

A plugin marketplace for workshop participants to demonstrate and contribute Claude Code skills.

## Install this marketplace

```
/plugin marketplace add efimovvo/claude-plugins-workshop
```

Then install a plugin:

```
/plugin install my-template-example-skills@workshop-skills
```

## Available plugins

| Plugin | Skills | Description |
|---|---|---|
| `my-template-example-skills` | `my-template-skill-creator` | Template skill for creating and iterating on new skills |

## Contributing

Want to add your own skill to this marketplace? See [CONTRIBUTING.md](./CONTRIBUTING.md).

## Structure

```
.claude-plugin/
  marketplace.json        ← marketplace manifest
skills/
  <skill-name>/
    SKILL.md              ← skill instructions and metadata
```

## Resources

- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills)
- [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [anthropics/skills](https://github.com/anthropics/skills) — official Anthropic skills repo
