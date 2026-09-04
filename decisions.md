# Decisions Log

A running, dated record of decisions made about the AI dev pipeline, beyond
what's captured in `assessment.md`. Entries are appended at the bottom in
the order they happened (in practice, newest at the bottom, not the top —
fixing this note to match actual behavior rather than the other way
around). When a decision changes something the assessment already
recommends, note it here rather than silently diverging from that document.

---

## 2026-09-03 — M1 complete: WhisperFlow on GitHub

WhisperFlow is at https://github.com/philippadler-boop/WhisperFlow — a
private repo (this account is now on GitHub Pro), with branch protection on
`master` requiring a pull request before merging. Commits use the GitHub
noreply email, matching global git config.

GitHub CLI (`gh`) is installed and authenticated as `philippadler-boop`,
needed for this and every GitHub-related step from M4 onward (Issues, PR
traceability checks, etc.).

---

## 2026-09-03 — Open questions from the assessment, resolved

1. **Pilot approach:** a real project, not the pipeline itself as the first project.
2. **Approval style:** Balanced — interactive approval at Requirements/Design
   sign-off and Merge/Release only; everything else proceeds with a summary,
   not a blocking prompt.
3. **Unattended agent tolerance:** broader autonomy is acceptable, including
   an agent picking up issues and opening PRs without real-time supervision,
   even for non-trivial tasks. This was the highest-consequence decision —
   it pulled two things forward into V1 that would otherwise have been
   deferred: unattended execution itself, and the stricter security/sandboxing
   baseline (see assessment.md, Section 12/14).
4. **Spec-driven workflow:** use GitHub Spec Kit as-is (Constitution → Specify
   → Plan → Tasks → Implement) rather than hand-rolling an equivalent.
