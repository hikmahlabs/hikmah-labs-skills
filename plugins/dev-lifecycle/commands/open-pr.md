Wrap up the current branch and open its PR in one shot: checks → cross-model review → commit → push
→ PR → classify → flag.

This is the single "I'm done, open the PR" command. You should not have to run `/preflight` or
`/commit` separately first — this does them. (Those still exist if you want them standalone.)

## Input

$ARGUMENTS — optionally the issue number this closes, and/or `--skip-review` / `--claude-review`
(see "Review"). If omitted, infer it from the branch name
(`<user>/<type>-<issue>-<slug>`); if still unknown, ask.

## Config

Read `.github/dev-workflow.yml` → `repo.*`, `project.number`, `sensitivePaths`, `review.command`,
`commitConvention`. Note that `review.command` is the **fallback** reviewer now — Codex is primary.
See "Review" below.

## Pipeline

1. **PR scope (see "PR scope" below)** — decide whether everything in the working tree belongs in
   one PR or should be split. Substantial + unrelated → split; a handful of minor changes → one PR;
   unsure → ask. Do this before the checks, so the review and the commit cover the same set.
2. **Checks** — run `bunx tsc --noEmit`, `bun run lint`, `bun run test`. If anything fails, STOP,
   show the blockers, and ask whether to fix now or proceed anyway (the user decides). Never run
   `bun run build` (it publishes translations). These run BEFORE the review — they're cheap and
   deterministic, and there's no point spending minutes of review time on a branch that doesn't
   compile.
3. **Cross-model review (see "Review" below)** — a second model reviews the work its author didn't
   write. Runs against the **working tree**, before the commit, so anything fixed lands in the same
   commit instead of a follow-up.
4. **Commit any uncommitted work** — if `git status --porcelain` is non-empty, commit it following
   **the exact same rules as `/commit`**: update `tracker.md` if present, do the HIGH-confidence
   security pass (stop + report if something real turns up), then `git add -A` (include untracked — unless splitting, see "PR scope")
   and write a **Conventional Commit** (`type: brief subject`, 50–72 chars, 3–6 bullets of what
   matters). **No AI attribution of any kind** — see the CLAUDE.md rule. (Invoking `/open-pr`
   authorizes this commit.)
5. **Push the feature branch** (never main): `git push -u origin HEAD`.
6. **Sensitive-path scan** — `git diff --name-only origin/main...HEAD` matched against
   `sensitivePaths`. If any match, OR you have open questions / low confidence about the change,
   this PR is `needs-human-review`.
7. **Compose the PR** — title = the same Conventional-Commit subject (it becomes the squash subject);
   body = the repo PR template filled in: summary, `Closes #<issue>`, ticked change-type + risk
   boxes, and any **post-merge steps** (e.g. a one-off `convex run migrations/x:run`) — deploy itself
   is automatic. Put open questions in "Reviewer notes". **The PR body must contain no AI
   attribution** (no "Generated with Claude", no 🤖, no Co-Authored-By) — write it as a human would.
   ```bash
   gh pr create --base main --head "$(git branch --show-current)" --title "<title>" --body "<body>"
   ```
8. **Classify** — copy the linked issue's `type:*` / `priority:*` / `area:*` labels onto the PR
   (`gh pr edit <pr#> --add-label ...`). If step 6 flagged it, also `--add-label needs-human-review`.
9. **Board** — add the PR to project `number`, then set **BOTH** single-select fields on the card
   (the board has two: the built-in **Status** that the default column view groups by, and the
   custom **Stage**):
   - **Stage** (custom) → **In Review** (or **Needs Human Review** if step 6 flagged it).
   - **Status** (built-in — only Todo / In Progress / Done) → **In Progress**. "In Review" / "Needs
     Human Review" aren't Status options; the work stays In Progress until the PR merges (merge →
     the board's built-in workflow sets Status = Done). Stage→Status mapping: **Backlog/Ready → Todo
     · In Progress/In Review/Needs Human Review → In Progress · Done → Done.**

   Resolve IDs via `gh project field-list <number> --owner <owner> --format json`, then one
   `gh project item-edit ...` **per field** — setting only Stage leaves the card stuck in its old
   Status column. Non-blocking if field resolution genuinely fails.

10. **Report** — print the PR URL and a one-line summary. If it's `needs-human-review`, say so plainly
   so the human knows to look before merging. Merging the PR (squash) is the final human step — it
   auto-deploys and closes the issue; there is no separate ship command.

## PR scope

Settle this FIRST, before the checks and the review — everything downstream depends on it: what gets
reviewed, what gets committed, and what the PR claims to be.

Read `git status --porcelain` and group the changes by what they actually are.

- **One coherent body of work** → one PR. The normal case.
- **Several distinct bodies of work, and they're substantial** → **split**. A feature and an
  unrelated refactor in one PR get reviewed, reverted, and understood as a single unit forever
  after.
- **Several changes, but each is minor** — a small fix here, a tweak there → **one PR**. Splitting
  three-line fixes into three PRs is overhead with nothing to show for it.
- **Not sure** → **ask.** Show the groups and let the user decide.

"Substantial" is about the work, not the line count: a 30-line auth change is substantial, a
200-line regenerated catalog is not. Two large but genuinely related changes still belong together —
the test is whether you can describe them in one sentence without an "and".

