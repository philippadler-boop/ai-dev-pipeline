# CLAUDE.md

This folder is the planning/reference project for an AI-powered software
development pipeline. It is not a pilot project's own code repo.

Before acting on anything in this folder, read:

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
- `implementation-plan.md` — the concrete build plan for V1. Empty until a
  pilot project is chosen; once it isn't, treat it as the actual task list.

## Working conventions for this folder

- New decisions get appended to `decisions.md` (newest entry at the top),
  not silently absorbed into `assessment.md`.
- Real pipeline artifacts for an actual project (specs, ADRs, `.claude/agents/`
  subagent definitions, GitHub Issues/PRs) belong in that project's own repo,
  not here — see `assessment.md`, Section 5 (Artifact & Contract Model) and
  Section 14 (Recommended V1) for what that looks like.
- Keep this folder itself lightweight: it's a small set of Markdown files,
  not a project with its own build/test/CI.
