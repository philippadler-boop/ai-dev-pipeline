# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This folder is the planning/reference project for an AI-powered software
development pipeline. It is not a pilot project's own code repo, and it has
no build/test/CI of its own — it's a small set of Markdown files.

## Required reading before acting

- `assessment.md` — the full requirements & architecture research assessment
  (lifecycle stages, agent roles, artifact/contract model, quality gates,
  human-in-the-loop model, security model, framework evaluation, the
  recommended Version 1 architecture in Section 14, and the evolution path
  in Section 15).
- `decisions.md` — a dated, running log of decisions made about how the
  pipeline should work. Newest entries are at the top. If something here
  conflicts with `assessment.md`, `decisions.md` is more current — it's
  meant to be appended to over time rather than requiring the assessment
  to be rewritten.
- `implementation-plan.md` — the concrete, milestone-by-milestone (M0-M9)
  build plan for V1, targeting the pilot project. Its checkboxes are the
  source of truth for what's actually done; don't infer status from memory
  or from this file.

## How the pipeline is meant to work

These are the load-bearing decisions behind every recommendation in
`assessment.md` and `implementation-plan.md` — worth knowing before acting
on either, or before advising on the pilot project itself:

- **Stage-gate lifecycle**: Idea → Concept/Requirements → Architecture/Design
  → Implementation → Test → Review → Validation → Release → Maintenance
  (`assessment.md` Section 2). Concept and Requirements collapse into one
  lightweight doc for anything smaller than a multi-week feature.
- **Five V1 subagent roles**, defined per pilot project in *that project's*
  `.claude/agents/` (not here): `analyst`, `architect`, `developer`,
  `reviewer`, `qa`. The `reviewer` role never gets `Write`/`Edit` — a
  reviewer that can fix what it's reviewing isn't an independent review;
  this is called out in `assessment.md` Section 3 as the single most
  load-bearing separation in the whole design.
- **Human approval gates at exactly two points**: Requirements/Design
  sign-off, and Merge/Release. Everything else (implementation, testing,
  drafting a review) proceeds without blocking, with a summary available
  rather than a prompt (`decisions.md`, "Approval style").
- **Unattended agents are explicitly allowed** to pick up issues and open
  PRs without real-time supervision, even for non-trivial tasks
  (`decisions.md` #3). This is *why* the security baseline (credential
  deny-lists, repo-scoped GitHub tokens, container/CI sandboxing, pinned
  third-party Actions) is mandatory from V1 rather than deferred — broader
  autonomy is what makes the stricter posture necessary.
- **Traceability is enforced by CI, not convention**: a required check
  fails any PR whose title/body doesn't reference a `REQ-`/issue ID, since
  unattended agents won't self-police that link the way a human would.

## Working conventions for this folder

- New decisions get appended to `decisions.md` (newest entry at the top),
  not silently absorbed into `assessment.md`.
- Real pipeline artifacts for an actual project (specs, ADRs, `.claude/agents/`
  subagent definitions, GitHub Issues/PRs) belong in that project's own repo,
  not here — see `assessment.md`, Section 5 (Artifact & Contract Model) and
  Section 14 (Recommended V1) for what that looks like.
- Keep this folder itself lightweight: it's a small set of Markdown files,
  not a project with its own build/test/CI.

## Known open gap

Where "shared project memory" (conventions, architecture summaries, reusable
subagent templates) should live once there's a second pilot project is
explicitly unresolved (`decisions.md`, bottom of the first entry). Don't
invent an answer for this — it's deliberately deferred until a second
project exists to make the question concrete.
