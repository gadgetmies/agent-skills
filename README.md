# agent-skills

Personal collection of authored agent skills, installable **one at a time** with
[`skills`](https://github.com/vercel-labs/skills) (`npx skills`).

## Install a skill

```sh
# user-level (global), a single skill by name:
npx skills add gadgetmies/agent-skills -s <skill-name> -g

# list what's in this repo without installing:
npx skills add gadgetmies/agent-skills -l
```

## Skills

| Skill | What it's for |
|---|---|
| [`commit-and-test-discipline`](skills/commit-and-test-discipline/SKILL.md) | Commit & test conduct when making changes in a repo: `wip:` checkpoints and relevant-subset test runs during work; atomic history + full suite green at the merge boundary (none for documentation-only changes); state-isolated, parallel-safe, pyramided tests; scratch tests for checking the change, committed tests only for end-state behaviour. |

## Layout

```
skills/
  <skill-name>/
    SKILL.md
```

One folder per skill, each with its own `SKILL.md`, so `npx skills add … -s <skill-name>`
installs exactly that skill and nothing else.
