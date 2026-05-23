# Attest — Design Spec

## What it is

Attest is a command-line tool that tracks whether AI coding agents implemented what they said they would. It exists because agents routinely under-deliver on multi-item specs — implementing some requirements, skipping others, and over-reporting completion.

Attest externalizes work state out of the model's context and into a CLI-enforced state machine. Agents claim items, implement code, and call `done`. Attest independently scans the codebase, runs tests, and detects drift — producing evidence that the agent's claims match reality.

Attest is part of the chief-tribe ecosystem but is fully standalone. It composes with chief, council, and reeve but depends on none of them.

## The problem it solves

Three failure modes show up repeatedly when agents work from a spec.

**Interpretation drift** — agent breaks the spec into fewer items than it should, missing requirements silently.

**Implementation drift** — agent claims an item is done without actually implementing or testing it.

**Maintenance drift** — spec changes but tests and code don't follow. Or code changes but no one notices the spec is now stale.

Attest doesn't solve all three on its own. It provides the structured ground truth that lets agents, humans, and other tools detect and fix these drifts cheaply.

## Design principles

**Filesystem is source of truth.** Every artifact is a markdown file with YAML frontmatter. State lives in files, not a database.

**CLI is the only writer.** Agents and humans read files directly, but state transitions go through CLI commands. This makes the integrity boundary explicit and prevents agents from silently editing their way around the state machine.

**Agent has autonomy. Attest has authority.** Agents choose what to work on. Attest enforces state transitions, scans for declared tests, runs verification, and detects drift. The agent's claim is one signal, not the only one.

**Quantitative before qualitative.** Cheap deterministic checks (scan, verify) run on every cycle. Expensive semantic checks (agent review) run on demand.

**Compose, don't bundle.** Attest ships standalone. Composition with chief/council/reeve is opt-in.

## Mental model

Two abstraction levels.

**Flow** — high-level intent. A user journey, a feature area, a milestone goal. Optional but recommended for projects beyond a handful of items.

**Item** — atomic, verifiable unit of work. One spec, one test declaration. The thing the agent actually implements.

Items can live inside flows or flat (without flows). Flows can be mandatory or optional depending on the project.

## Directory layout

```
.attest/
  attest.config.json                 # global config
  default/                           # scope (default | milestone-N | phase-N | ...)
    flows/                           # optional
      F-01-authentication/
        overview.md
        diagram.md                   # optional, mermaid
    items/
      FR-001-login/
        spec.md
        test.md
      FR-002-signup/
        spec.md
        test.md
  milestone-1/                       # another scope
    items/
      FR-100-dashboard/
        spec.md
        test.md
```

Scope is an arbitrary namespace. The directory name has no enforced semantics — it's just a folder. Projects using chief can name scopes after milestones.

## Entities

### Flow

Holds intent. Deattests user journey, why it exists, what's out of scope, and lists implementing items. Flow has its own status (`draft | approved | changed`) but doesn't block items beneath it.

```markdown
---
id: F-01
status: approved
items: [FR-001, FR-002, FR-003]
---

# Authentication

## Why this exists
Users need to access their dashboard.

## User journey
1. New user → /signup → email verification → /dashboard
2. Returning user → /login → /dashboard
3. Forgot password → /reset → email link → new password → /login

## Out of scope
- Social login (later)
- 2FA (later)
```

### Item — spec.md

Deattests what to build. Plain markdown body, agents and humans read this directly.

```markdown
---
id: FR-001
flow: F-01
status: approved
hash: sha256:abc123
depends_on: []
implementation_hints:
  - src/auth/login.ts
  - src/auth/hash.ts
---

# Sign up with email/password

User can authenticate with email + password.

## Acceptance
- Email + password input
- Wrong password → 401
- Success → JWT cookie
- Rate limit on repeated failures
```

### Item — test.md

Declares test names. Each name should match the name of an actual test in your codebase. Body is a markdown bullet list of test names. Read-only from the agent's perspective — agents don't update test.md state, the CLI computes it.

```markdown
---
id: FR-001
status: defined
test_hints:
  - tests/auth/login.test.ts
  - tests/auth/hash.test.ts
verified_against:
  spec: sha256:abc123
  commit: 789xyz
last_scanned: 2026-05-23T10:00:00+07:00
last_verified: 2026-05-23T10:15:00+07:00
scan_result:
  found: 3
  missing: 1
  orphan: 0
verification_result:
  passing: 3
  failing: 0
---

# Tests for FR-001

- hash function
- wrong password returns 401
- successful login sets JWT cookie
- rate limit on failed attempts
```

## State machine

Items move through these states.

```
pending → in_progress → done → verified
                          ↓         ↑↓
                       failing → stale
                      
pending → blocked → pending
pending → skipped
```

| State           | Meaning                                          |
| --------------- | ------------------------------------------------ |
| `pending`     | Item exists, not started                         |
| `in_progress` | Agent claimed work                               |
| `done`        | Agent claims complete (not yet checked)          |
| `verified`    | CLI confirmed: scan + verify pass                |
| `failing`     | Verify ran, tests failed                         |
| `stale`       | Spec content or commit changed after last verify |
| `blocked`     | Dependency unmet, or external block              |
| `skipped`     | Intentionally not done                           |

