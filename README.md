# vitos-exe/skills

Personal Claude Code skills repository. Custom skills for specialized workflows and automation.

## Quick Start

Install locally in Claude Code:

```bash
/plugin marketplace add vitos-exe/skills
```

Then use the skills by mentioning them or describing tasks they handle.

## Skills

| Skill | Description |
|-------|-------------|
| [atomic-review](./skills/atomic-review/) | Review code changes by replaying a branch diff as a series of atomic commits in a git worktree. Use when reviewing PRs/branches and you want to split changes into smaller, logical units for easier review. |

## Adding a New Skill

1. Create a folder under `skills/<skill-name>/`
2. Add a `SKILL.md` file with frontmatter:
   ```yaml
   ---
   name: skill-name
   description: When to trigger and what the skill does
   ---
   ```
3. Register in `.claude-plugin/plugin.json` by adding the path to `plugins[0].skills`
4. (Optional) Update the Skills table in this README

See CLAUDE.md for development details.

## Structure

```
skills/                    # Skill implementations
.claude-plugin/
  plugin.json             # Plugin registry
CLAUDE.md                 # Development guide
```
