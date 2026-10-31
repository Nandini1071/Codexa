# Phase 1: Planning and requirement finalisation

**Goal:** define exactly what we are building before writing code.
**Deliverable:** finalised requirements, user flows, MVP scope, and test scenarios.
**Status:** drafted. Decisions in section 10 still need confirming.

**Contents:** 1. Scope · 2. Workflow · 3. Extension UI · 4. File-change handling · 5. Risk signals · 6. Findings · 7. Data model · 8. Test scenarios · 9. Success metrics · 10. Decisions · 11. Exit checklist · Appendix: finding JSON schema

---

## 1. MVP scope

The MVP is one vertical slice that works end to end.

**In scope**
- VS Code extension for TypeScript/JavaScript repositories only
- Change detection on file save and Git diff (working tree vs HEAD)
- Changed-symbol extraction (functions, classes, exports)
- Basic impact analysis: direct callers, importers, related tests
- Sidebar with Changes, Impact and Findings views
- **Investigate Change**: sends a scoped evidence bundle to FastAPI; an LLM returns structured findings
- Every finding has a source reference, evidence and a suggested verification step
- Investigations persisted, plus a minimal web dashboard to list and open them
- Feedback on findings: useful / irrelevant / incorrect / verified

**Basic or small version in the MVP**
- Approved test execution: propose a test, ask permission, run, show output
- RAG over historical failures (small initial version)

**MVP+ (after the slice works)**
- GitHub App, PR webhooks, CI status
- Teams, roles, shared policies
- Advanced release simulation
- Baseline comparison (base vs changed branch)
- More language analyzers

> The original blueprint marked GitHub App, teams and release simulation as MVP. They are moved to MVP+ to keep the first release achievable (decision D1).

**Out of scope**
- Autonomous commits or merges
- Unrestricted shell access
- Multi-agent swarms
- A risk model trained on real outcomes (only if data supports it later)
- Uploading the whole source tree

---

## 2. User workflow

1. Developer opens a TS repo; Codexa activates.
2. Developer edits and saves a file.
3. Codexa debounces (~1.5s), diffs against HEAD, extracts changed symbols, updates **Changes**.
4. Codexa builds the local impact graph (callers, importers, tests) and updates **Impact**.
5. Local rules run and show **static findings** immediately (no AI call).
6. Developer clicks **Investigate Change**. A preview of the evidence bundle is shown; developer confirms.
7. Backend retrieves context, runs the bounded investigator, returns structured findings.
8. **Findings** shows results by severity with evidence; clicking jumps to the code.
9. Developer clicks **Propose Verification**; Codexa shows the test and exact command; developer approves or declines.
10. Test runs; output is shown. The finding becomes verified, unresolved or inconclusive.
11. Developer leaves feedback. The investigation is saved and visible in the web dashboard.

```
save -> debounce -> diff -> symbols -> impact graph -> local rules -> views
                                                         |
                                  [Investigate Change] -> bundle preview -> consent
                                                         |
                          backend: retrieve -> investigate -> validate -> persist
                                                         |
                          findings -> [Propose Verification] -> consent -> run
                                                         |
                                       status + feedback -> web dashboard
```

**Alternate and failure flows**

| Situation | Behaviour |
|---|---|
| Cloud analysis disabled | Local findings only; Investigate explains how to enable it |
| Backend unreachable / LLM error | Local findings keep working; clear offline/error state; retry offered |
| Diff too large | Bundle truncated by priority; warning shows what was dropped |
| Secret detected in diff | Redacted in preview; never uploaded |
| User declines test run | Finding stays `unresolved`; nothing runs |
| Test passes | Status `inconclusive`: "Passed. This does not prove the change is safe." |
| Non-TS/JS repo | Unsupported-language notice |

---

## 3. What the extension displays

**Activity bar sidebar**
- **Changes:** changed files, then symbols, with +/- line counts
- **Impact:** each changed symbol with its callers, importers, tests, config files
- **Findings:** grouped by severity; each row has a type badge (Verified / Static / AI hypothesis) and a status

**Other surfaces**
- **Finding detail panel:** all fields from section 6, clickable evidence, feedback buttons
- **Status bar:** `Codexa: 3 changes | 2 findings | idle` (states: idle, analyzing, offline, error)
- **Inline diagnostics:** only for high and critical findings, to avoid warning noise
- **Verification output:** panel showing the approved command and its result

**Commands:** Investigate Change · Re-analyze · Propose Verification · Open Report in Web · Clear Findings

**Settings**

| Setting | Default | Purpose |
|---|---|---|
| `codexa.backendUrl` | `http://localhost:8000` | Backend base URL |
| `codexa.cloudAnalysis.enabled` | `false` | Allow sending evidence bundles |
| `codexa.debounceMs` | `1500` | Save debounce |
| `codexa.ignoreGlobs` | node_modules, dist, build, generated | Files to skip |
| `codexa.maxBundleKb` | `256` | Evidence bundle size cap |

---

## 4. What happens when a developer changes a file

```
save -> debounce -> git diff -> parse changed files (TS compiler API)
     -> map hunks to symbols -> build impact graph (1-2 hops)
     -> map symbols to tests -> run local rules -> update views
```

