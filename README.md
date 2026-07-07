# Agentic-SkillForge
Skill forge refers to skills I find, create, or adapt to my agentic workflows.

## Usage

Skills are authored and stored in this repo, but Claude Code only discovers skills under `.claude/skills/<name>/SKILL.md` in a project. To avoid loading every skill's description into context on every session, keep this repo's skills library separate from `.claude/skills/`:

- Only copy/symlink a skill into `.claude/skills/` when you're actively going to use it in a project.
- Run `/reload-skills` after adding or removing one so the change takes effect.
- Remove it from `.claude/skills/` again (and `/reload-skills`) once you're done, so unrelated sessions aren't carrying skill descriptions they don't need.
