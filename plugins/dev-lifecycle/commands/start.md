Begin work on an issue in the current Conductor worktree: load its context, claim it, move it to In Progress.

## Input

$ARGUMENTS — the issue number to start (e.g. `142`). If empty, ask which issue.

## Config

Read `.github/dev-workflow.yml` for `repo.owner`, `repo.name`, `project.number`, `branchConvention`.

## Process

1. **Sanity-check the branch.** Run `git branch --show-current`. If it's the default branch
   (`main`), STOP — tell the user to create a Conductor workspace for this issue first, then re-run.
   Do **not** rename or create branches here (that's Conductor's job; renaming is forbidden).
   - The convention (FYI, not enforced): name the Conductor workspace so its branch reads like
     `branchConvention`, e.g. `<user>/feat-142-wali-otp`. If the current branch doesn't match,
     just note it — don't change it.
2. **Load the issue:**
   ```bash
   gh issue view <n> --repo <owner>/<name> --json number,title,body,labels,assignees
   ```
   Print the title, the **acceptance criteria**, the area/priority labels, and call out any
   `needs-human-review` flag or sensitive scope up front.
3. **Claim it:**
   ```bash
   gh issue edit <n> --repo <owner>/<name> --add-assignee @me
   ```
4. **Move board Stage → In Progress** (resolve IDs via `gh project field-list`/`item-edit`; if the
   item isn't on the board yet, `gh project item-add <number> --owner <owner> --url <issue-url>`
   first). If field resolution fails, note it and continue — it's non-blocking.
5. **Summarize the plan** in 2–4 bullets so we start aligned, then begin implementing.

## Rules

- This command sets up context only — it never writes feature code on its own. After it runs,
  proceed with normal interactive development under the `CLAUDE.md` guardrails.
- Keep the acceptance criteria visible; they are the definition of done for `/preflight` and the PR.