- **Trigger:** file save and Git state changes, debounced. No analysis per keystroke.
- **Incremental:** only changed files are re-parsed; results cached by content hash.
- **Diff base:** working tree vs HEAD (staged and unstaged). Branch comparison is MVP+.
- **Symbol mapping:** hunks map to the enclosing function/class/export via the TypeScript compiler API.
- **Impact graph:** direct callers and importers (1 hop), 2 hops for exported symbols. Tests are linked by imports and references, with filename convention as a fallback.
- **Ignored by default:** `node_modules`, build output, generated files. Lockfiles are read only as a dependency signal.
- **Local vs cloud:** local analysis and rules run on every save. Cloud investigation runs only on **Investigate Change**.
- **Latency target:** under 2s for a typical save.

---

## 5. How Codexa identifies risky changes

Risk comes from **documented signals** with stable IDs. There is no opaque AI score. Signals produce static findings directly and are the starting evidence for the AI investigator.

| ID | Trigger | Default severity |
|---|---|---|
| `SIG-API-SIG` | Exported function/method signature changed and it has callers | high |
| `SIG-NO-TEST` | Changed symbol not referenced by any test | medium |
| `SIG-TEST-DEL` | Test deleted, skipped, or assertions removed | high |
| `SIG-ERR-REMOVED` | try/catch, error check, or guard clause removed | high |
| `SIG-NULL-GUARD` | null/undefined check removed or optional chaining dropped | medium |
| `SIG-COND-FLIP` | Condition, comparison operator or boundary changed | medium |
| `SIG-ASYNC` | async/await added or removed; possible unhandled promise | high |
| `SIG-CONFIG` | Env var, config file or feature flag changed | medium |
| `SIG-DEP` | Dependency added or version bumped (major = higher) | medium |
| `SIG-SCHEMA` | DB schema, migration, or API contract/type changed | high |
| `SIG-FANOUT` | Large blast radius (many callers/importers) | raises other signals |
| `SIG-SIZE` | Large diff relative to accompanying test changes | low |

**Rules**
- Formatting-only and comment-only changes trigger nothing.
- Default severities are starting points; fan-out and missing tests can raise them.
- The AI may add hypotheses beyond these signals, but must label them `ai_hypothesis` and cite evidence.
- A summary indicator (Low / Elevated / High) is derived from signal counts and severities by a published rule. It is not a probability of failure. A numeric `risk_score` waits until calibrated against labelled examples.

---

## 6. Findings, evidence and recommendations

| Field | Notes |
|---|---|
| `type` | `verified_failure` / `static` / `ai_hypothesis` |
| `severity` | critical / high / medium / low / info |
| `confidence` | high / medium / low |
| `location` | repo-relative `path`, `symbol`, `line_start`, `line_end` |
| `why_it_matters` | plain-language reason |
| `evidence[]` | kind (diff, caller, test, config, history, test_output), path, lines, note |
| `uncertainty` | what we do not know |
| `recommendation` | action, optional suggested test (description, file, command) |
| `status` | open / verified / unresolved / inconclusive / dismissed |
| `signals[]` | signal IDs that triggered it |

**Rules**
1. A finding with no evidence is rejected by schema validation.
2. Only a recorded test run can produce `verified_failure`. An `ai_hypothesis` is never shown as verified.
3. A passing test never proves safety: status becomes `inconclusive` with an explicit message.
4. Paths are repo-relative; no absolute paths or secrets.

**Example**

> **HIGH | Static | `SIG-API-SIG`**: `calculateDiscount(cart, code)` now returns `number | null`
> **Why it matters:** 4 callers assume a number.
> **Evidence:** diff `src/pricing.ts:42-58`; caller `src/checkout.ts:91`; no test covers the null path.
> **Uncertainty:** callers in `src/admin/` may guard null elsewhere.
> **Recommended:** add a test for an invalid code; handle null in `checkout.ts`.
> **Suggested test:** `pricing.test.ts`: "returns null for invalid code and checkout handles it"

The full JSON Schema is in the appendix.

---

## 7. What information is stored

| Entity | Stores |
|---|---|
| `users` | Account and auth reference |
| `repositories` | Name, owner, connection settings |
| `investigations` | Repo, branch, commit, diff hash, analysis version, status, timestamps |
| `findings` | Fields from section 6 |
| `verification_runs` | Approved command, truncated output summary, exit code, approval time, status |
| `feedback` | Finding id, user, label (useful / irrelevant / incorrect / verified), note |
| `evaluation_cases` | Labelled scenario with expected signals/findings (section 8) |
| `audit_events` | Consent, test runs, investigation state changes |

**Privacy rules**
- Store repo-relative paths, symbol names, commit ids and line ranges.
- No absolute paths, secrets or full source files. Keep only short redacted snippets when needed as evidence.
- Secret redaction runs on the developer's machine before anything is uploaded.
- The developer sees and confirms the evidence bundle before upload.

**Evidence bundle (extension to backend):** repo id, commit, diff hunks, changed symbols, impact graph summary (names and paths), related test file names, triggered signals, and small snippets around changes. Never the whole source tree. Capped by `codexa.maxBundleKb`.