**When splitting, never move the user's work around without confirming first:**

1. Show the groups and say which one this PR will carry.
2. Commit that group only — stage its paths explicitly (`git add <paths>`), **not** `git add -A`.
3. Leave the rest uncommitted in the working tree and call it out in the final report so it isn't
   forgotten. Only move work onto another branch if the user asks for it.

## Review

The point is that a change is reviewed by a model that did not write it. Claude writes most of the
work here, so **Codex reviews by default**.

### Locating Codex

The installed plugin path is version-pinned and moves on every upgrade, so resolve it at run time —
newest install first, marketplace source as backup:

```bash
CODEX_COMPANION=$(ls -t \
  ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs \
  ~/.claude/plugins/marketplaces/*/plugins/codex/scripts/codex-companion.mjs 2>/dev/null | head -1)
```

Health probe: `node "$CODEX_COMPANION" status --all --json`. Clean JSON = Codex is reachable.

Do **not** try to call `/codex:review` or `/codex:adversarial-review` — both are
`disable-model-invocation: true`, so a command cannot invoke them. The `codex:codex-rescue` agent is
also explicitly forbidden from calling review. Shelling out to the companion script is the only
route that works from here.

### Running it

Foreground — the pipeline needs the verdict before it can continue:

```bash
node "$CODEX_COMPANION" review --wait --model gpt-5.6-sol
```

**Pick the scope from the branch's actual state — the two scopes do not overlap.** `working-tree`
collects staged + unstaged + untracked but *no commits*; `branch` collects `merge-base..HEAD` but
*no uncommitted work*. Neither alone covers a branch that has both. So, with
`COMMITS_AHEAD=$(git rev-list --count origin/main..HEAD)`:

- **dirty tree, 0 commits ahead** — the normal case here. `--scope working-tree` IS the whole PR.
- **clean tree** — `--base origin/main`, which is the whole PR.
- **dirty tree AND commits ahead** — no single call covers everything. Review the working tree, then
  say plainly which commits were out of scope, and offer a second pass with `--base origin/main`
  after step "Commit". Do not silently report a partial review as a complete one.

**Model ladder** (step down only when the model itself is rejected, never because findings came
back): `gpt-5.6-sol` → `gpt-5.6-terra` → omit `--model` and let Codex choose. `--model` is accepted
by `review` even though the usage banner only documents it under `task`.

**Detect failure from the OUTPUT, not the exit code.** A rejected model exits **0** — verified. It
prints `[codex] Turn failed.` and a review body of `Reviewer failed to output a response.` So treat
the run as failed, and step down the ladder, when any of these hold:

- the output contains `Turn failed` or `Reviewer failed to output a response`
- the review body is empty
- the process exits non-zero

Only a run that produces actual review prose counts as a completed review. Never report "clean"
because a failed run happened to contain no findings.

**Reviewer ladder** — always name which tier actually ran; never let a downgrade pass silently:

1. **Codex** on the best model it accepts.
2. **Claude self-review** via the local `review.command` (`/review`) if Codex is unreachable — CLI
   missing, health probe failing, or auth rejected. Say plainly that Codex was unavailable and that
   this is the author reviewing their own work, which is weaker.
3. If `/review` fails too, **stop**. Do not open a PR with no review at all.

### Flags

- `--skip-review` — skip the review entirely (trivial change, or Codex is known-down and you don't
  want to wait). Must be stated in the final report.
- `--claude-review` — invert the direction, for when Codex wrote the code. Claude cannot change its
  own model mid-command, so this spawns a subagent with `model: fable` rather than "switching to
  Fable".

### Acting on findings

**Clean** → one line naming the reviewer and model, then continue.

**Findings** → STOP. Present them as a numbered list — file:line, severity, and the actual claim —
and let the user decide each one. **Never fix silently.**

The single exception is a finding that is **both**:

- objectively breaking — a crash, data loss, an auth/permission hole, a secret in the diff, or
  something that fails the build; **and**
- confirmed by reading the code yourself, not taken on the reviewer's word.

Style, preference, architecture and "consider…" are always presented, never applied. When something
is auto-fixed, say what and why, and re-run `tsc` / `lint` / `test` before continuing.

Findings the user declines are dropped — they don't go into the PR body.

A rejected finding is not automatically a wrong one. Codex sees the diff without the conversation
that produced it, so it will sometimes flag a deliberate decision. Present those with the context
that explains them rather than passing the claim through unexamined.

## Rules

- Push only the current feature branch; never push to `main`.
- **Substantial + unrelated work does not share a PR.** Split it; a pile of minor changes can ride
  together. Unsure → ask, never guess. See "PR scope".
- Commit messages AND the PR body follow `/commit`'s style and carry **zero AI attribution**.
- Always include `Closes #<issue>` so merge auto-closes the issue and advances the board.
- If the checks have blockers or the security pass finds something, stop and let the user decide.
- **The work's author is never its only reviewer.** Codex reviews Claude's work by default; if Codex
  is unreachable, say so out loud rather than quietly self-reviewing as if nothing changed.
- **Never auto-fix a review finding** unless it is both objectively breaking and confirmed by reading
  the code. Everything else is the user's call.
