# Attest — Design Spec (v2)

> Revised from spec-v1 against grill `0001-attest-spec-v1` (11 resolutions). Changes: CLI-as-canonical-interface framing, multi-agent `.claim` lock + TTL, Pass 3 `review` payload + hash-guarded `mark-reviewed`, drift recovery with spec-drift carve-out, scan `ambiguous` category, coverage matrix output.

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

**CLI is the canonical interface.** Agents and humans read files directly, but state transitions go through CLI commands. This is a consistency boundary, not access control — direct frontmatter edits are unsupported but not technically blocked. CLI exists for efficiency and integrity; git is the audit log. Full automation is the goal: agents are first-class callers, not gated by humans.

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
        .claim                       # present only when in_progress
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

Holds intent. Describes user journey, why it exists, what's out of scope, and lists implementing items. Flow has its own status (`draft | approved | changed`) but doesn't block items beneath it.

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

Describes what to build. Plain markdown body, agents and humans read this directly.

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

`implementation_hints` are populated by the spec author (human or proposer agent) — pre-implementation intent, written alongside acceptance criteria, not derived post-hoc. They drive `attest affected` for TIA-lite and are emitted as path references in the review payload.

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
  ambiguous: 0
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

`test_hints` indicate where the tests live. Used as a tie-breaker for scan disambiguation (see Pass 1) and for `attest affected`. They are not a hard scope.

### Item — .claim (when in_progress)

Created by `attest start` via atomic `rename(2)`. Present only while the item is `in_progress`. Removed on any transition out of `in_progress`.

```yaml
agent: builder-agent-7
token: 9f3b2a1c
started_at: 2026-05-30T14:23:11+07:00
```

The `token` is a per-claim random id used for conflict detection at `done`. See **Multi-agent (lock file)** below.

## State machine

Items move through these states.

```
pending → in_progress → done → verified
                          ↓         ↑↓
                       failing → stale

pending → blocked → pending
pending → skipped
```

| State           | Meaning                                                        |
| --------------- | -------------------------------------------------------------- |
| `pending`     | Item exists, not started                                       |
| `in_progress` | Agent claimed work; `.claim` file present                      |
| `done`        | Agent claims complete; behaviorally unverified                 |
| `verified`    | CLI confirmed: scan + verify pass (or Pass 3 verdict accepted) |
| `failing`     | Verify ran, tests failed                                       |
| `stale`       | Spec body, test declarations, or commit changed after last verify |
| `blocked`     | Dependency unmet, or external block                            |
| `skipped`     | Intentionally not done                                         |

Transitions:

- `pending → in_progress` via `attest start` (creates `.claim` atomically; rejected if claim already held and lease not expired, or if any `depends_on` item is not verified).
- `in_progress → done` via `attest done` (gated by `verify.gates`; rejected if claim token does not match — see lock lifecycle; removes `.claim`).
- `done → verified | failing` via `attest verify`.
- `verified → stale` automatic on spec body hash, test declarations, or HEAD commit divergence from `verified_against` (drift).
- `stale → verified | failing` via `attest verify` — **for code drift or test drift only** (`stale.auto_recover`).
- `stale → verified` for **spec drift**: either `attest mark-reviewed FR-001 --verdict pass --against-spec <hash>` (Pass 3; hash must match current `spec.md` hash) **or** `attest accept-drift FR-001 --spec` (shallow consent, no review).
- `done | failing → verified` via `attest mark-reviewed --verdict pass --against-spec <hash>` (Pass 3 alternative path).
- `pending → blocked | skipped` via `attest block | skip`.

Only the CLI transitions states. Agents call commands; they do not edit frontmatter.

## Dependencies

Items declare `depends_on: [FR-001, FR-002]` in spec.md frontmatter.

Scope-local — an item can only depend on items in the same scope. Cross-scope dependencies are out of scope for v0.1.

CLI enforces:

* `attest start FR-003` rejected if any item in `depends_on` is not in state `verified` (or `done` if verify is optional, configurable via `dependencies.block_state`).
* `attest list --ready` shows items whose dependencies are met (and excludes stale items by default).
* Cycle detection on init/load — refuse to start if cycle detected.

## Multi-agent (lock file)

Attest coordinates concurrent agents via a `.claim` file in each item folder. `attest start FR-001` creates `.claim` via atomic `rename(2)` — POSIX-safe on local filesystems. Exactly one caller wins; losers receive a conflict error and should return to `attest list`.

`.claim` contents:

```yaml
agent: <id>
token: <random>
started_at: <iso8601>
```

**Lifecycle:**