**Deferred:** a `risk_score` column, until it has an explicit definition and calibration.

---

## 8. Test scenarios

A sample repo (working name `codexa-sample-shop`, TypeScript, passing tests) gets seeded changes. Each scenario becomes an `evaluation_cases` entry.

| # | Seeded change | Expected result |
|---|---|---|
| 1 | Return type changed to `T \| null` on a function with 4 callers | `SIG-API-SIG`, high, callers cited |
| 2 | Function changed; no test references it | `SIG-NO-TEST`, medium, suggests a test |
| 3 | `if (qty > 0)` changed to `>= 0` | `SIG-COND-FLIP`, medium/high |
| 4 | try/catch removed around a payment call | `SIG-ERR-REMOVED`, high |
| 5 | Test deleted alongside a code change | `SIG-TEST-DEL`, high |
| 6 | Missing `await` on a DB write | `SIG-ASYNC`, high |
| 7 | Env var renamed in code but not in `.env.example`/config | `SIG-CONFIG`, medium |
| 8 | Dependency major-version bump | `SIG-DEP`, medium |
| 9 | Comment/formatting-only change | **No findings** (control) |
| 10 | Safe refactor with tests updated | Low or no risk (control) |
| 11 | Bug that genuinely fails a test | After approved run: `verified_failure` |
| 12 | Backend unreachable / LLM error | Local findings still work; clear offline state |
| 13 | Very large diff | Bundle truncated gracefully with warning |
| 14 | Secret in the diff | Redacted in preview; never uploaded |

Scenarios 9 and 10 are the false-positive controls: a tool that flags everything is useless.

---

## 9. MVP success metrics

- Local analysis updates in under 2s for a typical save
- At least 80% of seeded scenarios detected with the correct signal
- 0 high-severity findings on the control scenarios (9, 10)
- 100% of findings carry at least one evidence item with a valid repo-relative path

---

## 10. Decisions

Status stays **Proposed** until the project owner confirms. Update this table when decided.

| # | Decision | Proposed default | Status |
|---|---|---|---|
| D1 | GitHub App, teams, release simulation move to MVP+ | Yes | Proposed |
| D2 | LLM provider and model for the backend | TBD | Open |
| D3 | First test runner(s) | Vitest and Jest | Proposed |
| D4 | Auth in MVP | Dev API key only; real login with the web dashboard | Proposed |
| D5 | Sample repo | Write our own so bugs are seeded and known | Proposed |
| D6 | Repo layout | Monorepo: `codexa/{docs,vscode-extension,backend,web}` | Decided |

---

## 11. Exit checklist

- [x] MVP scope drafted
- [x] Workflow drafted
- [x] Extension UI described
- [x] Signal catalogue drafted
- [x] Finding schema written
- [x] Data model drafted
- [x] Test scenarios drafted
- [ ] Decisions D1–D5 confirmed
- [ ] Extension UI wireframes (sketches)
- [ ] Sample repo created (Phase 2)

---

## Appendix: finding JSON schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://codexa.dev/schemas/finding.schema.json",
  "title": "Finding",
  "type": "object",
  "required": ["id", "type", "severity", "confidence", "title", "summary", "location", "why_it_matters", "evidence", "uncertainty", "recommendation", "status", "signals"],
  "properties": {
    "id": { "type": "string" },
    "investigation_id": { "type": "string" },
    "type": { "enum": ["verified_failure", "static", "ai_hypothesis"] },
    "severity": { "enum": ["critical", "high", "medium", "low", "info"] },
    "confidence": { "enum": ["high", "medium", "low"] },
    "title": { "type": "string" },
    "summary": { "type": "string" },
    "location": {
      "type": "object",
      "required": ["path"],
      "properties": {
        "path": { "type": "string", "description": "Repo-relative path", "not": { "pattern": "^(/|[A-Za-z]:\\\\)" } },
        "symbol": { "type": "string" },
        "line_start": { "type": "integer", "minimum": 1 },
        "line_end": { "type": "integer", "minimum": 1 }
      }
    },
    "why_it_matters": { "type": "string" },
    "evidence": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["kind", "path", "note"],
        "properties": {
          "kind": { "enum": ["diff", "caller", "test", "config", "history", "test_output"] },
          "path": { "type": "string", "not": { "pattern": "^(/|[A-Za-z]:\\\\)" } },
          "lines": { "type": "string", "description": "e.g. 42-58" },
          "note": { "type": "string" }
        }
      }
    },
    "uncertainty": { "type": "string" },
    "recommendation": {
      "type": "object",
      "required": ["action"],
      "properties": {
        "action": { "type": "string" },
        "suggested_test": {
          "type": "object",
          "required": ["description"],
          "properties": {
            "description": { "type": "string" },
            "file": { "type": "string" },
            "command": { "type": "string" }
          }
        }
      }
    },
    "status": { "enum": ["open", "verified", "unresolved", "inconclusive", "dismissed"] },
    "signals": { "type": "array", "items": { "type": "string", "pattern": "^SIG-[A-Z-]+$" } }
  }
}
```
