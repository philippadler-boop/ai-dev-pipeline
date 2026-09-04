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
| `architect.md` | Read, Grep, Glob, Write | Writes architecture doc + ADRs (structured template, per decision 6); no code edits |
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

## M3 — Spec Kit: Constitution + Specify

**Goal:** turn the raw idea note into a real concept + requirements spec,
using GitHub Spec Kit per decision 4 — this is the Requirements Gate from
`assessment.md` Section 6.

- Install Spec Kit in WhisperFlow.
- Run the Constitution phase: a short set of project principles (e.g.,
  "prefer explicit CLI flags over interactive prompts," "no network calls
  unless a cloud provider is explicitly selected") — this matters more than
  it looks like it should, since it's what later agents will be measured
  against.
- Run the Specify phase against `docs/ideas/initial-idea.md`, working
  through the open questions already listed there (interface, local vs.
  cloud transcription/translation, supported formats, target languages,
  privacy/cost constraints).
- Output: a numbered, testable requirements list (`FR-001`, `FR-002`, ...).

**Evidence:** a committed requirements spec with no open
`[NEEDS CLARIFICATION]` markers; you've reviewed and approved it (Requirements
Gate is human-approved per Section 6).

## M4 — Architecture + task breakdown

**Goal:** the Design Gate — a real architecture doc, ADRs for the
non-trivial calls, and a task list that's actually implementable one item
at a time.

- Invoke `architect.md` against the approved requirements spec.
- Output: `docs/architecture.md` plus one ADR per significant decision
  (transcription engine choice, translation approach, subtitle format,
  CLI framework) using the structured template (context / decision /
  alternatives considered / consequences).
- Convert the task list into GitHub Issues, one per task, each referencing
  its `FR-xxx` ID in the issue body — this reference is what M5's
  traceability check will enforce.

**Evidence:** architecture doc + ADRs committed; every requirement maps to
at least one issue; you've approved the architecture (Design Gate, human-approved).

## M5 — CI: build, test, lint, security, traceability

**Goal:** the objective, non-agent-controlled evidence layer that makes the
Implementation and Test gates real, per Section 6 and Section 11.

- One GitHub Actions workflow, triggered on PR: build, run tests, lint.
- Enable Dependabot (alerts + security updates) and GitHub code scanning —
  both free, both configured once.
- Add a traceability check: a small script that fails the workflow if the
  PR body/title doesn't reference a `FR-` or issue number. This is
  required per decision 5, not optional, because unattended agents (M8)
  won't self-police that link the way a human would.
- Update `main`'s branch protection to require these checks before merge
  (completing what M1 left open).
- Pin any third-party Actions used in the workflow to a commit SHA, not a
  floating tag (Section 12).

**Evidence:** a throwaway PR (e.g., a one-line README fix, deliberately
*not* referencing a REQ ID) is opened and observed to be blocked by the
traceability check — proving the gate actually gates before any real code
depends on it. A second PR that does reference a REQ ID passes.

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
