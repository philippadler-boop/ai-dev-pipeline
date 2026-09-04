# V1 Implementation Plan

Status: active — pilot project is **WhisperFlow** (`../WhisperFlow`)

This turns `assessment.md` Section 14 (Recommended V1) and the decisions in
`decisions.md` into an ordered, concrete sequence of milestones. Each
milestone has a stated deliverable and the evidence that closes it — in
keeping with the assessment's core principle, a milestone isn't "done"
because it was attempted, it's done because the stated evidence exists.

Milestones are ordered so the pipeline's own machinery (repo, CI, gates,
security) is proven out on trivial content *before* any real WhisperFlow
requirement depends on it — the same "don't trust a claim without evidence"
principle applied to building the pipeline itself.

---

## M0 — Preconditions (done)

- [x] Research & architecture assessment complete (`assessment.md`)
- [x] Open questions resolved (`decisions.md`)
- [x] Pilot project chosen: WhisperFlow, idea captured

## M1 — WhisperFlow on GitHub (done)

**Goal:** move WhisperFlow from a local-only repo to GitHub, since almost
everything downstream (Issues, PRs, Actions, branch protection) needs it there.

- [x] Create a GitHub repo for WhisperFlow (personal account, private is fine
  to start — nothing here needs to be public).
- [x] Push the existing local history (`README.md`, `docs/ideas/initial-idea.md`).
- [x] Add a minimal branch protection rule on `main`: require a pull request
  before merging. (Required status checks get added in M5, once there are
  checks to require — an empty required-check list blocks every PR.)

**Evidence:** private repo at https://github.com/philippadler-boop/WhisperFlow
(GitHub Pro account), history pushed to `main` (renamed from the initial
`master` default for consistency with this plan; old branch deleted).
Verified via `gh api repos/philippadler-boop/WhisperFlow/branches/main/protection`:
`required_pull_request_reviews` present (0 required approvals — direct
pushes blocked, no approval count required yet), `required_status_checks`
off pending M5, force-push and branch deletion both disabled. `enforce_admins`
is `false` by design — the gate binds agents, not the repo owner's own
override ability (Section 7).

## M2 — Subagent definitions (done)

**Goal:** the five roles from `assessment.md` Section 3 exist as real,
usable Claude Code subagents in WhisperFlow's own repo (`.claude/agents/`),
not just a description in a document.

| File | Tools (allowlist) | Notes |
|---|---|---|
| `analyst.md` | Read, Grep, Glob, Write, WebSearch | Writes concept/requirements docs; no code access |
| `architect.md` | Read, Grep, Glob, Write | Writes ADRs (structured template, per decision 6) from Spec Kit's Plan-phase output (`plan.md`/`research.md`/`data-model.md`); no `docs/architecture.md`, no task breakdown (that's `/speckit.tasks`), no code edits, no shell — corrected 2026-09-04, see decisions.md |
| `developer.md` | Read, Edit, Write, Bash, Grep, Glob | Implements on a feature branch; Bash needed for running builds/tests/git |
| `reviewer.md` | Read, Grep, Glob | **No Write/Edit** — this is deliberate (Section 3): a reviewer that can fix what it's reviewing isn't an independent review |
| `qa.md` | Read, Bash, Grep, Glob, Write (validation report only) | Runs tests/build to gather evidence; writes the requirement → evidence validation report |

- [x] All five files committed under `WhisperFlow/.claude/agents/`, tool
  allowlists matching the table above exactly.
- [x] Repo-root `CLAUDE.md` added to WhisperFlow stating where specs/ADRs/
  reports live, the branch-per-task convention, and that no subagent merges
  its own work.
- [x] Smoke test: each subagent invoked once via `claude --agent <name> -p
  "..."` from a session rooted at `WhisperFlow` (fixed by opening WhisperFlow
  itself as the VS Code workspace root, per the M2 audit finding). The
  load-bearing case passed cleanly: `reviewer` refused an edit request,
  correctly citing that it has no `Write`/`Edit` and that fixing issues is
  outside its role. `architect` self-reported its exact tool list correctly
  and additionally declined to produce real output ahead of an approved
  requirements spec. `analyst`/`developer`/`qa` stayed within their allowed
  tools in practice but skipped the "list your tools" part of the prompt, so
  their self-report is missing — a real evidentiary gap, not re-tested
  since nothing they did indicated overreach.