5. **Traceability enforcement:** enforced by CI (a required check fails a PR
   that doesn't reference a requirement/issue ID), not left to convention —
   direct consequence of decision 3.
6. **ADR formality:** structured template (context, decision, alternatives
   considered, consequences) for every non-trivial architectural call.
7. **Security baseline:** set up as part of V1, not deferred — credential
   deny-lists, least-privilege GitHub tokens, container/CI sandboxing for
   any agent that can act without supervision.

**Still open:** where "shared project memory" (conventions, architecture
summary, reusable subagent templates) should live once there's a second
project — not decidable in the abstract, revisit when that project exists.

---

## 2026-09-03 — Pilot project chosen: WhisperFlow

A tool that takes a given video and generates subtitles in a chosen
language. Set up as its own repo at `../WhisperFlow` (local git, no GitHub
remote yet), containing only the raw idea note so far
(`../WhisperFlow/docs/ideas/initial-idea.md`).

Chosen because it's self-contained and small enough to run through the full
lifecycle without ballooning, while still forcing real architecture
decisions (local vs. cloud transcription, local vs. cloud translation,
subtitle format) that will actually exercise the Architect agent and ADR
process rather than being trivial.

Next step: open `../WhisperFlow` in VS Code / Claude Code and run GitHub
Spec Kit's `specify` phase to turn the idea note into a concept brief and
requirements spec (per decision 4 in the entry above).

---

## 2026-09-03 — M1 branch fixup: master → main, verified

WhisperFlow's default branch was renamed from `master` to `main` for
consistency with `assessment.md`/`implementation-plan.md` (both used `main`
throughout). Sequence: renamed and pushed `main`, removed the protection
rule blocking deletion of `master` (`gh api -X DELETE .../branches/master/protection`),
deleted `master`, set `main` as the GitHub default branch, manually
re-added the "require a pull request before merging" rule on `main`. The
planning repo's own local branch (no GitHub remote) was renamed the same way
for consistency, trivially since nothing depended on its name.

Verified via `gh api` (not just observed in the UI) that the new rule on
`main` matches M1's original intent: PR required, no approval count yet, no
status checks yet (that's M5), force-push/deletion blocked. `enforce_admins`
is `false`, meaning the repo owner can still bypass the rule directly —
accepted as intentional (Section 7: human override stays available) rather
than a gap to close.

---

## 2026-09-03 — M2 subagent files shipped; smoke test deferred to a WhisperFlow-rooted session

`analyst`, `architect`, `developer`, `reviewer`, and `qa` are committed at
`WhisperFlow/.claude/agents/*.md` (commit `0c346ad`, pushed directly to
`main` — same direct-push precedent as M1, since this is pipeline
bootstrapping rather than a REQ-tracked task). `WhisperFlow/CLAUDE.md` was
added alongside them. Each file's `tools:` allowlist matches
`implementation-plan.md`'s M2 table exactly; `reviewer` carries no
`Write`/`Edit`.

M2's smoke test (invoke each subagent once, confirm it can't exceed its
allowlist) can't run from this planning repo's session — Claude Code
discovers `.claude/agents/*.md` from the session's working-directory root,
and this session is rooted here, not at `WhisperFlow`. The project owner
will run the smoke test directly in a Claude Code session opened on
`WhisperFlow`; result to be logged here once done.

---

## 2026-09-03 — Independent audit of M2 evidence

Reviewed WhisperFlow's files directly rather than trusting the plan's
self-report. Confirmed: all five subagents committed at
`WhisperFlow/.claude/agents/*.md` (commit `0c346ad`), every `tools:`
allowlist matches `implementation-plan.md`'s M2 table exactly, `reviewer`
has no `Write`/`Edit`. The plan's own "(files done, smoke test pending)"
status is accurate, not overclaimed.

New finding: `C:\Users\phili\OneDrive\Projects\.claude\settings.local.json`
exists at the Projects root, not inside `WhisperFlow\.claude\`. This
indicates the Claude Code/VS Code workspace is currently opened at the
parent `Projects` folder rather than at `WhisperFlow` itself — the likely
actual reason the smoke test is blocked (subagent discovery is relative to
the workspace root), and a more precise diagnosis than "wrong session."
Side effect: the Bash permissions already approved there (git add/commit/
config/push, a couple of literal `mkdir` commands) are scoped to the whole
`Projects` folder, not just WhisperFlow, so they'll silently apply to
`ai-dev-pipeline` and any future sibling project — a minor deviation from
the repo-scoped/least-privilege principle in Section 12/M6.

Recommendation (not yet acted on): open `WhisperFlow` itself as the VS Code
workspace root, not `Projects`, to unblock the smoke test and stop future
permission grants from leaking across projects.

Also noted: `docs/adr/` and `docs/validation/` exist on disk (created ahead
of M4/M7) but are empty and untracked by git — harmless, won't appear on a
fresh clone.

---

## 2026-09-03 — M2 smoke test run: reviewer's restriction verified at runtime

Ran all five subagents non-interactively (`claude --agent <name> -p "..."`)
from a session correctly rooted at `WhisperFlow` (the earlier workspace-root
fix was applied). Results:

- `reviewer`: asked to directly edit a wording issue in a doc. Refused,
  correctly citing "I don't have Write or Edit tools — that's intentional
  for this role" and correctly scoping itself to reviewing diffs, not
  making changes. This is the one restriction that's allowlist-enforced
  (hard) rather than instruction-enforced (soft), and it held under actual
  runtime pressure, not just as a YAML declaration.
- `architect`: self-reported its tool list exactly right (Read, Grep, Glob,
  Write), and independently declined to produce real architecture output
  because no approved requirements spec exists yet — correctly deferring to
  the Requirements Gate rather than improvising ahead of it.
- `analyst`, `developer`, `qa`: each completed its trivial task correctly
  and stayed within its allowed tools (Read+Write for analyst, Bash for
  developer and qa) — but none of the three actually answered the "list
  your tools" half of the prompt, so there's no explicit self-report to
  check against their declared allowlists, only absence of observed
  overreach. Not re-run, since nothing indicated a real problem, but worth
  knowing this part of the evidence is thinner than reviewer's and
  architect's.

M2 marked done on that basis: the one restriction that actually matters
(reviewer can't edit) is verified at runtime, not just declared.

---

## 2026-09-03 — MCP-leak investigation resolved: tool allowlists confirmed hard-enforced

Follow-up to the earlier smoke test gap. Sequence:

1. Re-tested `analyst`/`developer`/`qa` in isolation (single-purpose "list
   your tools" prompts, no competing task). `analyst` matched its declared
   allowlist exactly. `developer` and `qa` also matched their declared
   tools — but both additionally claimed `codegraph_explore`, an MCP tool
   not present in either file's `tools:` line.
2. User confirmed CodeGraph is a separate, self-installed tool
   (`codegraph install` registers its MCP server into Claude Code, Cursor,
   etc. at the agent/machine level — unrelated to WhisperFlow's own
   per-subagent config).
3. Tested `reviewer` with the same question, since it's the one subagent
   where a real leak would actually matter (any extra tool beyond
   `Read`/`Grep`/`Glob` would undercut the one hard guarantee this design
   depends on). `reviewer` correctly distinguished between injected MCP
   *server instructions* (descriptive context telling it how it would use
   `codegraph_explore` if available) and an actual callable function —
   reported it has neither, and correctly declined to "invoke" a function
   with no schema entry rather than pretending to try.
4. Re-tested `developer`/`qa` asking them to actually attempt invocation
   rather than just listing tools. Both got the literal result
   `Error: No such tool available: codegraph_explore` — confirming, by
   actual runtime rejection rather than self-report, that neither has real
   access to it either.

**Conclusion:** this was a false alarm caused by self-report inaccuracy,
not an actual gap in tool-allowlist enforcement. CodeGraph's MCP server
injects descriptive usage instructions broadly (likely because it's
registered at the machine/agent level), and `developer`/`qa` initially
conflated "I have instructions describing this tool" with "I can call this
tool." `reviewer` did not make that mistake unprompted; `developer`/`qa`
did, but corrected immediately once asked to actually invoke it. In every
case tested, the real callable tool schema matched the declared `tools:`
allowlist exactly — confirmed at the strongest available evidence level
(a literal invocation attempt and its rejection), not just a self-report.

**Methodological lesson for future smoke tests (this project and any
future one):** "list your tools" is a weaker test than "attempt to invoke
tool X and report the literal result" — a self-report can describe
injected context as if it were a capability; an actual invocation attempt
cannot. Default to the invocation form going forward.

This closes the M2 smoke-test evidence gap more thoroughly than originally
planned: every subagent's tool boundary was confirmed by actual runtime
behavior, not declaration alone.

---

## 2026-09-04 — M3 complete: Constitution + Specify, requirements approved

Spec Kit installed in WhisperFlow (`specify init --here --force --non-interactive
--integration claude`) and run through both phases:

- **Constitution** (`WhisperFlow/.specify/memory/constitution.md`, v1.0.1):
  five principles — Subagent Separation of Powers, Two Human Approval Gates,
  Requirement Traceability, Branch-per-Task/Protected Main, Evidence-Based
  Validation — all directly derived from decisions already logged here and
  from the existing `.claude/agents/*`/`CLAUDE.md`, not invented fresh.
- **Specify** (`WhisperFlow/specs/001-video-subtitle-generator/spec.md`):
  resolves all seven open questions from `docs/ideas/initial-idea.md` —
  CLI-only for v1, local/on-device transcription only (no cloud
  transmission), `.srt` as the sole output format, and, after an explicit
  re-check (see below), **transcription-only for v1 with translation
  deferred as a fast-follow**. Zero `[NEEDS CLARIFICATION]` markers remain
  (verified directly with `grep`, not by trusting the checklist).

**Requirements-ID convention corrected:** the constitution and this plan
originally specified `REQ-001`, `REQ-002`, ... but Spec Kit's own
`spec-template.md` hardcodes the `FR-` (Functional Requirement) prefix —
confirmed by inspecting the template directly, not assumed. Rather than
fight the tool on every future spec, adopted `FR-` as the project-wide
convention: constitution bumped to v1.0.1 (PATCH — wording/ID-format only,
no rule changed) with a proper Sync Impact Report entry, and `CLAUDE.md`/
`implementation-plan.md`/`assessment.md`/the four agent files updated to
match. This also would have silently broken M5's planned traceability
check, which was written to look for a `REQ-` pattern that Spec Kit never
actually produces — caught now, before that check exists, rather than
after it started failing every PR.

**Scope check-in, then reversed back:** during review, flagged that the
spec fully defers translation — not just leaves it open — which drops the
"chosen language" half of WhisperFlow's own one-line description from v1
entirely. Decision was briefly to route translation back into v1 via a
`/speckit.specify` amendment; on reflection, reversed — **v1 stays
transcription-only**. Rationale: the point of WhisperFlow as the pilot is
exercising the pipeline's own machinery (gates, CI, subagents, traceability)
on real-but-small requirements, not maximizing feature scope; adding
translation now would grow the pilot rather than prove the pipeline.
Translation remains a fast-follow after the core loop (M7/M8) is proven.

Also found and fixed in passing: the entire Spec Kit installation
(`.specify/templates/`, `.specify/scripts/`, `.specify/workflows/`,
`.claude/skills/speckit-*`, integration manifests) had never been
committed — only its output was. Committed separately so a fresh clone of
WhisperFlow has Spec Kit itself, not just what it produced.

**Evidence:** `specs/001-video-subtitle-generator/spec.md` +
`checklists/requirements.md` committed and pushed to `main` (WhisperFlow
commits `2a0bc65`, `593a38c`; verified via the repo's own
`.git/logs/refs/remotes/origin/main` reflog, not just the user's report).
Requirements reviewed and approved as transcription-only v1 — Requirements
Gate closed (Section 6, Principle II).

---

## 2026-09-04 — M4 corrected before starting: use Spec Kit's native Plan/Tasks chain, not a hand-rolled one

Asked "what about /speckit-clarify and /speckit-plan" while getting ready
to start M4 — a fair question that, on checking, surfaced a real
inconsistency rather than a simple answer.

M4 as originally drafted (`architect.md` hand-writes `docs/architecture.md`
+ ADRs, then a manual task list is converted to GitHub Issues by hand)
directly conflicts with decision 4 ("use GitHub Spec Kit as-is ... rather
than hand-rolling an equivalent") and with `assessment.md` Section 14's own
"Recommended V1" text, which already correctly scoped Spec Kit through
Plan/Tasks with the Architect's ADRs as a structured-template layer on
top — not a full custom architecture doc. M4 had drifted from both,
likely because it was drafted before Spec Kit was actually installed and
its skill set inspected: `.claude/skills/` in WhisperFlow already ships
`speckit-clarify`, `speckit-plan`, `speckit-tasks`, and
`speckit-taskstoissues` — the exact chain M4 was about to reinvent by hand.

**Corrected, before any M4 work started:**
- Design artifacts now come from Spec Kit's native chain — `/speckit.clarify`
  -> `/speckit.plan` -> `/speckit.tasks` -> `/speckit.taskstoissues` —
  landing under `specs/001-video-subtitle-generator/` (`plan.md`,
  `research.md`, `data-model.md`, `contracts/`, `quickstart.md`, `tasks.md`).
- No separate hand-authored `docs/architecture.md`. `docs/adr/` is kept as
  a companion layer: `architect.md` writes one ADR per significant call
  `plan.md` surfaces, per decision 6, but doesn't author the whole design
  from scratch.
- `/speckit.clarify` runs before `/speckit.plan`, matching Spec Kit's own
  recommended order (it warns of increased rework risk if skipped).

Updated: `WhisperFlow/.specify/memory/constitution.md` (Documentation &
Artifact Structure section rewritten; bumped 1.0.1 -> 1.1.0, MINOR per its
own governance rule — materially changed guidance, no Core Principle
touched), `WhisperFlow/CLAUDE.md` ("Where things live" section, and its
"What this repo is" line corrected to say transcription-only v1 instead of
repeating the "chosen language" framing we already resolved out of scope),
`implementation-plan.md`'s M4 (this file). `assessment.md` needed no
change — Section 14 was already right; only `implementation-plan.md` had
drifted from it.

**Lesson:** the same "verify before advising" discipline used for the
FR-/REQ- fix applies here — a milestone written before a tool is actually
installed needs to be re-checked against what that tool turns out to
provide, not assumed correct because it was written down first.

---

## 2026-09-04 — architect role narrowed, not removed

Asked whether `architect` should be removed now that M4 routes design work
through Spec Kit's native Plan/Tasks chain instead of a hand-authored
architecture doc. Checked before answering: `/speckit.plan`,
`/speckit.tasks`, and `/speckit.taskstoissues` each start by running a
PowerShell setup script (`.specify/scripts/powershell/*.ps1`), which needs
a shell tool — `architect.md`'s allowlist (`Read, Grep, Glob, Write`, no
`Bash`) genuinely cannot run them. Confirms those commands were always
meant to run in the interactive session, not via a subagent.

Decision: keep `architect`, narrowed to what's actually left — reading
Spec Kit's `plan.md`/`research.md`/`data-model.md` after `/speckit.plan`
runs, and writing one ADR per significant decision into `docs/adr/`
(decision 6's structured template). Removing the role outright would mean
amending Principle I (Subagent Separation of Powers), which names all five
roles explicitly — a MAJOR constitution bump for redefining a Core
Principle — and the justification isn't there: ADR-writing is real,
ongoing work that still needs an owner, and keeping it isolated to
Read+Write (no Bash, no Edit) is a genuine, cheap safety property.

Updated `WhisperFlow/.claude/agents/architect.md` (description, tools
unchanged, responsibilities rewritten to drop `docs/architecture.md` and
task-breakdown, explicit "you don't run Spec Kit slash commands" hard
rule) and `implementation-plan.md`'s M2 table (architect row corrected to
match, cross-referencing this entry). No constitution change needed —
Principle I only names the five roles, it doesn't specify each one's exact
job.

---

## 2026-09-04 — Clarify/Plan/Tasks closed; Analyze added to M4 before Implement

`/speckit.clarify` and `/speckit.plan` reviewed and committed
(WhisperFlow `e652071`): Clarify resolved 4 real gaps (max video length
firmed to 2h, no-resume-on-interrupt behavior, a new progress-indicator
requirement FR-011, a proportional processing-time target SC-006) with
zero `[NEEDS CLARIFICATION]` markers left. Plan's `research.md` gives a
real decision/rationale/alternatives per dependency (faster-whisper,
ffmpeg subprocess, `srt`, typer+rich); `contracts/cli.md` answered a
concern raised at Specify time — FR-009/FR-010 (review/edit) is a real
`--review`/`--no-review` flow via `$EDITOR`, not a vacuous restatement of
"it's a text file." Constitution Check in `plan.md` passed with no
violations. Also fixed in passing: no `.gitignore` existed in WhisperFlow;
added one (Claude Code's own `.claude/settings.local.json` convention,
plus basic Python hygiene now that the stack is locked in).

`/speckit.tasks` reviewed and committed (`b86dfff`): 27 tasks, organized
by user story. Verified independently (not from the tool's own completion
report) that all eleven FRs map to at least one task and the `[P]`/story
tagging matches the real dependency graph. Two inaccuracies caught in the
tool's self-reported completion summary: it claimed every task carries an
inline `FR-xxx` reference (T001–T003 and T008 reference `plan.md`/
`research.md`/`contracts/cli.md` instead — legitimate for scaffolding/
contract-surface tasks, but the summary overstated it as universal), and
it named T025 as one of "two file-path-less sweep tasks" alongside T027,
when T025 does name an exact file (`scripts/benchmark.py`) — only T027
genuinely lacks one. Same lesson as the earlier MCP-leak investigation:
a tool's own summary of its output is not a substitute for reading the
output.

**M4 gap closed:** the completion report flagged `/speckit.analyze` as an
optional next step; checked what it does before deciding — a non-
destructive cross-artifact consistency check across spec/plan/tasks,
explicitly meant to run after Tasks and before Implement. M4 hadn't
included it (same category of oversight as the original hand-rolled M4,
just one step later in the chain). Added it to `implementation-plan.md`'s
M4, between Tasks and the architect/Tasks-to-Issues steps.

---

## 2026-09-04 — M4 Analyze pass: 1 CRITICAL + 5 lower-severity findings, all fixed

`/speckit.analyze` run against spec.md/plan.md/tasks.md, cross-checked
against the constitution. Report claimed 100% FR coverage, 0 ambiguity,
0 duplication, 1 critical issue. Spot-checked the CRITICAL finding and
the four lower ones against the actual files before acting (not taken on
the report's word alone) — all confirmed real:

- **D1 (CRITICAL)**: tasks.md said task branches are cut "off
  001-video-subtitle-generator" — but no such branch was ever created
  (verified: `git branch -a` shows only `main`; no `.specify/extensions.yml`
  hooks exist to create one). Every Spec Kit artifact has been on `main`
  directly the whole time. Fixed tasks.md's branching note and plan.md's
  stale `Branch: 001-video-subtitle-generator` header to reflect reality
  — simpler than the report's own suggested fix (merge a feature branch
  that turned out not to exist).
- **F1/F2 (MEDIUM)**: plan.md's Project Structure comments didn't match
  where tasks.md actually places things (`src/lib/` described as holding
  shared types, but only error types land there; review/edit support
  described under `src/subtitles/`, but T020 puts it in `src/cli/review.py`).
  Corrected plan.md's comments to match tasks.md, the more specific and
  already-considered source.
- **C1 (LOW)**: `scripts/` and `tests/fixtures/` (introduced by T003/T025)
  were missing from plan.md's Project Structure tree. Added.
- **E1/E2 (MEDIUM)**: SC-003/SC-004 (accuracy/sync percentages) and
  SC-002/SC-006 (timing) had measurement but no action-on-result. Extended
  T025 to report accuracy/sync against a labeled corpus (with an explicit
  note that SC-003/SC-004 are spec.md's own human-judgment criteria that
  this anchors, not replaces — qa-owned per Principle V) and added T028,
  conditional on T025 showing the timing target missed.

All fixes committed to WhisperFlow (`28666de`) as manual edits to
plan.md/tasks.md, not by re-running `/speckit.plan`/`/speckit.tasks` from
scratch, per the report's own recommendation. M4's Analyze evidence bar
("no unresolved CRITICAL findings") is now met.

Also worth noting for future smoke tests: this is a second, independent
confirmation that a Spec Kit command's own completion report shouldn't be
taken at face value — the Tasks completion report itself had two minor
inaccuracies (see the prior entry), and now Analyze's CRITICAL finding
was real but its own suggested remediation assumed a branch existed that
didn't. Verify against the actual files every time, not just this once.

Next: `architect.md` writes ADRs from plan.md/research.md, then
`/speckit.taskstoissues` converts tasks.md into GitHub Issues.

---

## 2026-09-04 — D1 fixed at the root, for future features

Asked how D1 gets addressed for future features, since the earlier fix
only patched this one feature's tasks.md/plan.md. Traced where the bad
text actually came from before answering: not from Spec Kit's own
templates (checked `tasks-template.md` directly — no branch-related text
in it at all), but inferred by the model running `/speckit.tasks` from
`plan.md`'s templated `**Branch**: [###-feature-name]` field, which gets
populated with a plausible-looking name regardless of whether a real
branch exists — and none does, since `.specify/extensions.yml` hooks were
never configured (confirmed via `git branch -a`: only `main`).

Since `/speckit.plan` and `/speckit.tasks` both load `constitution.md`
before writing that text, fixed it there instead of relying on catching
this again per-feature: Principle IV (Branch-per-Task, Protected Main)
now explicitly states this project doesn't use Spec Kit's hook-based
branch-per-feature model, that a `plan.md` `Branch` field is a template
placeholder to verify rather than trust, and that branches only enter the
picture at the implementation-task level (WhisperFlow `4bdff9b`,
constitution bumped 1.1.0 -> 1.2.0). `CLAUDE.md`'s matching line updated
too.

This is also the answer for any future second project: the same
`.specify/extensions.yml`-hooks-not-configured gap will exist there too
unless deliberately set up otherwise, so a new project's M3-equivalent
constitution should include the same clarifying paragraph from the start
rather than waiting to rediscover it via another CRITICAL Analyze finding.

---

## 2026-09-04 — Reversed: a feature branch SHOULD have existed for 001-video-subtitle-generator; MUST going forward

The prior entry's fix ("this project doesn't use Spec Kit's hook-based
branch-per-feature model") was wrong on reflection: "I think never
creating `001-video-subtitle-generator` branch was a mistake on our
side." Reversed the root-cause fix rather than patching around it again.

Corrected `constitution.md` Principle IV (Branch-per-Task, Protected
Main, bumped 1.2.0 -> 1.2.0 final wording — see WhisperFlow `8ae8944`
after a rebase, superseding the reverted `4bdff9b` text) to require a
real feature branch for Spec Kit's planning phases going forward:
create and check out a branch matching the feature directory name
(e.g. `001-<slug>`) before `/speckit.specify`; `/speckit.specify`
through `/speckit.analyze` and `architect`'s ADRs all commit to that
branch; it merges to `main` via PR once the plan is approved (Design
Gate) — that merge *is* the approval action. `001-video-subtitle-generator`
is documented as a one-time, grandfathered exception, since the
requirement wasn't written down when that work happened. `CLAUDE.md`,
`tasks.md`'s Notes, and `plan.md`'s Branch header updated to match.

Also fixed the same gap at the source: `implementation-plan.md` M3 now
has a MUST-level instruction to create the feature branch before
Specify, and M4 now has a matching MUST-level instruction to merge that
branch to `main` via PR once Analyze is clean and the plan is approved,
with M4's evidence bar updated to require the merge. This is the
permanent fix — every feature after this one creates its branch as a
matter of following M3, not as something to remember to check for
separately.

Mid-rebase note: resolving this alongside an unrelated divergent-history
issue (local `main` had been `git reset` behind `origin/main` by one
commit, then diverged again with fresh local content) surfaced 3+3 real
conflicts in `plan.md`/`tasks.md`, all following the same shape (the
older, superseded branch-model text vs. the newer, decision-correct
text) — resolved keeping the newer side in both files.

---

## 2026-09-04 — architect writes 4 ADRs for 001-video-subtitle-generator; Design Gate approved

`architect` subagent (tools: Read, Grep, Glob, Write only — no Bash, no
code-editing tools) ran against `plan.md`/`research.md`/`data-model.md`,
producing `docs/adr/0001-local-asr-engine.md` through
`0004-cli-framework.md` (local ASR engine, audio extraction, subtitle
composition, CLI framework/progress display), each following
Context/Decision/Alternatives Considered/Consequences and tracing to
specific FR-/SC- IDs.

First invocation (non-interactive `claude --agent architect -p "..."`)
falsely appeared framework-side to have nothing left to do — but a
"architect is done" report was checked, not trusted: `find docs/adr`
and `git status --short` showed nothing had actually been written.
The subagent's own transcript explained why: Claude Code's *session-level*
permission gate (separate from the subagent's own `tools:` frontmatter
allowlist, which does grant `Write`) requires interactive approval for a
tool touching a new path, and `-p` mode can't prompt for that — it
silently declined the writes instead of erroring loudly. Re-run
interactively (no `-p`), with the write prompt approved live, produced
the 4 real files plus an auto-created `.claude/settings.json`
(`{"permissions": {"allow": ["Edit(docs/adr/*.md)"]}}`) — verified again
independently (`find`/`cat`) rather than taking the second "done" report
at face value either, since the first one had just been wrong.

Reviewed all 4 ADRs against `research.md`'s per-dependency
Decision/Rationale sections, `plan.md`'s Technical Context, and
`spec.md`'s FR-/SC- IDs — all four check out (correct structure, correct
traceability, consistent with the source artifacts). `.claude/settings.json`
committed as shared config (not gitignored) since it's not
machine-specific — it's a scoped, reviewable statement of what
`architect` may touch, and committing it means a fresh clone doesn't hit
the same non-interactive permission wall this run did. Both the ADRs and
that settings file committed together (WhisperFlow `6bb44db`).

Design Gate (Principle II) approved for this feature's ADRs. Push to
`origin/main` still pending (this session cannot push — no GitHub
credentials in the sandbox). Next: `/speckit.taskstoissues` converts
`tasks.md`'s 29 tasks into GitHub Issues, closing M4.

---

## 2026-09-04 — M4 closed: taskstoissues verified, all 29 tasks on GitHub

A "done" report (`gh issue list`/`gh issue create` via a script parsing
`tasks.md`, since the GitHub MCP server wasn't available) was not taken
at face value — this session can't reach GitHub itself (private repo,
no `gh` CLI or credentials in either this sandbox or the device-linked
shell, confirmed by trying both), so asked for `gh issue list` output
instead of accepting the claim. Output showed 29 of 29 issues,
`#1`-`#29`, titles matching `tasks.md`'s `T001`/`T025`/`T029` lines
exactly (already independently pulled from the file beforehand, so this
was a real cross-check, not just reading the report back).

M4 marked done in `implementation-plan.md`, with one nuance recorded
inline: the milestone's "MUST open a PR from the feature branch, merge
*is* Design Gate approval" step doesn't literally apply to this
feature, since `001-video-subtitle-generator` is the documented,
one-time exception with no feature branch at all (Principle IV) — its
Design Gate approval was your direct sign-off on the ADRs instead.
Every feature after this one goes through the PR-merge mechanic as
written.

Next: M5 — CI (build/test/lint), Dependabot + code scanning, the
FR-/issue-ID traceability check, updated branch protection, and pinning
third-party Actions to a SHA — must all exist before M7's first real
implementation task.

---

## 2026-09-04 — M5 shipped: CI, CodeQL, Dependabot, traceability, branch protection

Built on `chore/m5-ci-cd-setup` (WhisperFlow), PR against tracking issue
#30: `ci.yml` (build/lint/test), `codeql.yml`, `traceability.yml`,
`dependabot.yml`. Third-party Actions pinned to commit SHA (Section 12)
using SHAs pulled live via `git ls-remote` against the real upstream
repos rather than guessed from training data — this session's knowledge
of "latest" release tags is stale relative to the project's actual date,
so verifying rather than assuming mattered here too, same discipline as
everywhere else this project applies it.

Two real bugs caught before merge, not after:

- **CodeQL's `finalize` step failed fatally** (exit 32, "no source code
  seen during build") on the empty pre-T001 repo — different failure
  mode than `ci.yml`'s jobs, which were written to no-op gracefully on
  the same "no code yet" condition. `ci.yml` controls its own shell
  logic; `codeql-action`'s `init`/`analyze` steps are opaque, so the fix
  was a `git ls-files '*.py'` check exposed via `$GITHUB_OUTPUT`, gating
  those two steps behind `if: steps.check.outputs.has_python == 'true'`
  — same three-piece pattern (detect / expose via output / gate with
  `if:`) as `ci.yml`, not a one-off workaround.
- Considered and rejected a `dummy.py` placeholder as the fix: simpler
  YAML, but it's fake content with no written-down cleanup step —
  exactly the "someone has to remember to do X later" shape this project
  has repeatedly tripped on (D1's phantom branch, the constitution
  branch-model reversal). The check-and-skip approach self-resolves the
  moment T001 lands a real `.py` file; nothing to track or forget.

Branch protection updated last (`gh api PUT branches/main/protection`)
requiring all five checks plus `strict: true`, preserving M1's existing
review/force-push/deletion settings — confirmed live via the response
body itself (`checks[].app_id: 15368` against all five contexts), not
just assumed from the command having exited 0.

Also captured a standing instruction from Philipp: when fixing something
going forward, explain what broke and why in enough depth that he can
reproduce the fix himself, not just apply it silently.

Still open before M5 fully closes: the negative-test evidence
`implementation-plan.md` calls for (a non-referencing throwaway PR
observed to be blocked by `traceability.yml`). Next after that: M6
(security baseline) before M7's first real implementation task.

---

## 2026-09-04 — M5 fully closed: negative-test evidence, two live bugs fixed

Closed out the one item M5 was still missing: opened WhisperFlow PR #34
(branch `test/traceability-negative-check`, a one-line README scratch
edit, title deliberately not referencing anything), confirmed via
`gh pr checks 34` that `traceability` failed while `build`/`lint`/`test`/
`codeql` all passed, then closed it without merging. That's the actual
proof the gate blocks before any real code depends on it, not just that
it passed once on a PR that happened to reference something.

While handling the two Dependabot PRs this surfaced (#32/#33, routine
`codeql-action` SHA bumps — Dependabot's `github-actions` ecosystem entry
already doing its job), the traceability check itself turned out to have
two real bugs, both found live rather than by inspection:

- Checking the PR body (not just title) let #33's auto-generated
  Dependabot changelog — full of `codeql-action`'s *own* upstream PR
  references like `#4072` — produce a false pass. The regex couldn't
  tell "this repo's issue #30" from "some other repo's PR #4072" once it
  was allowed to match anywhere in a long, auto-generated body. Fixed by
  checking the PR title only — short, hand-written, and already the
  pattern used everywhere else in this project (task titles, every PR
  title so far).
- Independently, Dependabot's own PR titles ("Bump X from Y to Z") never
  contain a reference at all and never could — fixed by skipping the job
  entirely for `github.event.pull_request.user.login ==
  'dependabot[bot]'` (a skipped required check counts as passing, so this
  doesn't loosen the gate for anyone who can actually add a reference).

Both fixes shipped together via PR #35 (branch
`fix/traceability-exempt-dependabot`), rebased cleanly onto `main` after
#32/#33 merged ahead of it.

Two more operational snags along the way, neither a bug in our files:
`#33` briefly failed to merge ("head branch is not up to date with base")
after `#32` merged first — `strict: true` on branch protection working
as intended, resolved with `gh pr update-branch`. And the very first
attempt to merge any PR touching `.github/workflows/*` failed with
"refusing to allow an OAuth App to ... without `workflow` scope" — a
GitHub-wide restriction on tokens lacking that scope, unrelated to
anything in this project's own config, fixed with `gh auth refresh -h
github.com -s workflow`.

M5 marked done in `implementation-plan.md`. Next: M6 (security baseline)
before M7's first real implementation task.

---

## 2026-09-04 — M6 closed: credential deny-list, sandbox/token/container decisions documented

Before writing config, verified Claude Code's actual current permission/
sandbox mechanics via `claude-code-guide` rather than assuming from
training data (this session's own knowledge of Claude Code's settings
schema predates whatever version is current) — confirmed `permissions.deny`
syntax, that a `Read` deny also blocks `Edit`/`Write` on the same path
(>= v2.1.228), and the existence and platform limits of the separate
`sandbox.enabled` Bash-sandbox feature.

Shipped to WhisperFlow (`.claude/settings.json` + new `SECURITY-NOTES.md`,
PR #37, tracking issue #36): a `permissions.deny` credential list
(`~/.ssh`, `~/.aws`, `~/.gnupg`, gh's own config, `.netrc`, docker/npm
config, any `.env*`) that applies to every session in this repo regardless
of platform or whether it's interactive or unattended.

Explicitly decided *not* to enable Claude Code's stricter OS-level Bash
sandbox yet, rather than defaulting either way silently — asked Philipp
directly since it's a real trade-off (macOS/Linux/WSL2 only, restricts
Bash writes to working-dir/temp/added-dirs, real friction risk on daily
interactive use) with no dominant answer. Decision: document it as an
available step-up in `SECURITY-NOTES.md`, don't enable now — matches
assessment.md's own "ladder, not a switch" framing, and GitHub Actions'
per-run VM isolation already covers M8's actual unattended execution
environment, so the gap this would close doesn't exist yet in practice.

Also documented, as policy rather than action: the GitHub token scope
required once M8 needs one (fine-grained PAT, WhisperFlow-only, Contents +
PRs + Issues, nothing org-wide) — not created yet since nothing needs it
until M8 starts, and the container-boundary decision (GitHub Actions' own
VM is sufficient for M8; no devcontainer unless unattended work ever runs
locally instead).

M6 marked done in `implementation-plan.md`. Next: M7 — first end-to-end
task, supervised (pick the smallest task from `tasks.md`, prove
Developer -> CI -> Reviewer -> QA -> human merge works before trusting the
loop with anything real).

## 2026-09-04 -- CodeQL/GHAS private-repo gap found during M7 (T001); assessment.md corrected

PR #38 (T001) was the first PR with tracked `.py` files, so it was the
first time CodeQL's `analyze` step actually ran end-to-end rather than
being skipped by the "nothing to scan" guard. It hit two distinct
failures, diagnosed from real log output, not guessed:

1. `##[error]Resource not accessible by integration -
   .../actions/workflow-runs#get-a-workflow-run` -- the scan itself
   (extraction, database build, 45 queries, SARIF export) succeeded
   completely; this error came from CodeQL's own telemetry call to the
   Actions API, which needs `actions: read`. Once any `permissions:`
   block is declared on a job, GitHub Actions defaults every unlisted
   scope to `none`, and the original `codeql.yml` only listed
   `security-events: write` and `contents: read`. Fixed by adding
   `actions: read`.
2. Separately, and more fundamentally: SARIF upload to the Security tab
   needs the "GitHub Code Security" product enabled on the repo. Verified
   against current GitHub docs (not assumed from training data, since
   this is exactly the kind of platform-policy detail that changes): as
   of GHAS's March 2025 unbundling, GitHub Code Security -- the half that
   covers code scanning on **private** repos -- is purchasable *only* by
   organizations on GitHub Team or Enterprise plans. An individual GitHub
   Pro account (what this repo runs under) has **no purchase path at
   all**, at any price. Public repos get code scanning free regardless.
   Dependabot (alerts + security updates + version updates) is unaffected
   either way -- fully free on private repos on any plan.

This directly contradicted `assessment.md` Section 12/13's original
claim that code scanning was "cheap/included for private repos on most
plans." That claim was wrong for this account type. Corrected in
`assessment.md` (Sections 12 and 13) to state the actual gating
accurately.

**Decision: stay private, don't make this repo public just to get free
CodeQL upload.** Reasoning: going public would remove the account-type
gate, but it also removes the access control that M6's entire security
baseline (credential deny-list, least-privilege token policy) exists to
defend against -- a public issue tracker and PRs from strangers is
exactly the attack surface `assessment.md` Section 12 describes. Trading
a real increase in attack surface for a code-scanning nicety, on a solo
hobby/pilot project with no stated need for public distribution, isn't a
good trade. Dependabot -- confirmed unaffected by any of this -- remains
the automated dependency/security backbone; CodeQL keeps running as
*analysis* (findings still visible in the workflow's job log, so nothing
is lost except the Security-tab UI and PR-level SARIF annotations) with
`upload: false` on the `analyze` step, and `security-events: write`
dropped from the job's permissions since it's unused once upload is off.
Revisit if the repo ever goes public, or moves under an org on a paid
plan.

Philipp's own Code session applied this fix directly (branch
`fix/codeql-no-ghas-upload`, commit `f5c34fe`) rather than the narrower
`actions: read`-only fix suggested first -- correctly recognizing the
GHAS gate was the deeper, separate problem underneath the permissions
error.
