# CLAUDE.md

Guidelines for authoring and contributing skills in this repo.

## Repository purpose

This repo packages security review workflows for Uniswap v4 hooks as [Claude Code](https://claude.ai/claude-code) skills. Each top-level directory (other than docs/meta files) is a self-contained skill that Claude Code can discover and run. I used this article to generate the skills: https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive 

## Skill layout

Every skill must follow this structure:

```
<skill-name>/
├── SKILL.md          # Required. Frontmatter + skill body.
└── references/       # Optional. Files loaded into context on demand.
    ├── 01-*.md
    └── 02-*.md
```

Optional folders:

- `scripts/` — Executable helpers (bash, Python) for deterministic checks
- `assets/` — Templates or fixed outputs the skill emits

## SKILL.md format

YAML frontmatter is required at the top:

```markdown
---
name: skill-name
description: When to trigger and what the skill does. Both pieces matter — the description is the only thing Claude sees before deciding to consult the skill. Be specific about triggering contexts.
---

# Skill Name

Body in Markdown.
```

Constraints:

- Keep the body under ~500 lines. Push depth into `references/` files and link to them.
- Be explicit about triggering. Skills under-trigger by default — name the user phrases and contexts that should activate the skill.
- Lead the body with the workflow Claude should run, not background reading.

## Reference files

- Numbered prefixes (`01-`, `02-`, …) preserve a reading order when relevant.
- Each reference file should be self-contained — Claude may load only one.
- Link every reference back to its primary source (audit report, blog post, spec) so the human reviewer can dig deeper.

## Attribution

This skill set draws on public security research from many authors. When adapting material from a specific source, credit it in the reference file and in the README. If a reference file is a substantial port of someone else's work, get their explicit permission before publishing.

## Contributing

PRs welcome. Before opening one:

1. Make sure the skill triggers — write 2–3 example prompts in the PR description and confirm Claude Code picks up the skill on each.
2. Keep findings actionable — every heuristic should be phrased as something a reviewer can check, not abstract advice.
3. Link primary sources. Audit reports, post-mortems, and protocol docs over secondary commentary.