Only the CLI transitions states. Agents call commands; they do not edit frontmatter.

## Dependencies

Items declare `depends_on: [FR-001, FR-002]` in spec.md frontmatter.

Scope-local — an item can only depend on items in the same scope. Cross-scope dependencies are out of scope for v0.1.

CLI enforces:

* `attest start FR-003` rejected if any item in `depends_on` is not in state `verified` (or `done` if verify is optional, configurable)
* `attest list --ready` shows items whose dependencies are met
* Cycle detection on init/load — refuse to start if cycle detected

## Verification — three passes

Attest separates verification into layers by cost.

### Pass 1: Scan (cheap, deterministic)

`attest scan` does text-based search for declared test names in your codebase. Case-insensitive substring match, scoped to test files (configurable patterns). Reports which test names are found, which are missing, and which exist in code but aren't declared (orphans).

Cost: milliseconds, zero LLM tokens. Suitable for every agent loop iteration.

### Pass 2: Verify (run tests)

`attest verify` invokes your test runner via configured command, captures CLI output, and matches test names against declarations to determine pass/fail per declared test. Output capture is framework-agnostic — Attest doesn't parse JSON reporters or AST.

Cost: seconds, zero LLM tokens. Suitable for pre-commit, CI, milestone gates.

### Pass 3: Review (semantic)

`attest review` is a delegation hook. It doesn't do semantic review itself — it bundles spec.md, test.md, linked code, and verification results into a payload for an external reviewer (typically an agent invoked via chief or council). The reviewer judges whether tests actually verify the spec or just match names.

Cost: LLM tokens. On-demand only — milestone reviews, audits, contentious items.

## Drift detection

Three drift types, detected differently.

| Drift type | Trigger                         | Detection                                      | Severity                          |
| ---------- | ------------------------------- | ---------------------------------------------- | --------------------------------- |
| Spec drift | `spec.md`body changed         | Hash comparison with `verified_against.spec` | Definite                          |
| Test drift | `test.md`declarations changed | Parse current list vs verified list            | Definite                          |
| Code drift | Code changed after verify       | Compare HEAD with `verified_against.commit`  | Potential — re-verify to confirm |

When `implementation_hints` are present, code drift narrows via `git diff` — `attest affected` lists items whose hinted files changed. This is TIA-lite — selective, not bulletproof, but cheap.

## CLI surface

```bash
# discovery
attest list                          # all items in current scope
attest list --status pending
attest list --ready                  # pending items with deps met
attest list --format json            # for agents to parse
attest show FR-001
attest stats
attest coverage                      # RTM-style coverage report

# state transitions
attest start FR-001
attest done FR-001
attest block FR-001 --reason "..."
attest skip FR-001 --reason "..."
attest reset FR-001

# verification — three passes
attest scan FR-001                   # Pass 1
attest scan                          # all items
attest verify FR-001                 # Pass 2
attest verify --affected             # only items with recent diff in hints
attest review FR-001                 # Pass 3 — package context for reviewer

# drift
attest stale
attest diff FR-001
attest affected --since=HEAD~5

# scope
attest scope                         # show current
attest scope set milestone-1
attest --scope milestone-1 list

# init
attest init
attest init --with-flows
```

No `next` command. Agents pick their own work using `list` and decide order based on their own context.

## Configuration

`.attest/attest.config.json`, all fields optional with sensible defaults.

```json
{
  "scope": {
    "default": "default"
  },
  "item": {
    "id_pattern": "^[A-Z]+-\\d{3,}$"
  },
  "scan": {
    "test_file_patterns": ["**/*.test.ts", "**/*.spec.ts"],
    "case_sensitive": false,
    "require_deattest_block": false
  },
  "verify": {
    "command": "bun test",
    "capture": "stdout",
    "gates": {
      "require_test_link": true,
      "require_commit": false
    }
  },
  "review": {
    "command": "claude --context @attest-context.json"
  },
  "dependencies": {
    "block_state": "verified"
  }
}
```

`dependencies.block_state` controls whether dependencies must be `verified` (strict) or `done` (lenient) before dependent items can start.

## Agent loop contract

```
1. attest list --status pending --ready --format json
   → agent receives list, picks one based on its own logic

2. attest show FR-005
   → agent reads spec, test declarations, flow context

3. attest start FR-005
   → state: pending → in_progress (rejected if deps unmet)

4. agent implements code + writes tests in codebase

5. attest done FR-005
   → state: in_progress → done (gated by verify.gates)

6. (optional, cheap) attest scan FR-005
   → confirms declared tests exist in code

7. (CI or milestone) attest verify FR-005
   → runs tests, transitions to verified or failing

8. goto 1
```

Agents have full autonomy in choosing work order. The CLI guards transitions, scan checks structural presence, verify checks behavioral correctness, and review (when invoked) checks semantic alignment.

## Composition with chief-tribe

