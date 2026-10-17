---
name: "specforge"
description: "Builds complex software end to end with Claude Code: full-stack apps, SaaS, desktop agents, cloud deploys with SSL and CI/CD. Sizes the work, plans, gets plans independently reviewed, creates the subagents it needs, builds test-first with human gates."
---

# SpecForge

Turns a one-line request ("build me a full-stack e-commerce app with a desktop agent, SSL, deployed on AWS") into a working, verified, deployed product, using the workflow Anthropic recommends for Claude Code. It also creates the project's own CLAUDE.md, permissions, hooks and subagents, sized to the project.

## Core principles (why this workflow exists)

1. **Context is the scarce resource.** Performance drops as the context window fills. Keep CLAUDE.md lean, push large reading (logs, unfamiliar code, docs) into subagents that return summaries, and start fresh sessions between milestones with a written handoff file.
2. **Explore, then plan, then code, then verify.** Never write code before a plan exists and has been approved. Use plan mode for planning.
3. **Plans get an independent review.** The Claude that wrote a plan does not grade it. A separate reviewer with fresh context reviews it as a skeptical staff engineer.
4. **Verification is the definition of done.** A change is done only when tests pass, the running app behaves correctly, and there is evidence (test output, HTTP responses, screenshots). Code on disk is not the same as code running: restart processes before testing.
5. **Re-plan, don't patch.** If implementation goes sideways twice, stop, go back to plan mode, and re-plan.
6. **Humans own one-way doors.** Anything irreversible, costly or public waits for explicit approval (see One-way doors).
7. **Never fabricate.** Look up current library versions, APIs and cloud service behavior before relying on them. If unsure, check the docs or the real code; don't guess.
8. **Deterministic rules go in hooks and permissions, not prose.** CLAUDE.md is advisory; hooks and permission rules are enforced.

## Phase 0: Intake and sizing

1. Restate the request in one paragraph, then classify its size:

| Size | Example | Process |
|---|---|---|
| S | bug fix, small feature in one area | Plan mode → code → test. Skip spec and agents. |
| M | new feature across 2–3 layers | Short spec + plan + plan review + build loop. |
| L | new app or service | Full process, 2–4 subagents. |
| XL | multi-component product (web + backend + desktop agent + infra) | Full process, 4–6 subagents, milestones of 1–2 days each. |

2. List what only the human can provide: cloud accounts and credentials, domain/DNS access, payment provider keys, code-signing certificates, app store or OAuth consoles, budget, deadline.
3. Ask clarifying questions **in one batch** (use AskUserQuestion when available), max about 6, only ones that change architecture, cost or legal exposure. Typical ones:
   - Who are the users, and expected scale (users, orders, traffic)?
   - Stack preference, or free choice?
   - Hosting/cloud and region; existing account?
   - Payments provider and countries/currencies?
   - What exactly the "agent" does (for any desktop/background agent), and which OS?
   - Deadline and budget for cloud services.
   For anything not asked, choose a sensible mainstream default and record it in `docs/DECISIONS.md` so the user can overturn it.
4. Write `docs/SPEC.md`: goal, users, features (must/should/later), non-functional requirements (security, performance, availability, compliance), integrations, out of scope, open questions, assumptions.

## Phase 1: Explore

- **Existing codebase:** use the built-in Explore subagent (or a custom read-only explorer) to map structure, conventions, entry points, tests and risky areas. It returns a summary, not file dumps.
- **New project:** research current stable versions and official setup guides for each chosen framework and cloud service (web search). Record versions in `docs/DECISIONS.md`.

## Phase 2: Architecture and milestone plan

Write `docs/ARCHITECTURE.md` (components, data model, APIs, auth, deployment topology, a diagram in Mermaid) and `docs/PLAN.md`.

Plan rules:
- **Milestone 0 is a walking skeleton**: repo, CI running tests, a trivial end-to-end path (UI → API → DB) deployed to a staging environment. This flushes out infra risk on day one.
- Then **vertical slices** (each one a user-visible capability through every layer), riskiest first.
- Each milestone lists: scope, files/modules touched, tests to write, **"proven when"** acceptance criteria that can be checked by running something, risks, rollback.
- Mark every one-way door in the plan.

## Phase 3: Independent plan review (gate)

1. Spawn the `plan-reviewer` subagent. Give it only `docs/SPEC.md`, `docs/ARCHITECTURE.md` and `docs/PLAN.md`, not your reasoning, so its review is independent.
2. It returns APPROVE, APPROVE WITH CHANGES, or REJECT plus problems with fixes. Revise and re-review. After 2 rounds without APPROVE, show the user the disagreement and let them decide.
3. Present to the user: the plan summary, the reviewer's verdict, the one-way doors, and what you need from them. **Wait for explicit approval before writing code.**
4. The user may also run the review in a second, separate Claude session for full independence. Support that: tell them which files to give it.

