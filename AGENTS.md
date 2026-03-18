# harbor-skills — Agent Quick Reference

Agent Skills for the [Harbor](https://github.com/harbor-framework/harbor) framework. Skills follow the [Agent Skills open standard](https://agentskills.io).

## Skill Format Quick Reference

Each skill lives in `skills/<skill-name>/` with a required `SKILL.md`:

**YAML frontmatter** (required):
- `name`: kebab-case, max 64 chars, must match directory name
- `description`: max 1024 chars, no `<` or `>` characters

**Optional frontmatter:**
- `license`, `compatibility`, `metadata`, `allowed-tools`

**Markdown body:** Instructions loaded on trigger. Keep under 500 lines.

**Bundled resources** (optional subdirectories):
- `references/` — loaded as needed (detailed tables, examples)
- `scripts/` — executed, not loaded into context
- `assets/` — used in output (templates, images)

## Harbor Documentation Map

| Resource | Location |
|----------|----------|
| Harbor repo | https://github.com/harbor-framework/harbor |
| Harbor docs | https://harborframework.com/docs |
| Harbor cookbook | https://github.com/harbor-framework/harbor-cookbook |
| Task config model | `src/harbor/models/task/config.py` |
| Verifier | `src/harbor/verifier/verifier.py` |
| CLI entry points | `src/harbor/cli/` |
| Agent base class | `src/harbor/agents/base.py` |
| Adapter system | `src/harbor/adapters/` |

## Conventions

- Each skill directory has a `SKILL.md` (required) plus optional `references/` subdirectory
- Keep `SKILL.md` body < 500 lines; put detailed tables and walkthroughs in `references/`
- Descriptions should be slightly "pushy" — err toward triggering rather than not
- No `scripts/` — Harbor already has CLI tools for scaffolding and validation (`harbor tasks init`, `harbor adapters init`)
- The `name` frontmatter field must match the directory name exactly

## Common Pitfalls

- Don't exceed 1024 chars in the `description` frontmatter field
- Don't use `<` or `>` in the description field (breaks YAML parsing)
- Don't duplicate content between `SKILL.md` body and `references/` — body should point to references
- Don't write skills that wrap simple CLI commands — skills should teach complex multi-step workflows
- Don't put Harbor version-specific details in skills without noting the version

## Validation & Testing

- Run the CI workflow locally or push a change to `skills/**` to trigger validation
- CI runs validation on every push and PR (`.github/workflows/validate-skills.yml`)
- Use the skill-creator workflow to test skills with real prompts before publishing