- Auto-released on any transition out of `in_progress` (`done`, `block`, `skip`, `reset`).
- `lock.ttl` configures the lease (default `"15m"`). `attest start` on an item whose `started_at` is older than `ttl` reclaims the claim with a new token and logs the takeover.
- The displaced holder's later `attest done` is rejected via token mismatch ("your claim was reclaimed at T, you no longer own this item").
- `attest reset FR-001` always force-releases, regardless of TTL.

**Advisory only.** The lock is enforced at CLI write boundaries. Attest cannot physically prevent a misbehaving holder from continuing to write source files after losing the lease. Sandboxing is reeve's job. Well-behaved agents follow step 4.5 of the agent loop contract to re-check claim ownership during long sessions.

**Deferred for v0.1+1:** heartbeat/renewal of leases; pid/host liveness checks.

## Verification — three passes

Attest separates verification into layers by cost.

### Pass 1: Scan (cheap, deterministic)

`attest scan` does case-insensitive substring match for declared test names across files matching `scan.test_file_patterns`. Per item, reports:

| Field      | Meaning                                                                |
| ---------- | ---------------------------------------------------------------------- |
| `found`    | Declared name uniquely attributable to this item                       |
| `missing`  | Declared name not matched anywhere                                     |
| `orphan`   | Test in code that no item declares                                     |
| `ambiguous`| Declared name matched in files belonging to (or hinted by) >1 item     |

When a declared name matches in files hinted by multiple items, `test_hints` are used as a **tie-breaker** to attribute the match to one item. `test_hints` are not a hard scope — a name that lives outside an item's hints still counts as `found` if uniquely attributable. When no tie-breaker resolves a collision, the name is reported `ambiguous` for both items rather than silently attributed.

Cost: milliseconds, zero LLM tokens. Suitable for every agent loop iteration.

### Pass 2: Verify (run tests)

`attest verify` invokes your test runner via configured command, captures CLI output, and matches test names against declarations to determine pass/fail per declared test. Output capture is framework-agnostic — Attest doesn't parse JSON reporters or AST.

Cost: seconds, zero LLM tokens. Suitable for pre-commit, CI, milestone gates.

### Pass 3: Review (semantic)

`attest review FR-001` packages a structured payload and emits it to stdout. The payload is consumed by an external reviewer (typically an agent invoked via chief or council). The reviewer judges whether tests actually verify the spec — not just match names — and writes the verdict via `attest mark-reviewed`.

**Payload contents** (same in both formats):

| Field                    | Description                                                                 |
| ------------------------ | --------------------------------------------------------------------------- |
| `item`                   | `id`, `flow`, `status`, `verified_against`, spec/test hashes                |
| `spec_md`                | `spec.md` body (embedded)                                                   |
| `test_md`                | `test.md` body (embedded)                                                   |
| `flow_overview_md`       | flow `overview.md` body (embedded, if applicable)                           |
| `scan_result`            | Pass 1 output                                                               |
| `verification_result`    | Pass 2 output                                                               |
| `depends_on`             | List of `{id, status}` for each dependency                                  |
| `implementation_hints`   | Paths + current commit SHA. **File contents NOT embedded** — reviewer fetches as needed |

`--format json` (default — matches agent-as-consumer principle). `--format markdown` returns a human-readable bundle suitable for human review or council artifacts.

Code is referenced by path + commit, not embedded. Reviewer (or its host) fetches excerpts as needed. Keeps the payload bounded and consistent with the framework-agnostic / no-AST principle.

Cost: LLM tokens. On-demand only — milestone reviews, audits, contentious items.

### `mark-reviewed` — writing the verdict

```bash
attest mark-reviewed FR-001 --verdict pass --against-spec sha256:abc
attest mark-reviewed FR-001 --verdict fail --against-spec sha256:abc
```