## Phase 4: Set up the harness

Create these files, show them to the user as one diff, and wait for approval.

### 4a. CLAUDE.md (root, under about 80 lines)

Include only what Claude would otherwise get wrong:
```markdown
# <Project>
One-paragraph purpose. Specs: @docs/SPEC.md  Plan: @docs/PLAN.md  Progress: @docs/PROGRESS.md

## Commands
- Install / dev / test / lint / typecheck / build: exact commands

## Rules
- Plan before coding; wait for approval. Re-plan if stuck twice.
- Write failing tests first for new behavior.
- Restart running processes before testing a change.
- Never read, print or commit secrets; use the secret manager / .env (git-ignored).
- Never run one-way-door actions (prod deploy, prod migrations, DNS, live payments, sending real email). Prepare them and ask.
- Commit only when the user asks.

## Conventions
- Only non-obvious ones (error handling pattern, folder layout, naming).

## Gotchas
- Grows over time: add one line each time Claude makes a repeatable mistake.
```
Add small nested CLAUDE.md files per major area (`frontend/`, `backend/`, `infra/`, `agent/`) for area-specific commands and conventions. Never paste whole docs into CLAUDE.md; import them with `@path`.

### 4b. .claude/settings.json (enforced rules)

Adapt commands to the stack:
```json
{
  "permissions": {
    "allow": ["Bash(npm run test:*)", "Bash(npm run lint:*)", "Bash(pytest:*)", "Bash(terraform plan:*)", "Bash(terraform validate:*)"],
    "ask": ["Bash(git push:*)", "Bash(docker push:*)", "Bash(npm publish:*)"],
    "deny": ["Read(./.env)", "Read(./.env.*)", "Read(./**/secrets/**)", "Bash(terraform apply:*)", "Bash(terraform destroy:*)", "Bash(aws * delete*:*)", "Bash(rm -rf /*)"]
  },
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write", "hooks": [{ "type": "command", "command": "<fast formatter + linter for changed files>" }] }
    ]
  }
}
```
- Denying `terraform apply` means the human runs it after reading the plan output.
- Hooks must be fast (seconds). Put full test suites in CI, not in hooks.
- If the project has a critical invariant (two functions that must stay identical, a schema that must match), add a fast test for it and run it in the hook.

### 4c. Decide which subagents to create