**With chief** — Attest scopes map to chief milestones. Items can be generated by a chief skill that breaks milestone goals into atomic items. The chief agent loop uses Attest commands as its work-tracking layer.

**With council** — items flagged for review become artifacts that humans approve before merge. Attest's frontmatter and scan/verify results provide council with evidence.

**With reeve** — `verify.command` and `review.command` can run inside reeve's allowlist for safe execution.

None are required. Attest ships standalone.

## What Attest does not do

* Does not generate items from a spec. That's a proposer agent's job; Attest consumes its output.
* Does not run agents, schedule them, or coordinate parallel work.
* Does not store implementation code or test code.
* Does not judge whether tests are good — only whether declared tests exist (Pass 1) and pass (Pass 2). Quality judgment is Pass 3, delegated.
* Does not provide a web UI or dashboard. CLI + json export only.
* Does not detect code drift definitively — uses commit hash as proxy.

## Tech stack & deliverable

TypeScript on Bun, distributed via npm. Frontmatter parsing via gray-matter, schema validation via zod, state machine as hand-rolled transition table. Single binary via `bun build`.

Day-1 deliverable:

* `attest` CLI binary
* `attest init` scaffolding
* Reference `attest.config.json`
* One worked example (5–10 items, spec + test + implementation)
* README with the agent loop contract

---

## Open Questions

### Q1 — Item ID format

Strict (`^[A-Z]+-\d{3,}$`) or free-form? Strict helps tools, free-form helps adoption. **Lean: configurable via `item.id_pattern`, default to strict.**

### Q2 — Multi-agent concurrency

Two agents call `attest start` simultaneously on the same item. File locking, atomic claim via rename, or document as single-agent for v0.1? **Lean: single-agent in v0.1, design lock primitive for v0.2.**

### Q3 — Body schema enforcement

Should certain sections in spec.md (`## Acceptance`, `## Out of scope`) be required and parsed? **Lean: optional convention in v0.1, no enforcement.**

### Q4 — Test name resolution across deattest blocks

If two items have a test named "validates input", scan can't tell them apart. Options: require `deattest('FR-001', ...)`, rely on file location, or full-name match (`FR-001 > validates input`). **Open — likely configurable.**

### Q5 — Test framework support strategy

Output capture is universal, but exit codes and test name extraction vary. **Lean: capture raw stdout/stderr, run grep-based name detection, document edge cases per framework.**

### Q6 — Hash sensitivity

What changes count as spec change? Whitespace only? Capitalization? Frontmatter changes? **Lean: hash body content with normalized whitespace; ignore frontmatter since CLI manages it.**

### Q7 — Flow ↔ item sync direction

If `items: [...]` in flow overview and `flow: F-01` in item frontmatter both exist, which wins? **Lean: item is the source of truth; CLI auto-updates flow's items list on mutation.**

### Q8 — Cross-scope dependencies

Can item in milestone-2 depend on item in milestone-1? **Lean: no in v0.1, scope-local only.**

### Q9 — Item rename / scope move

Moving FR-001 from `milestone-1` to `milestone-2` — manual directory move, or CLI command that updates hashes and refs? **Open.**

### Q10 — Review payload format

What does `attest review` package look like — JSON, markdown bundle, structured for specific agents? **Open — possibly multiple output formats.**

### Q11 — Flow without items

A flow exists with no items yet — draft of intent or misconfiguration? **Lean: allow it; warn but don't error.**

### Q12 — Spec drift acceptance

User reviews a spec change and confirms test+code are still valid. Command to mark drift as accepted without re-running verify? **Lean: `attest accept-drift FR-001 --spec` with clear warning.**

### Q13 — Audit / history

v0.1 has no audit log (rely on git). What if user wants `attest history FR-001`? **Lean: derive from git log on `.attest/` directory; no separate audit file.**

### Q14 — Coverage threshold gates

Should Attest support "fail if coverage < N%"? **Lean: not in v0.1; user can script with `attest stats --format json`.**

### Q15 — Item status when test.md is empty

Can an item with no test declarations be `verified`? **Lean: no; require ≥1 test declaration. Configurable via `verify.gates`.**

### Q16 — Scope vs flow conceptual overlap

Both group items. When does a user use scope (milestone) vs flow? **Lean: scope = lifecycle phase (milestone, sprint), flow = functional area (auth, dashboard). Document with examples.**

### Q17 — Scan result invalidation

When does `scan_result` in frontmatter become stale? **Lean: invalidate when test.md hash or HEAD commit changes; re-scan cheap enough to redo.**

### Q18 — Reviewer trust boundary

Pass 3 invokes an agent. Does that agent get write access to Attest (mark verified) or read-only (return verdict, human applies)? **Open — security-relevant.**

### Q19 — Dependency cycles

If user creates a cycle (FR-001 deps on FR-002, FR-002 deps on FR-001), how does CLI behave? **Lean: detect on load, refuse to start any item in cycle, output cycle path.**

### Q20 — Optional dependencies on flows

Can an item depend on an entire flow being complete (`depends_on: [F-01]`)? Useful but conflates entity types. **Lean: no in v0.1, items only.**
