---
description: Anti-duplication guardrail for adding or changing skills
paths:
  - skills/**
---

# Adding or changing a skill

Before creating a new `skills/<name>/` directory or significantly reworking an existing one:

1. **Check for overlap first.** List the existing skills under `skills/*/SKILL.md` and read their `description` frontmatter. If the new idea's trigger conditions overlap an existing skill's territory, that overlap is a signal to extend the existing skill rather than add a near-duplicate.
2. **Prefer extending over duplicating.** If the idea overlaps an existing skill, edit that skill's `SKILL.md` or add to its `references/` instead of adding a new directory.
3. **Follow the established anatomy** when a new skill is genuinely justified: a `SKILL.md` with `name`/`description` frontmatter — the description carries the trigger conditions, so be explicit and concrete about when to invoke — plus a `references/` subfolder for material too deep to inline in `SKILL.md` itself. `skills/kubernetes-mentor/` and `skills/ansible-mentor/` are the canonical examples to match in shape and depth. For the details of writing that SKILL.md well (naming, description wording, freedom levels, progressive disclosure, workflows/feedback loops), see [skill-authoring-best-practices.md](skill-authoring-best-practices.md).
4. **Never duplicate content between skills.** If two skills would need the same reference material, link to the other skill's file instead of copying it.

New skills are authored here in `skills/` (the library) and only staged into `.claude/skills/` when actively in use for a project — see `README.md` for that workflow.
