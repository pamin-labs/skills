# Pamin Labs skills

Agent skills that Pamin Labs' repositories share, in the open [Agent Skills](https://agentskills.io) format, usable by Claude Code, Codex and other agents that read `SKILL.md`.

| Skill | What it holds |
| --- | --- |
| [`pamin-engineering`](skills/pamin-engineering/SKILL.md) | Engineering rules for any language (`references/core.md`) and per-language references (Go, Rust, web, Workers), a performance guide and a rule-scan procedure |

## Install

```sh
npx skills add pamin-labs/skills --skill pamin-engineering
```

Inside a repository this installs into `.agents/skills/` and records the source commit in `skills-lock.json`. Commit both; restore or check them with `npx skills experimental_install --frozen-lockfile`.

## Changes

A skill changes here, never in an installed copy. Pamin Labs' repositories receive each change as a pull request.

## Licence

[Apache-2.0](LICENSE).
