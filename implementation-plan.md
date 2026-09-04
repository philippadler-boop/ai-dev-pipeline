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

## M5 — CI: build, test, lint, security, traceability

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
- [x] Traceability check: `.github/workflows/traceability.yml` fails the
  PR if `github.event.pull_request.title`/`.body` don't match
  `FR-[0-9]{3,}` or `#[0-9]+`, per decision 5.
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

**Evidence:** shipped via PR `philippadler-boop/WhisperFlow#<PR-number>`
(branch `chore/m5-ci-cd-setup`, tracking issue #30), all five checks
green, merged to `main`. Branch protection confirmed live via the `gh
api` response (`checks[].app_id: 15368` on all five contexts, matching
the GitHub Actions app). **Still open:** a throwaway PR (e.g., a one-line
README fix, deliberately *not* referencing an `FR-` ID or issue) opened
and observed to be blocked by the traceability check, then closed
without merging — proves the gate actually gates before any real code
depends on it, not just that it passed once on a PR that happened to
reference one.

## M6 — Security baseline

**Goal:** the stricter posture decision 7 calls for, done now rather than
deferred, since decision 3 accepted broader unattended autonomy.

- Configure Claude Code's sandbox credential deny-list (`~/.ssh`,
  `~/.aws/credentials`, and equivalents) for any session working in this repo.
- Scrub sensitive environment variables from any context the `developer`
  or unattended agents' Bash tool can see.
- Create a repo-scoped GitHub token (or GitHub App installation limited to
  WhisperFlow) for any automation — never an org-wide or account-wide token.
- Decide the container boundary for unattended work (M8 uses GitHub
  Actions' own sandboxing by default, which already satisfies this; a
  devcontainer is only needed if you also want unattended work runnable
  locally).

**Evidence:** a documented (one paragraph in this repo's `CLAUDE.md` or a
short `SECURITY-NOTES.md`) statement of what's denied and what token scope
is in use — something you can point to later rather than trusting memory.

## M7 — First end-to-end task (supervised)

**Goal:** prove the whole loop — Developer → CI → Reviewer → QA → human
merge — works on one deliberately small, low-risk task before trusting it
with anything real. Pick the smallest task from M4's breakdown (e.g., "CLI
skeleton that accepts a video file path and prints its detected duration"
— exercises the toolchain without touching transcription/translation yet).

- `developer.md` implements on a feature branch, opens a PR referencing its
  REQ/issue ID.
- CI runs (M5); must be green.
- `reviewer.md` reviews the diff in an isolated context (it has not seen
  the developer's reasoning, only the diff and the spec) and produces a
  review report.
- `qa.md` maps the requirement to evidence (does the CLI actually print a
  duration for a real sample file?) and produces a validation report.
- You merge.

**Evidence:** one merged PR with a full trail — CI run, review report,
validation report, all linked from the PR — that you can point to as "this
is what the pipeline actually produces," not a description of what it's
supposed to produce.

## M8 — Turn on unattended work (per decision 3)

**Goal:** now that the gates are proven under supervision (M7), extend
autonomy to unattended execution for a narrow, low-risk task category —
this is the point where decision 3's "broader autonomy is fine" actually
gets exercised, deliberately sequenced *after* M7 rather than from day one,
so the first thing running unattended is a pipeline you've already watched
work correctly once.

- Enable GitHub Copilot coding agent (or configure a Claude Code GitHub
  Action) on the WhisperFlow repo.
- Assign it one small, well-scoped issue (still something low-risk — not
  the first real feature).
- Confirm it produces a PR that still has to pass M5's gates and M7's
  review/validation loop like everything else — unattended execution changes
  *who* starts the work, not which gates it has to clear.

**Evidence:** an unattended-agent-authored PR that passed the same gates as
M7's, merged the same way.

## M9 — Retro and adjust

**Goal:** close the loop on this implementation plan itself.

- Note any friction from M2–M8 in `decisions.md` (a subagent's tool
  allowlist was too tight/loose, the traceability check had false
  positives, review reports needed a stricter template, etc.).
- Update the subagent definitions / CI config in WhisperFlow accordingly.
- Only then start treating the pipeline as "working" for real WhisperFlow
  features beyond the first pilot task.

**Evidence:** a `decisions.md` entry summarizing what was learned and what
changed as a result.

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
