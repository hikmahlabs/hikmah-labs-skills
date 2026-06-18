# dev-lifecycle

The per-feature command layer for the **Dev Lifecycle** — a flexible, project-adjustable way to take
work from idea → classified GitHub issue → worktree → PR → squash-merge, with AI doing the heavy
lifting and a human at the gates (merging, anything sensitive).

Pairs with **[`hikmahlabs/project-template`](https://github.com/hikmahlabs/project-template)**, which
ships the repo-side layer (issue forms, PR template, labels, CI, the Projects v2 board, and the
`bootstrap-project.sh` script). This plugin is the *agent* layer; the template is the *GitHub* layer.

## Install

```
/plugin marketplace add hikmahlabs/plugins
/plugin install dev-lifecycle
```

## The flow

```
/intake "<idea>"  →  /start <#>  →  build  →  /open-pr  →  "merge it"
   (file issue)      (begin work)           (preflight+review+commit+PR)   (auto-deploys)
```

You type ~3 commands per feature, or just say it in plain English.

## Commands

| Command      | What it does                                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------------------------ |
| `/intake`    | Turns an idea/bug/chore into a classified GitHub issue (type/priority/effort/area) on the board.             |
| `/triage`    | Reviews untriaged issues, confirms classification, marks them `ready`.                                        |
| `/start`     | `/start <issue#>` — loads the issue in the current worktree, assigns you, moves it to **In Progress**.        |
| `/preflight` | Quick local gate: typecheck + lint + tests + the local `/review`. _(Optional — `/open-pr` runs it.)_         |
| `/open-pr`   | One shot: preflight → commit → push → opens a PR from the template, `Closes #`, copies labels, **auto-flags sensitive diffs** `needs-human-review`. |

Merging the PR (squash) is the final human step — it auto-deploys and closes the issue. There's no
separate "ship" command.

## Configuration

Every command reads **`.github/dev-workflow.yml`** in the target repo (created by the template /
bootstrap): repo coordinates, the Projects board number, the classification taxonomy, the
`sensitivePaths` globs that trigger `needs-human-review`, and the local review command. The same
plugin behaves correctly on every project because only that file changes.

## Conventions enforced

- Conventional-Commit titles; squash merges.
- **No AI attribution** anywhere (commits, PR bodies, issues) — written as a human would.
- Reviews are **local only**; sensitive diffs (auth, payments, schema/migrations, destructive ops)
  always get a human before merge.

See the template repo's `CONTRIBUTING.md` for the full methodology.