**Evidence:** all five files + `CLAUDE.md` committed (`WhisperFlow` commit
`0c346ad`); smoke test transcripts recorded in `decisions.md` (2026-09-03,
"M2 smoke test run" and "MCP-leak investigation resolved"). Every
subagent's actual callable tool schema was confirmed to match its declared
`tools:` allowlist exactly, via literal invocation attempts (not just
self-report) for reviewer, developer, and qa, and via exact self-report
plus consistent behavior for analyst and architect. An initial false
alarm (developer/qa appearing to have an extra MCP tool, `codegraph_explore`,
from a separately machine-installed CodeGraph server) was run down and
resolved: it was a self-report artifact (conflating injected MCP server
instructions with an actual tool grant), not a real allowlist gap — see
decisions.md for the full investigation.

## M3 — Spec Kit: Constitution + Specify (done)

**Goal:** turn the raw idea note into a real concept + requirements spec,
using GitHub Spec Kit per decision 4 — this is the Requirements Gate from
`assessment.md` Section 6.

- [x] Install Spec Kit in WhisperFlow (`specify init --here --force
  --non-interactive --integration claude`).
- [x] Run the Constitution phase: five principles derived from decisions
  already made and the existing `.claude/agents/*`/`CLAUDE.md`, not
  invented fresh (`.specify/memory/constitution.md`, v1.0.1). Constitution
  is project-wide, not feature-specific, so it's the one Spec Kit phase
  that correctly stays on `main` directly.
- **MUST, before running Specify** (corrected 2026-09-04 — skipped for
  this feature, see decisions.md): create and check out a real feature
  branch matching the feature directory name Spec Kit will use (e.g.
  `git checkout -b 001-<slug>`, matching `001-video-subtitle-generator`'s
  own naming). Every phase from Specify through the end of M4 — Specify,
  Clarify, Plan, Tasks, Analyze, ADRs — commits to this branch, never
  `main` directly. `.specify/extensions.yml` hooks are not configured, so
  nothing creates this branch automatically; it must be done by hand,
  every time, for every future feature. Skipping this is exactly what
  produced the `/speckit.analyze` CRITICAL finding (D1) on this feature —
  `tasks.md` described task branches relative to a feature branch that
  was never actually created.
- [x] Run the Specify phase against `docs/ideas/initial-idea.md`, resolving
  all seven listed open questions — CLI-only, local/on-device transcription
  only, `.srt` output, **transcription-only for v1** (translation deferred
  as a fast-follow — reaffirmed after an explicit scope check-in, see
  decisions.md).
- Output: a numbered, testable requirements list (`FR-001`...`FR-010` —
  corrected from this plan's original `REQ-` prefix to match Spec Kit's
  own template convention; see decisions.md).

