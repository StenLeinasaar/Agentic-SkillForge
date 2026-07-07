---
description: Authoring guide for writing effective SKILL.md files and reference material
paths:
  - skills/**
---

# Skill authoring best practices

Condensed from Anthropic's Skill authoring guidance. Read this before writing or substantially revising a `SKILL.md` or its `references/`.

## Core principles

- **Be concise.** SKILL.md competes with conversation history for context. Assume Claude already knows general concepts (what a PDF is, what a library is) — only include what Claude doesn't already know: your specific conventions, commands, file layouts, gotchas.
- **Match freedom to fragility.** Open-ended judgment calls → high-freedom prose ("check X, consider Y"). A preferred-but-flexible pattern → pseudocode/template with params. A fragile, must-follow-exactly sequence → an exact script/command with "do not modify."
- **Test across model tiers if the skill will be used broadly.** What's enough detail for Haiku may over-explain for Opus, and vice versa.

## Frontmatter

- `name`: ≤64 chars, lowercase letters/numbers/hyphens only, no XML tags, no "anthropic"/"claude". Prefer gerund form (`processing-pdfs`) or a clear noun/action phrase. Avoid vague names (`helper`, `utils`).
- `description`: ≤1024 chars, third person, non-empty, no XML tags. State **what it does** and **when to use it** with concrete trigger terms — this is the only thing loaded into context before the skill fires, and it's how Claude chooses between 100+ skills. "Helps with documents" is useless; "Extracts text/tables from PDFs, fills forms, merges documents — use when working with PDF files or forms" works.

## Structure (progressive disclosure)

- Keep `SKILL.md` body under ~500 lines. Push depth into `references/*.md`, linked directly from `SKILL.md`.
- **Keep references one level deep from SKILL.md.** SKILL.md → reference.md is fine; reference.md → details.md → deeper.md causes Claude to skim with `head` instead of reading fully. Flatten it.
- For reference files over ~100 lines, put a table of contents at the top so a partial read still shows the full scope.
- Organize multi-domain skills by domain (`reference/finance.md`, `reference/sales.md`), not by arbitrary numbering (`doc1.md`, `doc2.md`) — this also keeps irrelevant context out of a given task.
- Use MCP tools by fully-qualified name (`ServerName:tool_name`), never bare tool names.

## Workflows and feedback loops

- For multi-step tasks, give an explicit ordered checklist Claude can copy and check off — this prevents skipped validation steps.
- For quality-critical output, build in a validate → fix → re-validate loop (a lint/test script, or a checklist to re-compare against) rather than a single unchecked pass.
- For fragile batch/destructive operations, use plan → validate → execute → verify: have Claude write an intermediate plan file, validate it with a script, then apply it. Catches errors before they touch real state.

## Content rules

- No time-sensitive claims ("before August 2025, use..."). If legacy behavior matters, put it in a clearly labeled "old patterns" section instead of the main flow.
- Pick one term per concept and use it everywhere (always "API endpoint", never a mix of "endpoint"/"URL"/"route").
- Don't offer a menu of equivalent options ("use pypdf, or pdfplumber, or..."). Give one default, name an escape hatch only if a real alternate case exists.
- Scripts should handle errors and provide sane fallbacks rather than punting to Claude with a bare `open(path).read()`. Justify any constant (timeout, retry count) with a one-line comment — no unexplained magic numbers.
- State required packages explicitly and confirm they're actually available in the target execution environment; don't assume.

## Anti-duplication reminder

This repo already has a stricter rule for *whether* to add a new skill at all — see [skill-anti-duplication.md](skill-anti-duplication.md). Apply that check first; use this file for *how* to write the skill once a new one (or a rework) is justified.

## Checklist before shipping a skill

- [ ] Description names both the trigger and the action, in third person
- [ ] SKILL.md body < 500 lines; deeper material moved to `references/`
- [ ] All reference links are one level deep from SKILL.md
- [ ] No time-sensitive statements outside a clearly labeled legacy section
- [ ] Terminology is consistent throughout
- [ ] Scripts (if any) handle their own errors and document any constants
- [ ] Tested against a real task, not just read for plausibility
