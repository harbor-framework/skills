# Harbor Skills

[![](https://dcbadge.limes.pink/api/server/https://discord.gg/6xWPKhGDbA)](https://discord.gg/6xWPKhGDbA)
[![Docs](https://img.shields.io/badge/Docs-000000?style=for-the-badge&logo=mdbook&color=105864)](https://harborframework.com/docs)

[Agent Skills](https://agentskills.io) for [Harbor](https://github.com/harbor-framework/harbor).

## Skills

| Skill | Description |
|-------|-------------|
| [harbor-task-creator](skills/harbor-task-creator/) | Create Harbor tasks |
| [harbor-adapter-creator](skills/harbor-adapter-creator/) | Build adapters that convert external benchmarks into Harbor tasks |
| [harbor-cli](skills/harbor-cli/) | Complete reference for Harbor CLI commands and flags |

## Usage

To add the Harbor skills, just run the following commands.

**Claude Code**
```
/plugin marketplace add harbor-framework/skills
/plugin install harbor-skills
```

**Codex**
```
$skill-installer install skills from https://github.com/harbor-framework/skills
```

**Cursor**

Install from the marketplace panel in Cursor, or for local testing:
```
ln -s /path/to/harbor-skills ~/.cursor/plugins/local/harbor-skills
```
