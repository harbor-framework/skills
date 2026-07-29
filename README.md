# Harbor Skills

[![](https://dcbadge.limes.pink/api/server/https://discord.gg/6xWPKhGDbA)](https://discord.gg/6xWPKhGDbA)
[![Docs](https://img.shields.io/badge/Docs-000000?style=for-the-badge&logo=mdbook&color=105864)](https://harborframework.com/docs)

[Agent Skills](https://agentskills.io) for [Harbor](https://github.com/harbor-framework/harbor).

## Skills

| Skill | Description |
|-------|-------------|
| [create-adapter](skills/create-adapter/) | Scaffold and implement Harbor benchmark adapters |
| [create-task](skills/create-task/) | Create Harbor tasks, environments, solutions, and verifiers |
| [harbor-exec](skills/harbor-exec/) | Compile inputs into tasks and run Harbor Exec workflows |
| [publish](skills/publish/) | Publish Harbor tasks and datasets to the registry |
| [rewardkit](skills/rewardkit/) | Build Harbor task verifiers with Reward Kit |
| [upload-parity-experiments](skills/upload-parity-experiments/) | Upload Harbor parity experiment results to Hugging Face |

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