**Evidence:** `WhisperFlow/specs/001-video-subtitle-generator/spec.md` +
`checklists/requirements.md`, committed and pushed to `main`
(commits `2a0bc65`, `593a38c`), zero `[NEEDS CLARIFICATION]` markers
(verified with `grep`, not the checklist's checkboxes alone). Reviewed and
approved as transcription-only v1 — Requirements Gate closed
(Section 6, human-approved per Principle II).

## M4 — Architecture + task breakdown (done)

**Goal:** the Design Gate — via Spec Kit's own Plan/Tasks chain per
decision 4 (use Spec Kit as-is), not a hand-rolled equivalent. Corrected
2026-09-04: the version of this milestone below (hand-authoring
`docs/architecture.md` and manually converting tasks to issues) was found
to conflict with decision 4 and with `assessment.md` Section 14's own
"Recommended V1" text once Spec Kit's installed skill set was actually
inspected — it already ships `/speckit.plan`, `/speckit.tasks`, and
`/speckit.taskstoissues`, covering exactly what this milestone was about
to reinvent. See decisions.md.

- [x] Run `/speckit.clarify` against the approved spec first (Spec Kit's
  own recommended order). Four resolutions: 2-hour max length firmed up,
  no-resume-on-interrupt, new FR-011 (progress indicator), new SC-006
  (timing target).
- [x] Run `/speckit.plan`: produced `plan.md`, `research.md`,
  `data-model.md`, `contracts/cli.md`, `quickstart.md` under
  `specs/001-video-subtitle-generator/`.
- [x] Run `/speckit.tasks`: produced `tasks.md` — 29 dependency-ordered
  tasks (`T001`–`T029`, expanded from an initial 27 by two Analyze-driven
  additions, T028/T029).
- [x] Run `/speckit.analyze`: found 1 CRITICAL (D1: task branches
  described relative to a feature branch that didn't exist) + 5
  lower-severity findings, all independently spot-checked against the
  actual files (not taken on the report's word) and fixed. Re-verified
  clean.
- [x] Invoke `architect.md` to write one ADR per significant decision
  `plan.md`/`research.md` surfaced: `0001-local-asr-engine.md`,
  `0002-audio-extraction.md`, `0003-subtitle-composition.md`,
  `0004-cli-framework.md`, all under `docs/adr/`. First non-interactive
  (`-p`) invocation silently wrote nothing — Claude Code's session
  permission gate can't prompt for approval in that mode; re-run
  interactively, with the write approved live, produced all four. Each
  independently reviewed against `research.md`/`plan.md`/`spec.md` —
  correct structure, correct FR-/SC- traceability.
- [x] Run `/speckit.taskstoissues` to convert `tasks.md` into GitHub
  Issues: 29 of 29 converted, `#1`–`#29`, none skipped, titles verified
  (via `gh issue list`) to match `tasks.md` character-for-character on
  spot-checked entries (T001, T025, T029).

- **MUST, once `/speckit.analyze` is clean and you approve the plan**
  (Design Gate): open a PR from the feature branch to `main` carrying
  every planning artifact produced since the branch was created —
  `spec.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`,
  `plan.md`, `tasks.md`, and the ADRs. Merging this PR *is* the Design
  Gate approval action (Principle II), not a separate verbal sign-off it
  merely follows. Only after this merge do `/speckit.taskstoissues` and
  M7's per-task branches begin — task branches (Principle IV) are cut
  from `main` for real at that point, because `main` actually contains
  the approved plan/tasks by then, not just a description claiming it
  will. **Did not apply to this feature**: per Principle IV's documented,
  one-time exception, `001-video-subtitle-generator` never had a feature
  branch, so there was no PR for the merge-is-approval mechanic to attach
  to — Design Gate approval here was your direct sign-off on the ADRs
  instead. Every feature after this one follows the MUST as written.

**Evidence:** `plan.md`/`research.md`/`data-model.md`/`tasks.md` + ADRs
committed to `main` directly (`8ae8944`, `6bb44db` — grandfathered
exception, no feature branch/PR for this feature, see above);
`/speckit.analyze` run with no unresolved CRITICAL findings; all 29 tasks
mapped to GitHub Issues `#1`–`#29` (verified via `gh issue list`, not the
completion report alone); you approved the ADRs (Design Gate,
human-approved). M4 closed 2026-09-04.

## M5 — CI: build, test, lint, security, traceability (done)

**Goal:** the objective, non-agent-controlled evidence layer that makes the
Implementation and Test gates real, per Section 6 and Section 11.

- [x] One GitHub Actions workflow (`.github/workflows/ci.yml`), triggered
  on PR and push to `main`: three jobs, `build`/`lint`/`test`. Each job
  detects whether there's anything to build/test yet (`pyproject.toml`,
  `tests/`) and no-ops cleanly before T001/T002 land, rather than failing
  on a repo that doesn't have code yet.
- [x] Enable Dependabot: `.github/dependabot.yml` (`pip` + `github-actions`
  ecosystems, weekly) plus the two account-level toggles (vulnerability
  alerts, automated security fixes) via `gh api -X PUT
  repos/.../vulnerability-alerts` and `.../automated-security-fixes`.
- [x] GitHub code scanning: `.github/workflows/codeql.yml`, Python,
  PR/push + weekly cron. Same "nothing to scan yet" problem as `ci.yml`,
  but a different failure mode — CodeQL's `finalize` step exits fatally
  (code 32) on zero source files instead of no-op'ing, so it needed an
  explicit `git ls-files '*.py'` check gating `init`/`analyze` behind a
  step output, not just a shell conditional inside one step.
- [x] **Correction (found during M7/T001, not before):** SARIF upload
  (the part of `codeql-action/analyze` that pushes findings to the
  Security tab) needs the "GitHub Code Security" product on a private
  repo, and that product has **no purchase path for an individual GitHub
  Pro account** -- org-only, Team/Enterprise plans. `assessment.md`
  Section 12/13 originally said code scanning was "cheap/included for
  private repos on most plans" -- that was wrong for this account type
  and has been corrected there. Decision: stay private, keep CodeQL
  *analysis* running (findings visible in the job log) but set
  `upload: false` on the `analyze` step and drop `security-events: write`
  from the job's permissions (unused once upload is off); Dependabot is
  unaffected and remains the primary automated dependency/security
  backbone. See decisions.md for the full reasoning. `actions: read` was
  also added to the job's permissions -- a separate, unrelated fix for a
  "Resource not accessible by integration" error on the workflow-runs API
  that CodeQL's own telemetry calls, which only surfaced once PR #38
  became the first PR with tracked `.py` files for CodeQL to actually run
  against.
- [x] Traceability check: `.github/workflows/traceability.yml` fails the
  PR if its title doesn't match `FR-[0-9]{3,}` or `#[0-9]+`, per decision
  5. Corrected twice after shipping, both times from real evidence rather
  than review: (1) checking the PR body too let Dependabot's
  auto-generated changelog (full of the *upstream* repo's own `#NNNN` PR
  references) produce a false pass, unrelated to anything in this repo —
  narrowed to title-only, since that's short and hand-written everywhere
  else in this project; (2) Dependabot's own PRs have no way to add a
  reference at all (titles are template-generated, e.g. "Bump X from Y to
  Z"), so the job is skipped entirely for `dependabot[bot]` — a skipped
  required check counts as passing, so this doesn't loosen the gate for
  anyone who actually can add one.
- [x] `main`'s branch protection updated (`gh api PUT
  branches/main/protection`) to require all five checks
  (`build`/`lint`/`test`/`codeql`/`traceability`), `strict: true`
  (branch must be up to date before merge) — preserves M1's existing
  0-required-approvals / no-force-push / no-branch-deletion settings.
- [x] Third-party Actions pinned to commit SHA, not a floating tag
  (Section 12): `actions/checkout@v7.0.1`, `actions/setup-python@v7.0.0`,
  `github/codeql-action@v4.37.0` — SHAs confirmed via `git ls-remote`
  against the real upstream repos, not assumed from training data (which
  is stale for "latest" as of this project's actual date). Kept current
  going forward by Dependabot's `github-actions` ecosystem entry above.

**Evidence:** shipped via PR `philippadler-boop/WhisperFlow#31`
(branch `chore/m5-ci-cd-setup`, tracking issue #30), all five checks
green, merged to `main`. Branch protection confirmed live via the `gh
api` response (`checks[].app_id: 15368` on all five contexts, matching
the GitHub Actions app).

Negative-test evidence: PR #34, deliberately titled without an `FR-`/`#N`
reference, was observed blocked by `traceability` while `build`/`lint`/
`test`/`codeql` all passed clean — confirmed via `gh pr checks 34`, not
assumed — then closed without merging.

Two real bugs surfaced live by Dependabot's own PRs (#32, #33 — routine
`codeql-action` SHA bumps) before the traceability check's own logic was
trustworthy, both fixed and shipped via PR #35: the false-pass-on-body
bug and the Dependabot-titles-never-match bug (see the check description
above). Also surfaced and worked around during this: branch protection's
`strict: true` blocked `#33` from merging after `#32` landed first
(head branch no longer up to date with `main` — expected behavior, fixed
via `gh pr update-branch`), and GitHub's OAuth `workflow`-scope
requirement blocked the first attempt to merge any PR touching
`.github/workflows/*` (fixed via `gh auth refresh -h github.com -s
workflow`).

## M6 — Security baseline (done)

**Goal:** the stricter posture decision 7 calls for, done now rather than
deferred, since decision 3 accepted broader unattended autonomy.

- [x] Configure Claude Code's credential deny-list (`~/.ssh`,
  `~/.aws/credentials`, and equivalents) for any session working in this
  repo: `.claude/settings.json`'s `permissions.deny` (Read, which per
  Claude Code >= v2.1.228 also blocks Edit/Write on the same path),
  applies on every platform regardless of session type. Verified against
  Claude Code's own current docs (permissions/sandboxing/security pages)
  via `claude-code-guide`, not assumed from training data — including the
  real caveat that a `Read` deny doesn't stop an arbitrary Bash
  subprocess from opening a file itself; only OS-level sandboxing closes
  that gap.
- [x] Decided, deliberately, **not** to enable Claude Code's stricter
  OS-level Bash sandbox (`sandbox.enabled`) yet — documented in
  `SECURITY-NOTES.md` rather than silently skipped: it's macOS/Linux/WSL2
  only (no native Windows), restricts Bash writes to
  working-dir/temp/added-dirs (real friction risk on everyday interactive
  work), and per assessment.md's own "ladder, not a switch" framing,
  isn't needed until unattended work might run outside GitHub Actions'
  own VM isolation. Revisit then.
- [x] GitHub token scope for future unattended automation (M8): documented
  policy in `SECURITY-NOTES.md` — a fine-grained PAT scoped to WhisperFlow
  only, Contents + Pull requests + Issues (read/write), nothing org-wide.
  Not yet created, since M8 hasn't started and there's nothing to scope it
  for yet — this is the decided policy, applied when M8 actually needs a
  token, per decision 7's "set up as part of V1" without inventing an
  unused credential today.
- [x] Container boundary for unattended work: GitHub Actions' own per-run
  VM isolation accepted as sufficient for M8 (documented in
  `SECURITY-NOTES.md`); no devcontainer unless unattended work is ever run
  locally instead.
- Environment-variable scrubbing: no `.env`/secret lives in this repo or
  is needed for WhisperFlow itself (FR-004, no network calls), so there's
  nothing repo-specific to scrub today. `SECURITY-NOTES.md` documents this
  plainly and flags — rather than glossing over — that the exact mechanics
  of Claude Code's env-var masking/`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`
  weren't fully confirmed from current docs, so it isn't presented as a
  settled guarantee.

**Evidence:** `SECURITY-NOTES.md` (WhisperFlow, merged via PR #37,
tracking issue #36) — what's denied, why the stricter sandbox isn't
enabled yet, the token-scope policy, and the container-boundary decision,
all in one place to point to later rather than trusting memory.

## M7 — First end-to-end task (supervised) (done)

**Goal:** prove the whole loop — Developer → CI → Reviewer → QA → human
merge — works on one deliberately small, low-risk task before trusting it
with anything real. Picked T001 from `tasks.md` (project structure
scaffolding: `src/`/`tests/` subdirectories + `__init__.py` files,
`scripts/`) — smaller than the CLI-skeleton example originally sketched
here, but the right size for a first run: pure filesystem structure, zero
logic, so any pipeline failure would be about the *process*, not the code.

- [x] `developer` implemented on branch `001/T001-project-structure`,
  opened PR #38 ("T001: Create project structure (#1)"), referencing
  issue #1.
- [x] CI ran and found two real bugs live, not in review: `CI/test`
  failed with pytest exit code 5 ("no tests collected") once T001 created
  an empty `tests/` tree, and `CodeQL/codeql` failed with a permissions
  error (`Resource not accessible by integration`) on its first real run
  against tracked `.py` files — every earlier PR had zero `.py` files, so
  `init`/`analyze` had always been skipped before this. Both root-caused
  from actual `gh run view --log-failed` output. Fixed across two more
  PRs (#39 for the two CI bugs, #40 once the CodeQL fix surfaced the
  deeper GHAS/private-repo gap — see the 2026-09-04 decisions.md entry).
- [x] `reviewer` reviewed the (by then merged) diff in an isolated
  session — fresh `claude --agent reviewer` invocation, no access to the
  developer's conversation, working only from the diff/on-disk state, the
  task spec, and `plan.md`'s target tree. **Approved**, structure matched
  `plan.md` exactly, all 9 required `__init__.py` files present.
- [x] `qa` independently re-verified via `git ls-files`/`git show` against
  the actual merge commit (not trusting CI or the review) and wrote
  `docs/validation/T001.md`. **PASS** on all 5 sub-requirements checked.
- [x] Merged (commit `0fb26b9` on `main`).

**Process gap found and worth naming honestly:** PR #38 was merged as
soon as CI went green, *before* `reviewer`/`qa` ever ran — the human-merge
step in this loop didn't actually wait on the review/QA gate the way this
milestone was designed to prove out. Caught when I cross-checked the merge
commit against the plan and asked; `reviewer`/`qa` were then run
retroactively against the already-merged commit, which is a legitimate
post-hoc content check but not the real gate this milestone exists to
prove — a real gate has to run *before* merge is possible, not after.
Both agents' output was fully independent and correct regardless (see
evidence below), so the *agents* are proven to work; what's proven weaker
is that a human reliably waits for them. Decision: for T002 onward,
merging before both `reviewer` approval and a passing `qa` validation
report exist is an explicit rule, not an assumed one — worth eventually
also enforcing as a required PR review/status check the way CI already is,
rather than relying on memory each time.

**Evidence:** PR #38 (merged, all 5 CI checks green per `gh pr checks 38`:
`CI/build`, `CI/lint`, `CI/test`, `CodeQL/codeql`, `Traceability/traceability`),
reviewer's approval report (full text, not summarized, reviewed 2026-09-04),
`docs/validation/T001.md` (qa's validation report, PASS on all 5
sub-requirements, committed in `b4107b4`), and this process-gap note. Full
narrative in decisions.md, 2026-09-04 entries.

