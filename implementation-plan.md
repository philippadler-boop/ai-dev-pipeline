# V1 Implementation Plan

Status: not started — waiting on a pilot project (see README.md).

Once a pilot project is chosen, this document becomes the concrete,
ordered build plan for the Version 1 architecture in `assessment.md`
(Section 14), covering at least:

- Repo setup: branch protection rules, required status checks
- `.claude/agents/` subagent definitions (analyst, architect, developer,
  reviewer, qa) with tool allowlists
- GitHub Spec Kit installation and first Constitution/Specify pass
- CI workflow: build, test, lint, Dependabot, code scanning, traceability check
- ADR template file
- Security baseline: credential deny-list, scoped tokens, sandboxing setup
- First end-to-end run of one small feature through the full pipeline, as
  a validation that the gates and handoffs actually work before relying on
  them for real work
