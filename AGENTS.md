<!-- BEGIN:nextjs-agent-rules -->

## This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

# agent.md: Mandatory Guardrails for Agentic Development, BisTech EMS

Version: 1.1 | Date: 2026-11-17
Applies to every AI agent and every human working in this repository. These rules are **mandatory**. If a rule blocks you, stop and ask; never work around it.

Related docs: PRD.md, TAD.md, SAD.md, FSD.md, FTL.md (ticket list).

---

## 1. Source of truth

1. Behaviour: FSD.md. Security: SAD.md. Structure: TAD.md. Intent: PRD.md. Work items: FTL.md.
2. On conflict between documents, stop and ask the human owner. Do not guess.
3. Never implement anything on the PRD non-goals list (leave, payroll, documents, check-out, nightly absence job, Redis, file uploads, etc.).

---

## 2. Ask before starting a new task (mandatory)

1. **Before starting implementation of any new ticket, the agent MUST ask the human owner for approval.** Starting means: creating a branch, writing code, creating a migration, or editing tracked files for that ticket.
2. The request must state:
   - Ticket ID and title (from FTL.md)
   - Priority and confirmation that all dependencies are Done
   - Planned files/modules and expected size (split if larger than one PR)
   - Any open question (Q-id) that affects it, and the default assumption it will use
3. Proceed **only after an explicit "yes/approved"**. Silence, an earlier approval for a different ticket, or a general "keep going" is not approval.
4. Reading code, researching, and drafting a plan are allowed before approval; changing the repository is not.
5. If approval is not available (human absent), do the preparatory work, write the plan, and stop. Do not start the ticket.
6. When a ticket is finished, ask again before picking the next one. Do not chain tickets.

---

## 3. One task, one agent (no overlapping work)

Multiple agents must never work on the same ticket or the same files at once.

### Claiming a ticket
1. Pick a ticket from FTL.md whose **dependencies are all Done** and which is **not claimed**.
2. Check for a claim:
   - A remote branch for the ticket already exists (`git ls-remote --heads origin "*/F-041-*"`), or
   - The ticket appears in `CLAIMS.md` with status `in-progress` or `in-review`.
3. After human approval (section 2), claim it by **pushing the ticket branch first** (the push is the atomic lock) and then adding a line to `CLAIMS.md` on that branch:

   ```
   | F-041 | Task creation | agent-name | in-progress | feat/F-041-task-creation | 2026-11-17 |
   ```
4. If the push is rejected because the branch exists, the ticket is taken. Pick another ticket; do not take it over.
5. Status values: `in-progress`, `in-review`, `blocked`, `done`. Update `CLAIMS.md` when status changes.

### Rules while a ticket is claimed
- One ticket has exactly **one owner**. No pair-writing on the same branch. Reviewers comment but do not push feature code to it.
- One agent works on **one ticket at a time**.
- **File overlap**: before editing a file outside your ticket's own module, check `CLAIMS.md` and open branches for conflicts. If another ticket touches the same file (e.g. `packages/contracts`, `app.module.ts`, a shared migration), coordinate through the human owner before editing.
- **Migrations**: only one open migration branch per area at a time; use timestamped, ordered file names and never edit a merged migration.
- **Handoff**: to give a ticket to another agent, set status `blocked`, push work, and get human approval for the reassignment. Never take over silently.
- **Stale claim**: a claim with no commits for 48 hours is flagged to the human; do not steal it.
- Release the claim (status `done`, delete the branch after merge) when the ticket merges.

### Review independence
- A PR must be reviewed by an agent or human **other than its author** before merge.
- Nobody approves their own changes.

---

## 4. Safety guardrails (always on)

### Scope and permissions
- Work only on the approved ticket. No drive-by refactors, dependency upgrades or unrelated fixes; raise them as new tickets.
- Do not change CI, deployment, secrets configuration or permissions definitions unless the ticket says so.
- Never disable, skip or weaken a test, lint rule, boundary rule, CI check, or security control to make something pass.

### Destructive and irreversible actions (forbidden without explicit human approval)
- `git push --force`, `git reset --hard` on shared branches, history rewrites, deleting remote branches you do not own
- Dropping or truncating tables, editing merged migrations, deleting data, or running anything against staging/production databases
- Deleting files outside your ticket's scope
- Installing global packages or changing machine/system configuration

### Secrets and data
- Never commit, print or log secrets, tokens, OTP codes, passwords, Gmail credentials or personal data. Use environment variables validated by the config schema.
- Use synthetic data only. Never use or copy a production dump.
- If you find a committed secret, stop, tell the human, and do not try to hide it in history yourself.

### Project architecture rules (from TAD.md / SAD.md / FSD.md)
- Every controller handler declares `@RequirePermission()` or `@Public()`.
- Organization id comes from the session only; it is a required repository argument.
- Multi-step operations use one `transactionService.run()`; audit rows are written in the same transaction; no email or network call inside a transaction.
- Out-of-scope records return 404; use the exact error codes in FSD.md section 3.
- Modules import only other modules' exported facades. `forwardRef` is banned.
- No raw SQL string concatenation; parameterized queries only.
- Responses use explicit DTOs; never return entities, hashes or tokens.
- Migrations are plain SQL, backward compatible, never run on app start.
- Frontend: no tokens in browser storage; no `dangerouslySetInnerHTML` on user content; dark mode must pass axe contrast.