## M8 — Turn on unattended work (per decision 3) (done)

**Goal:** now that the gates are proven under supervision (M7), extend
autonomy to unattended execution for a narrow, low-risk task category —
this is the point where decision 3's "broader autonomy is fine" actually
gets exercised, deliberately sequenced *after* M7 rather than from day one,
so the first thing running unattended is a pipeline you've already watched
work correctly once.

- [x] Chose the Claude Code GitHub Action over GitHub Copilot coding agent
  (issue #43) — reuses the exact `developer`/`reviewer`/`qa` subagent
  definitions already proven in M7 rather than standing up a second,
  differently-configured agent product. Authenticated via
  `CLAUDE_CODE_OAUTH_TOKEN` (subscription allowance, one billing surface)
  installed through the guided `/install-github-app` flow. Deliberately
  used the shared Claude GitHub App rather than the fine-grained,
  WhisperFlow-only PAT M6 originally specified — a conscious deviation,
  reasoned through and logged (2026-09-04 decisions.md entries), with
  `SECURITY-NOTES.md` updated to describe what was actually implemented
  rather than left describing an unfollowed policy. `SECURITY-NOTES.md`
  update went through its own PR (#44) rather than a direct commit —
  learned the hard way earlier in M7 after I mistakenly committed straight
  to `main` once.
- [x] Built `.github/workflows/claude-dev-agent.yml` (PR #47) as a
  *separate* workflow from the `/install-github-app`-generated `claude.yml`
  — `claude.yml` stays the interactive `@claude`-mention workflow;
  `claude-dev-agent.yml` triggers in automation mode (a `prompt` input) on
  an issue getting the `claude-dev` label, invoking `--agent developer` via
  `claude_args`. No `github_token` override (required for the Action's
  commits to actually trigger downstream CI — a default `GITHUB_TOKEN`
  silently wouldn't) and deliberately no auto-merge step: the PR it
  produces has to clear the same gates as every other PR (CI, `reviewer`,
  `qa`, human merge).
- [x] First smoke-test run (throwaway issue #48) failed silently —
  completed with exit success and zero visible errors, but produced no
  branch/commit/PR, `permission_denials_count: 20` in the result JSON.
  Root cause, found from actual evidence not guessing: automation mode
  grants Claude no shell/GitHub tool access at all until explicitly
  allowed. Fixed (PR #49) by adding `--allowedTools "Bash,Edit,Write,Read,
  Grep,Glob"` — deliberately mirroring `developer.md`'s own `tools:` line
  exactly, so the unattended run gets no more access than the subagent
  already has interactively. Second smoke test (throwaway issue #50)
  confirmed the fix and, with `show_full_output: true` turned on,
  independently confirmed `--agent developer` was genuinely loading
  `developer.md`'s config (the session's own "init" event reported
  exactly developer.md's five-tool list, not the default Claude Code
  toolset) — closed without merging (PR #51), as a smoke test should be.
- [x] Real pilot: T002 (`pyproject.toml`) and T003 (`tests/conftest.py`)
  from `tasks.md`, each picked up unattended via the `claude-dev` label.
  Both implementation commits are authored by `claude[bot]
  <209825114+claude[bot]@users.noreply.github.com>` — the actual GitHub
  App bot identity, independently confirmed via `git log --format`, not
  self-reported — real evidence the trigger fired rather than a human
  running the CLI by hand and calling it unattended.
- [x] Both went through the exact same gates as M7: CI green (`gh pr
  checks`, independently verified), `reviewer` caught a real issue on
  T002 (an unsourced `license` field in `pyproject.toml`, fixed before
  merge) and approved T003 clean, `qa` independently re-verified both —
  T002 by building a fresh disposable virtualenv and running a real `pip
  install -e ".[dev]"` plus `ruff`/`pytest`, T003 by probing the fixture
  videos directly with a local `ffmpeg`/`ffprobe` install — neither
  trusted CI or the reviewer's word alone. T003 also caught and fixed a
  real CI failure (`ffmpeg` missing from the `test` job's `PATH`) before
  merging. A human merged both (PRs #53/#54 for T002, #55/#56 for T003,
  all independently confirmed `MERGED` via `gh pr list --json`).

**Evidence:** two unattended-agent-authored PRs (T002, T003), both
independently confirmed `claude[bot]`-authored, both passing the identical
CI/reviewer/qa/human-merge gate sequence M7 proved out, plus a documented
smoke-test failure and fix showing the setup was actually debugged from
real evidence (`permission_denials_count`, actual init-event tool lists)
rather than assumed correct. `docs/validation/T002.md` and
`docs/validation/T003.md` in WhisperFlow carry the full qa evidence.

## M9 — Retro and adjust (done)

**Goal:** close the loop on this implementation plan itself.

- [x] Read back through every `decisions.md` entry from M2 through M8
  looking for recurring friction, not just each milestone's own
  close-out note.
- [x] Two real gaps found and fixed in WhisperFlow (branch
  `docs/m9-retro-adjustments`, commit `2db0850`, PR pending): `reviewer`
  had no documented way to receive a PR diff (it has no Bash/git/gh) —
  now explicit in `reviewer.md`/`CLAUDE.md` that the invoker must supply
  it; and two hard-won lessons (automation-mode `claude-code-action`
  grants zero tools by default; smoke-test a subagent's tool boundary by
  invocation, not self-report) are now written into a new "Operational
  lessons" section of `CLAUDE.md` instead of living only in this
  project's `decisions.md` history.
- [x] Checked, not re-fixed: M2's open recommendation about the VS Code
  workspace root leaking Bash permissions across sibling repos — confirmed
  resolved (likely as a side effect of the `C:\Users\phili\Projects`
  migration), not assumed fixed.
- [x] Pipeline is now to be treated as "working" for real WhisperFlow
  feature work beyond the T001–T003 pilot tasks, not something still
  being proven out.

**Evidence:** `decisions.md` entry "M9 closed" summarizing what was found
and what changed; WhisperFlow commit `2db0850` (branch
`docs/m9-retro-adjustments`) with the actual `reviewer.md`/`CLAUDE.md`
fixes.

---

## Explicitly out of scope for V1

Per `assessment.md` Section 15 (Evolution Path), these are intentionally
deferred, not forgotten: a dedicated orchestration framework or automated
gate-sequencing script beyond what GitHub's own triggers provide; Security
Reviewer and Documentation Engineer as separate subagents (start as
checklists inside `developer`/`architect` until a specific gap is felt);
Release Engineer and Maintenance/Monitoring agents (no release cadence or
issue backlog exists yet to justify them); any tracing/observability tool
beyond git history, PR threads, and Actions logs.
