# Grill: Attest Spec V1

## Design Clarification (fundamental)

**CLI-as-canonical-interface.** The final goal is full automation. Agents are first-class callers of CLI commands. "CLI is the only writer" means: CLI is the canonical interface for state transitions, enforced by prompt/skill convention, not OS-level access control. Direct frontmatter edits are unsupported — they bypass state machine integrity but are not technically blocked. Git is the audit log. This reframes Q6, Q18, and any "security-relevant" framing that assumed agents need to be gated by humans.

## Open Questions

Q4b (opened): Multi-agent is intended to be supported from v0.1 (overrides spec's Open Q2 lean). What is the atomicity mechanism for `attest start`? Options: (1) atomic file rename/move as claim token, (2) OS file lock, (3) optimistic — last writer wins with conflict detection, (4) advisory lock file per item. Depends on Q4.
Q2 (reopened): Where does "scan before done" belong — `verify.gates.require_scan_on_done` (conflates with verify semantics) or a separate `transitions.done.require_scan`? Also: does gating `done` with scan break the "not yet checked" definition, or is structural presence check (scan) a different category from behavioral check (verify)?
Q3b (opened): How should `stale` recovery be configured? Three sub-questions: (1) separate `stale.auto_recover` flag per drift type (spec/test/code)? (2) single `stale.block_ready` boolean? (3) or one `stale` section with per-cause overrides? Depends on Q3.

## Resolved

Q1: What is the primary consumer of `attest list --format json`? → AI agents in an autonomous loop. Why: spec explicitly marks it "for agents to parse"; the agent loop contract frames CLI as the agent-facing runtime interface.
Q2: Should `attest done` auto-run scan or transition immediately? → REOPENED.
Q4: What stops parallel agents from double-claiming an item? → Multi-agent supported from v0.1. Lock file via atomic rename per item folder. Spec amendments: (1) replace Open Q2 lean with lock file decision, (2) remove "Does not coordinate parallel work" from "What Attest does not do", (3) add conflict-error path to agent loop contract step 3. Why: item-per-folder layout supports .claim file naturally; atomic rename is POSIX-safe on local fs.
Q4b: Atomicity mechanism → lock file (atomic rename). Resolved via Q4.
Q5: Who populates `implementation_hints`? → Spec author (human or proposer agent) — hints are intent, written before implementation. They drive `attest affected` for TIA-lite. Why: hints live in spec.md alongside acceptance criteria; pre-implementation intent, not post-hoc discovery. Keeping spec.md stable after writing avoids blurring the spec-is-stable assumption.
Q7: What should `attest coverage` output? → (C) Both: ratio as summary line + item-level traceability matrix as detail. With `--format json`, matrix lets agents find gaps autonomously (which items have ≥1 test declaration, which are verified). Why: bare ratio is unactionable for an agent; matrix is achievable without body parsing (no criterion-level traceability needed in v0.1). Spec amendment: clarify "RTM-style" means item-level matrix, not criterion-level.
Q6: Should `attest review` verdict be writable by the reviewer agent or human-only? → Reviewer agent calls `attest mark-reviewed FR-001 --verdict pass` (a CLI command). CLI writes; git is the audit trail. Why: CLI is an interface for efficiency/consistency, not a security gate — full automation is the goal. Open Q18 resolved: git = audit log, no human gate required. Spec amendments: add `attest mark-reviewed` to CLI surface; add Pass 3 → `verified` transition to state machine; reframe "CLI is the only writer" in design principles to clarify it means interface consistency, not agent restriction.
Q3: Should `stale` be auto-recoverable via `attest verify`, or require human intervention? → Configurable. Default: stale items excluded from `--ready`. Re-verify auto-transitions to `verified`/`failing`. Why: agents run autonomously overnight. **Spec amendments required:** (1) add `stale → verified | failing` arc to state diagram, (2) update `--ready` definition to explicitly exclude stale items, (3) add `verify.auto_recover_stale` config key (or similar) to config section. How to configure the per-drift-type behavior is Q3b (open). Verifier flagged conflict: scan is post-done by design in agent loop contract; gating `done` with scan contradicts "not yet checked" definition; config namespace unclear (`verify.gates` vs `transitions.done`).

## Final Review
