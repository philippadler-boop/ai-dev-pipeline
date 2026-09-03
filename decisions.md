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