`--against-spec` must match the current `spec.md` hash. A mismatch is rejected as **"stale payload — re-run review"** (the spec changed between the reviewer's input and the verdict write).

On `pass`:
- Item in `done` or `failing` (with Pass 1+2 passing) transitions to `verified`.
- Item in `stale` due to spec drift re-stamps `verified_against.spec` to the current hash and transitions to `verified`.

On `fail`, the item stays in its current state; the verdict is recorded (git is the audit log).

## Drift detection

Three drift types, detected differently.

| Drift type | Trigger                         | Detection                                      | Severity                          |
| ---------- | ------------------------------- | ---------------------------------------------- | --------------------------------- |
| Spec drift | `spec.md` body changed          | Hash comparison with `verified_against.spec`   | Definite                          |
| Test drift | `test.md` declarations changed  | Parse current list vs verified list            | Definite                          |
| Code drift | Code changed after verify       | Compare HEAD with `verified_against.commit`    | Potential — re-verify to confirm  |

**Recovery rules:**

| Cause       | Auto-recover via `attest verify`?                                                                                              | Recovery paths                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| Spec drift  | **No** — `verify` cannot confirm tests cover the new spec; re-stamping the hash would produce a false `verified` (integrity hole) | `attest mark-reviewed --verdict pass --against-spec <hash>` (Pass 3) **or** `attest accept-drift FR-001 --spec` |
| Test drift  | Yes (`stale.auto_recover`)                                                                                                     | `attest verify`                                                                                                  |
| Code drift  | Yes (`stale.auto_recover`)                                                                                                     | `attest verify`                                                                                                  |

`attest accept-drift FR-001 --spec` is the **shallow** path: re-stamps the spec hash without running a Pass 3 review. Intended for cases where a human or agent has verified by inspection that existing tests still cover the new spec and a full review isn't needed.

When `implementation_hints` are present, code drift narrows via `git diff` — `attest affected` lists items whose hinted files changed. TIA-lite — selective, not bulletproof, but cheap.

Hash sensitivity: body content with normalized whitespace; frontmatter excluded (CLI manages it).

## CLI surface

```bash
# discovery
attest list                          # all items in current scope
attest list --status pending
attest list --ready                  # pending items with deps met; excludes stale
attest list --format json            # for agents to parse
attest show FR-001
attest show FR-001 --format json
attest stats
attest coverage                      # ratio + item-level traceability matrix
attest coverage --format json

# state transitions
attest start FR-001
attest done FR-001
attest block FR-001 --reason "..."
attest skip FR-001 --reason "..."
attest reset FR-001                  # force-releases .claim regardless of TTL

# verification — three passes
attest scan FR-001                   # Pass 1
attest scan                          # all items
attest verify FR-001                 # Pass 2
attest verify --affected             # only items with recent diff in hints
attest review FR-001                 # Pass 3 — package payload (default --format json)
attest review FR-001 --format markdown
attest mark-reviewed FR-001 --verdict pass --against-spec <hash>
attest mark-reviewed FR-001 --verdict fail --against-spec <hash>

# drift
attest stale
attest diff FR-001
attest affected --since=HEAD~5
attest accept-drift FR-001 --spec    # shallow consent path (no Pass 3)

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
    "case_sensitive": false
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
  "stale": {
    "block_ready": true,
    "auto_recover": true
  },
  "lock": {
    "ttl": "15m"
  },
  "dependencies": {
    "block_state": "verified"
  }
}
```

- `verify.gates` — preconditions for the `done` transition (despite the namespace name; left as-is for v0.1 compatibility, naming is a known oddity).
- `stale.block_ready` — exclude stale items from `attest list --ready`.
- `stale.auto_recover` — `attest verify` auto-transitions stale → `verified | failing`. Applies to **code/test drift only**; spec drift is never auto-recovered (see Drift detection).
- `lock.ttl` — lease duration for the `.claim` file. A claim older than `ttl` may be reclaimed by another `attest start`.
- `dependencies.block_state` — whether dependencies must be `verified` (strict) or `done` (lenient) before dependent items can start.

## Coverage

`attest coverage` reports both a summary ratio and an item-level traceability matrix.

```
Coverage: 8/10 items verified (80%)

FR-001    verified    tests: 3/3 found, 3 passing
FR-002    done        tests: 2/2 found, not verified
FR-003    pending     tests: 0/0 declared
FR-004    failing     tests: 4/4 found, 1 failing
FR-005    stale       tests: 3/3 found, last verified at sha:abc
...
```

`--format json` emits the matrix as structured rows for agent consumption. RTM-style means **item-level** — criteria inside `spec.md` body are not parsed in v0.1.

## Agent loop contract

```
1. attest list --status pending --ready --format json
   → agent receives list, picks one based on its own logic

2. attest show FR-005
   → agent reads spec, test declarations, flow context

3. attest start FR-005
   → state: pending → in_progress
   → rejected if deps unmet (return to step 1)
   → rejected if claim conflict (another agent holds active lease — return to step 1)

4. agent implements code + writes tests in codebase

4.5. if elapsed since `start` approaches `lock.ttl`:
     attest show FR-005 --format json
     → if your claim token no longer matches, your lease was reclaimed.
       Stop writing files. Return to step 1.

5. attest done FR-005
   → state: in_progress → done (gated by verify.gates)
   → rejected if claim token mismatch (your lease was reclaimed) — return to step 1.

6. (optional, cheap) attest scan FR-005
   → confirms declared tests exist in code

7. (CI or milestone) attest verify FR-005
   → runs tests, transitions to verified or failing

8. goto 1
```

Agents have full autonomy in choosing work order. The CLI guards transitions, scan checks structural presence, verify checks behavioral correctness, and review (when invoked) checks semantic alignment.

## Composition with chief-tribe

**With chief** — Attest scopes map to chief milestones. Items can be generated by a chief skill that breaks milestone goals into atomic items. The chief agent loop uses Attest commands as its work-tracking layer.

**With council** — items flagged for review become artifacts that humans approve before merge. Attest's frontmatter and scan/verify results provide council with evidence. `attest review --format markdown` is the natural input to a council artifact.

**With reeve** — `verify.command` and `review.command` can run inside reeve's allowlist for safe execution. Reeve also enforces the advisory `.claim` lock at the file-write layer.

None are required. Attest ships standalone.

## What Attest does not do

* Does not generate items from a spec. That's a proposer agent's job; Attest consumes its output.
* Does not run agents or schedule them. (Note: parallel-work coordination IS in scope, via `.claim` — see Multi-agent.)
* Does not store implementation code or test code.
* Does not judge whether tests are good — only whether declared tests exist (Pass 1) and pass (Pass 2). Quality judgment is Pass 3, delegated.
* Does not provide a web UI or dashboard. CLI + json export only.
* Does not detect code drift definitively — uses commit hash as proxy.
* Does not sandbox file writes — the `.claim` lock is advisory. Reeve handles enforcement.

## Tech stack & deliverable

TypeScript on Bun, distributed via npm. Frontmatter parsing via gray-matter, schema validation via zod, state machine as hand-rolled transition table. Single binary via `bun build`.

Day-1 deliverable:

* `attest` CLI binary with all commands listed in **CLI surface**
* `attest init` scaffolding (with `--with-flows` variant)
* Reference `attest.config.json` (including `stale`, `lock` sections)
* `.claim` lock file mechanism with TTL recovery
* All three verification passes (scan, verify, review + mark-reviewed)
* Drift detection (spec/test/code) with spec-drift carve-out
* One worked example (5–10 items, spec + test + implementation)
* README with the agent loop contract

---

## Open Questions

Resolved by grill `0001-attest-spec-v1` and folded into the body above: spec-v1 Q2 (multi-agent), Q4 (test name resolution), Q6 (hash sensitivity), Q10 (review payload format), Q12 (spec drift acceptance), Q18 (reviewer trust boundary). Remaining open items below retain their spec-v1 numbering for traceability.

### Q1 — Item ID format

Strict (`^[A-Z]+-\d{3,}$`) or free-form? Strict helps tools, free-form helps adoption. **Lean: configurable via `item.id_pattern`, default to strict.**

### Q3 — Body schema enforcement

Should certain sections in spec.md (`## Acceptance`, `## Out of scope`) be required and parsed? **Lean: optional convention in v0.1, no enforcement.**

### Q5 — Test framework support strategy

Output capture is universal, but exit codes and test name extraction vary. **Lean: capture raw stdout/stderr, run grep-based name detection, document edge cases per framework.**

### Q7 — Flow ↔ item sync direction

If `items: [...]` in flow overview and `flow: F-01` in item frontmatter both exist, which wins? **Lean: item is the source of truth; CLI auto-updates flow's items list on mutation.**

### Q8 — Cross-scope dependencies

Can item in milestone-2 depend on item in milestone-1? **Lean: no in v0.1, scope-local only.**

### Q9 — Item rename / scope move

Moving FR-001 from `milestone-1` to `milestone-2` — manual directory move, or CLI command that updates hashes and refs? **Open.**

### Q11 — Flow without items

A flow exists with no items yet — draft of intent or misconfiguration? **Lean: allow it; warn but don't error.**

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

### Q19 — Dependency cycles

If user creates a cycle (FR-001 deps on FR-002, FR-002 deps on FR-001), how does CLI behave? **Lean: detect on load, refuse to start any item in cycle, output cycle path.**

### Q20 — Optional dependencies on flows

Can an item depend on an entire flow being complete (`depends_on: [F-01]`)? Useful but conflates entity types. **Lean: no in v0.1, items only.**

### Q21 (new) — `verify.gates` naming

Config namespace `verify.gates` actually gates the `done` transition, not `verify`. **Lean: leave as-is in v0.1; revisit when adding more gates. Renaming would be a breaking config change.**

### Q22 (new) — Lease heartbeat / renewal

`lock.ttl` is a fixed lease; a legit task longer than `ttl` risks premature reclaim. **Lean: not in v0.1; default `ttl` (15m) is generous for atomic-item work. Heartbeat/renewal is a v0.1+1 candidate.**