A subagent earns its place only if it gives at least one of: **isolated context** (reads a lot, returns a little), **independence** (must not share the author's reasoning, i.e. reviewers), or **parallelism** (independent work running at the same time). Do not create one agent per layer by default.

Decision rules:

| Create | When |
|---|---|
| `plan-reviewer` | Always for M and above |
| `code-reviewer` | Always for M and above |
| `security-reviewer` | Auth, payments, PII, file uploads, public APIs, or a desktop agent with system access |
| `infra-reviewer` | Any IaC, cloud deployment, CI/CD, DNS/SSL |
| `ui-verifier` | Any user-facing UI (checks real rendering via browser screenshots) |
| `migration-reviewer` | Relational DB with schema changes after launch, or data that must not be lost |
| custom explorer | Large existing codebase where the built-in Explore agent isn't enough |

Start with what the table requires (usually 2–5). Add a new subagent later only when you notice the same specialized task repeating 3 or more times. State in `docs/DECISIONS.md` which agents were created and why.

Write each to `.claude/agents/<name>.md`. Reviewers get read-only tools. Templates:

```markdown
---
name: plan-reviewer
description: Independent skeptical review of SPEC/ARCHITECTURE/PLAN before any code is written. Use after every new or revised plan.
tools: Read, Grep, Glob, WebSearch
model: opus
---
You are a skeptical staff engineer. You did not write this plan. Read the files you are given.
Check: requirements coverage; riskiest parts scheduled first; walking skeleton first; each milestone has runnable "proven when" criteria; security and data-loss risks; one-way doors identified; unrealistic estimates; simpler alternatives; unverified assumptions about libraries or cloud services.
Output: VERDICT (APPROVE / APPROVE WITH CHANGES / REJECT), then problems most serious first, each with a concrete fix. Do not edit files.
```

```markdown
---
name: code-reviewer
description: Reviews the current diff in fresh context for correctness, tests, and adherence to CLAUDE.md. Use after each implementation step and before ending a milestone.
tools: Read, Grep, Glob, Bash
model: sonnet
---
Run `git diff` (and `git diff --staged`) to see the change. Read CLAUDE.md and the milestone in docs/PLAN.md.
Check: does it meet the milestone's "proven when"; logic errors and edge cases; missing or weak tests (tests that cannot fail); error handling; consistency with conventions; dead code; anything that breaks another module.
Run the tests and report the real result. Output problems most serious first with file:line and a fix. Do not edit files.
```

```markdown
---
name: security-reviewer
description: Security review of auth, payments, data handling, secrets and attack surface. Use on milestones touching those areas and before any production release.
tools: Read, Grep, Glob, Bash
model: opus
---
Check against OWASP Top 10 and the project's threat surface: authN/authZ on every endpoint (IDOR), input validation, injection, secrets in code or logs, token storage, CSRF/CORS, rate limiting, payment webhook signature verification and idempotency, PII minimisation, dependency vulnerabilities (run the audit tool), and for desktop agents: least privilege, signed updates, safe IPC, no remote code execution paths.
Output: findings by severity with file:line, exploit scenario, and fix. Do not edit files.
```

```markdown
---
name: infra-reviewer
description: Reviews IaC, CI/CD, networking, DNS/SSL and deployment plans before anything is applied. Use before every terraform apply or pipeline change.
tools: Read, Grep, Glob, Bash
model: opus
---
Read infra code and any `terraform plan` output. Check: resources destroyed or replaced unexpectedly; public exposure (security groups, buckets); IAM least privilege and OIDC instead of long-lived keys in CI; secrets in a secret manager; TLS everywhere (ACM or Let's Encrypt) with HTTP→HTTPS redirect; backups and restore tested; health checks, logs, alarms; cost surprises; a documented rollback.
Output: BLOCKERS, then warnings, then suggestions. Do not apply anything.
```

```markdown
---
name: ui-verifier
description: Verifies UI changes by running the app and inspecting real rendered pages. Use after any UI milestone.
tools: Read, Grep, Glob, Bash
model: sonnet
---
Start the app (or use the running one), open the affected pages with the available browser tool or Playwright, take screenshots at desktop and mobile widths, exercise the main flows, and read console errors.
Report: what works, what is broken (with screenshot paths and console errors), accessibility basics (labels, contrast, keyboard). Do not edit files.
```

```markdown
---
name: migration-reviewer
description: Reviews database migrations for data safety before they run anywhere shared.
tools: Read, Grep, Glob
model: opus
---
Check: reversible or with a written rollback; no data loss (drops, type narrowing); locks on large tables; backfills batched; default values; indexes; order relative to the code deploy (expand → migrate → contract). Output: SAFE / UNSAFE with reasons and fixes.
```

### 4d. Progress and handoff files

- `docs/PROGRESS.md`: current milestone, done, in progress, next step, blockers, last verified test results. Update at the end of every step so any fresh session can resume from it alone.
- `docs/DECISIONS.md`: dated one-line decisions with the reason (stack, versions, agents created, defaults chosen).

## Phase 5: Build loop (repeat per milestone)

1. **Fresh context.** Start a new session or `/clear`. Read CLAUDE.md, PROGRESS.md and the milestone from PLAN.md.
2. **Mini-plan in plan mode** for this milestone: files, tests, steps. For risky milestones (auth, payments, data model, infra), run `plan-reviewer` on it too.
3. **Tests first.** Write tests for the milestone's "proven when" criteria. Run them and show they fail for the right reason.
4. **Implement in small steps.** After each step run the relevant tests. Keep diffs reviewable.
5. **Verify for real.** Restart services, run the app, call the endpoints, run E2E tests, and for UI run `ui-verifier`. Capture evidence.
6. **Review in fresh context.** Run `code-reviewer`, plus `security-reviewer`, `infra-reviewer` or `migration-reviewer` when the milestone touches their area. Fix findings and re-run tests.
7. **Record and report.** Update PROGRESS.md and DECISIONS.md. Tell the user: what was built, evidence (test counts, URLs, screenshots), what is not verified yet, what's next. **Milestone gate: wait for the user's go-ahead.**

Rules inside the loop:
- If the same problem fails twice, stop, explain, and re-plan in plan mode.
- Do not expand scope mid-milestone; add new ideas to PLAN.md under "later".
- For independent milestones (no shared files), the user can run them in parallel using git worktrees, one session per worktree.
- Fix root causes, not symptoms. If a bug class could recur elsewhere, fix it where all callers benefit and add a test.

## Phase 6: Deploy and operate

Order: infrastructure code → staging → smoke tests → production.

- **IaC:** Terraform (or the cloud's native IaC). Run `terraform plan`, have `infra-reviewer` review the output, then the human runs `apply`.
- **Secrets:** cloud secret manager; nothing secret in the repo, images or logs.
- **SSL/TLS:** AWS Certificate Manager on the load balancer or CloudFront (DNS validation; the human adds records if they own DNS), or Let's Encrypt with auto-renewal elsewhere. Redirect HTTP to HTTPS. Verify with a real HTTPS request.
- **CI/CD:** GitHub Actions (or equivalent) runs lint, tests and build on every PR; deploys to staging on merge; production deploy needs a manual approval. Use OIDC to the cloud, not stored access keys. Optionally add an automated Claude code review on PRs.
- **Observability:** health endpoint, structured logs, error tracking, alarms on errors and latency, uptime check.
- **Data:** automated backups and one tested restore before launch.
- **Rollback:** a written, tested way back for every deploy.

**Desktop/background agents:** packaging per OS; code-signing certificate (the human purchases it); signed auto-update manifests with the private key outside the repo; test install, update and uninstall on a clean VM; check antivirus false positives; least-privilege permissions; secure pairing/auth to the backend; local logs with secrets redacted.

## Phase 7: Handover

Write `README.md` (setup and run), `docs/RUNBOOK.md` (deploy, rollback, rotate secrets, restore backup, common incidents), keep ARCHITECTURE.md current, and an honest **known limitations** list. Final report: what is live (URLs), what's verified and how, what remains, the costs that will recur.

## One-way doors (always stop and get explicit approval)

Production deploys; `terraform apply`/destroy; database migrations on shared or production data; deleting data or files; DNS changes; switching payments to live mode; sending real emails, SMS or notifications; publishing packages or app releases; purchases or anything that incurs cost; rotating credentials; force pushes; anything that touches real users' accounts. Prepare everything, show exactly what will happen, and wait.

## Model routing

| Work | Model |
|---|---|
| Spec, architecture, plan, plan review, security/infra/migration review, hard debugging | Opus |
| Implementation, tests, code review, UI verification, docs | Sonnet |
| Bulk mechanical edits (renames, translations, boilerplate) | Haiku or Sonnet |

Tell the user when switching the main session's model would help (`/model`). Subagents set their own model in frontmatter.

## Reporting rules

- Claims come with evidence: actual test output counts, commands run, URLs, screenshot paths.
- Say clearly what has not been verified and why.
- Keep status updates short; the files (PROGRESS.md, PLAN.md) hold the detail.

## Worked example

Request: "Build me a full-stack e-commerce app with a desktop agent, SSL, deployed on AWS, complete."

1. **Size:** XL. Human must provide: AWS account, domain/DNS, Stripe account, code-signing certificate, what the desktop agent should do.
2. **Questions (one batch):** target market and currency; expected catalog size and traffic; stack preference; what the desktop agent does (e.g. store-owner order notifications and label printing) and which OS; deadline; monthly cloud budget.
3. **Defaults if unanswered:** Next.js frontend, FastAPI or Node backend, Postgres (RDS), S3 + CloudFront for images, Stripe Checkout in test mode, ECS Fargate behind an ALB with ACM certificate, Terraform, GitHub Actions with OIDC, Electron or Tauri agent.
4. **Milestones:** M0 skeleton deployed to staging over HTTPS with CI → M1 catalog and search → M2 cart and Stripe Checkout (test mode, webhook verified) → M3 accounts and order history → M4 admin (products, orders, refunds) → M5 desktop agent (pairing, live order feed, signed auto-update) → M6 production infra, backups, alarms, load test → M7 security review and launch checklist.
5. **Subagents created (5), with reasons:** plan-reviewer (always), code-reviewer (always), security-reviewer (payments, auth, PII, desktop agent), infra-reviewer (AWS, Terraform, SSL, CI/CD), ui-verifier (storefront and admin UI). No migration-reviewer at first (no production data yet); add it at M6.
6. **Gates:** plan approval; harness approval; end of every milestone; every `terraform apply`; Stripe live mode; production launch; publishing the signed desktop installer.

## Anti-patterns to avoid

- Writing all the code in one pass, then testing at the end.
- Huge CLAUDE.md files that restate obvious practices.
- Creating many agents "just in case".
- Letting the author of a plan or diff be its only reviewer.
- Reporting "done" from reading code rather than running it.
- Continuing a polluted, very long session instead of handing off through PROGRESS.md.
- Doing irreversible steps because the plan was approved earlier; each one-way door needs its own approval.