# AI Dev Pipeline (Planning Project)

This is the planning/reference project for building an AI-powered software
development pipeline — not a pilot project's code repo. It holds the research,
the decisions made along the way, and the implementation plan once a pilot
project is chosen.

## Contents

- `assessment.md` — the full requirements & architecture research assessment
  (lifecycle, agent roles, artifact model, quality gates, security model,
  framework evaluation, recommended V1, evolution path).
- `decisions.md` — a running, dated log of decisions made about how the
  pipeline works. `assessment.md` reflects the decisions known at the time
  it was last updated; `decisions.md` is the place new decisions get logged
  as they're made, so this stays current without rewriting the assessment
  every time.
- `implementation-plan.md` — the concrete build plan for Version 1, once a
  pilot project is chosen. Empty/stub until then.

## Status

- [x] Research & architecture assessment complete
- [x] Open questions from the assessment resolved (see decisions.md)
- [x] Pilot project chosen: **WhisperFlow** (`../WhisperFlow`) — video subtitle generator
- [x] V1 implementation plan written
- [ ] V1 built on the pilot project's repo

## How this relates to an actual project repo

This folder is *not* where pipeline artifacts for a real project live —
per the assessment's recommendation, those (specs, ADRs, task issues, code)
belong inside that project's own GitHub repo (`docs/`, `.claude/agents/`,
GitHub Issues/PRs). This folder is the meta-level record: why the pipeline
is shaped the way it is, and what's still undecided.