### Honesty and verification
- Run the ticket's tests and the full CI checks locally before opening a PR. Report real results; never claim a test passed if it was not run.
- If something cannot be verified (e.g. no network, missing credentials), say so in the PR.
- State assumptions explicitly in the PR description (with Q-id where relevant).
- When unsure, ask. Stopping to ask is always acceptable; guessing on security, data or permissions is not.

---

## 5. Git rules

### Branches
- Never commit directly to `main` (or `develop`). All work happens on a ticket branch and merges through a reviewed PR.
- Branch name: `<type>/<ticket-id>-<short-kebab-description>`, e.g. `feat/F-041-task-creation`, `fix/F-044-review-race`.
- One ticket = one branch = one PR. Keep PRs small; split a ticket that is too large (after asking).
- Rebase or merge `main` into your branch regularly; resolve conflicts without discarding others' changes.

### Commits: Conventional Commits (mandatory)

Format:

```
<type>(<scope>): <subject>

<body, optional>

<footer, optional>
```

**Types**

| Type | Use for |
|---|---|
| `feat` | New feature or behaviour |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `build` | Build system, dependencies, packaging |
| `ci` | CI/CD configuration |
| `chore` | Maintenance not touching src or tests |
| `revert` | Reverts a previous commit |

**Scopes** (use the module or area): `identity`, `organization`, `employees`, `attendance`, `teams`, `tasks`, `feedback`, `notifications`, `dashboard`, `audit`, `platform`, `db`, `queue`, `email`, `contracts`, `web`, `ci`, `deps`.

**Rules**
1. Subject: imperative mood, lower-case start, no trailing period, **72 characters or fewer** ("add", not "added" or "adds").
2. Body (when the why is not obvious): wrap at 100 characters; explain what and why, not how.
3. Footer: always reference the ticket: `Refs: F-041`. Use `Closes: F-041` only in the final commit of the ticket.
4. Breaking change: add `!` after the type/scope and a footer `BREAKING CHANGE: <description>`.
5. Add the author as a trailer: `Agent: <agent-name>`.
6. One logical change per commit. Do not mix a feature with unrelated formatting or refactors.
7. Tests for a change go in the same commit or the immediately following `test` commit on the same branch.
8. Migrations get their own commit: `feat(db): add task_assignments table`.

**Examples**

```
feat(tasks): fan out team task to one assignment per member

Create one task_assignments row per active team member inside the
task creation transaction and emit task.assigned after commit.

Refs: F-041
Agent: agent-1
```

```
fix(attendance): exclude holidays from derived absence

Refs: F-031
Agent: agent-1
```

```
test(identity): cover otp attempt exhaustion

Refs: F-015
Agent: agent-2
```

```
feat(contracts)!: rename task status in_review to under_review

BREAKING CHANGE: API enum value in_review is now under_review.

Refs: F-042
Agent: agent-1
```

**Invalid** (reject in review): `update stuff`, `WIP`, `Fixed bug.`, `feat: Added task stuff` (past tense, vague), commits without type or ticket reference.

### Commit hygiene
- Never commit secrets, `.env` files, build output, large binaries, or generated files that are not tracked by convention.
- Do not bypass hooks (`--no-verify`) or signing requirements.
- Commit locally often; push at each meaningful checkpoint so the claim and work are visible.
- Never force-push a branch someone else has pulled; never force-push `main`.
- Do not amend or rebase commits that have been reviewed without telling the reviewer.

### Pull requests
- Title follows Conventional Commits: `feat(tasks): task creation with team fan-out (F-041)`.
- Description includes: ticket ID, what changed, how it was tested (commands and results), acceptance criteria checklist copied from FTL.md and ticked, assumptions/Q-ids, risks.
- CI must be green. The PR needs approval from someone other than the author.
- Merge strategy: squash-merge with a Conventional Commit message, so history stays readable. The squash message must keep the `Refs`/`Closes` footer.
- After merge: update `CLAIMS.md` to `done`, delete the branch, and ask the human before starting the next ticket.

---

## 6. Pre-PR checklist (every ticket)

- [ ] I asked and received approval before starting (section 2)
- [ ] Ticket is claimed only by me; no overlapping branch or file conflict (section 3)
- [ ] All FTL.md checklist items and acceptance criteria for the ticket are met
- [ ] Listed tests (T-xx) exist and pass locally; full CI green
- [ ] No secrets, personal data or debug output committed
- [ ] Every handler has `@RequirePermission()` or `@Public()`; org id from session only
- [ ] Audit and transaction rules from FSD.md followed
- [ ] Commits follow Conventional Commits with `Refs: F-xxx` and agent trailer
- [ ] PR description complete; assumptions and open questions listed
- [ ] Reviewer assigned (not me); `CLAIMS.md` status set to `in-review`

---

## 7. When to stop and escalate to the human

- Documents conflict or a requirement is ambiguous in a way that affects security, permissions or data
- A dependency ticket is not Done or the ticket seems too large for one PR
- Another agent's branch or claim overlaps your files
- A test fails for a reason you cannot explain, or a control would need weakening
- You discovered a secret, a vulnerability, or data loss risk
- Any action in the "destructive and irreversible" list appears necessary

<!-- END:nextjs-agent-rules -->
