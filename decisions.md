# Decisions Log

A running, dated record of decisions made about the AI dev pipeline, beyond
what's captured in `assessment.md`. Newest entries at the top. When a
decision changes something the assessment already recommends, note it here
rather than silently diverging from that document.

---

## 2026-09-03 — M1 complete: WhisperFlow pushed to GitHub, two deviations from plan

WhisperFlow is now at https://github.com/philippadler-boop/WhisperFlow with
branch protection on `master` requiring a pull request before merging. Two
things came up during M1 that the plan (`implementation-plan.md`) didn't
anticipate:

1. **Repo is public, not private.** GitHub's classic branch-protection API
   returns 403 ("Upgrade to GitHub Pro or make this repository public") for
   private repos on this account's plan. Since `implementation-plan.md`
   already noted "nothing here needs to be public," made it public rather
   than skip branch protection or pay for Pro. Revisit if this account's
   plan changes, or if a future pilot project actually needs privacy.
2. **Commit author email amended before first push.** The repo's sole
   existing commit was authored with `philipp.adler@gmail.com`, from before
   the global git config was switched to the GitHub noreply address. GitHub
   rejects pushes that would publish a private email, so the commit was
   amended (safe — it had never been pushed anywhere) to use
   `243643600+philippadler-boop@users.noreply.github.com`, matching the
   global config used for everything since.

**Also installed:** GitHub CLI (`gh`, via `winget install --id GitHub.cli`),
authenticated as `philippadler-boop` — needed for M1 and everything
GitHub-related in M4 onward (Issues, PR traceability checks, etc.).

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
