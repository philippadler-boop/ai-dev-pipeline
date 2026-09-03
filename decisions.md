# Decisions Log

A running, dated record of decisions made about the AI dev pipeline, beyond
what's captured in `assessment.md`. Newest entries at the top. When a
decision changes something the assessment already recommends, note it here
rather than silently diverging from that document.

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
