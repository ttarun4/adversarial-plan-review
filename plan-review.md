You are running the Adversarial Plan Review Protocol - a reusable adversarial review process between Claude and Codex.

Target to review: $ARGUMENTS (default: the active plan file from plan mode, or the most recently implemented code if post-implementation)

## Purpose

Use this protocol to review implementation plans or recent code in systems where correctness, runtime behavior, integration boundaries, and operational cost matter.

Review every plan or implementation through these five lenses:
- correctness and safety
- integration and boundary integrity
- config and domain integrity
- production wiring honesty
- runtime performance and cost discipline

## Core Contract

- The review loops until Codex returns `LGTM - zero findings.` or the protocol hits an explicit escalation condition.
- Every finding must be evidence-gated. A file path and line number are required, plus one of:
  - call-site grep or wiring trace
  - focused test evidence
  - schema or query-path evidence
  - final output or integration-payload evidence
- Never call something "wired" unless you traced the active runtime path and instance.
- Review cost must match risk. Default Codex effort by risk:
  - HIGH -> `xhigh`
  - MEDIUM -> `high`
  - LOW -> `medium`
- Escalate effort by one level only if the first pass is ambiguous, disputed, or under-evidenced.
- Keep the review scoped. Send the minimum relevant files and line ranges first; widen only when the evidence demands it.
- Stagnation guard: if two consecutive rounds produce no new CONFIRMED findings, or the same dispute repeats twice, escalate to the user with the exact blocker.

## Phase 0 - Scope and Risk Map

1. State whether this is:
   - plan review
   - post-implementation review
2. Read only the target plan, diff, and the minimum project context needed.
3. Identify touched surfaces such as:
   - latency-sensitive runtime paths
   - final output or integration paths
   - background jobs or workers
   - DB writes and schema changes
   - auth, tenant, or entity-boundary logic
   - config-backed thresholds or domain-specific rules
4. Classify risk:
   - HIGH: safety-critical execution, final integration payloads, DB write semantics, auth or tenant boundaries, latency-sensitive paths, unbounded external calls
   - MEDIUM: orchestration hints, learning logic, background jobs, API contract changes, queue or worker health, moderate query growth
   - LOW: docs, isolated tooling, read-only analytics
5. If risk is HIGH, Claude must write 3-5 independent pre-findings before reading Codex output to avoid anchoring.

## Paths You Must Map

Before reviewing, identify the project's equivalents of:
- the primary runtime path
- the final output or integration path
- any background or deferred execution path
- any auth, tenant, or entity-boundary path

If the plan claims behavior across more than one path, verify parity across those paths. Do not assume one path implies the others.

## Review Lenses

### 1. Correctness and Safety

- Could this introduce a correctness bug, invalid state, or unsafe behavior?
- Does the final output reflect the intended safe behavior?
- Are fallbacks honest?

### 2. Integration and Boundary Integrity

- Could data or commands cross the wrong tenant, account, org, zone, or entity boundary?
- Do auth and authorization boundaries match the plan?
- Do DB changes preserve behavior across supported backends or dialects?
- Are API contracts and downstream integrations still coherent?

### 3. Config and Domain Integrity

- Are thresholds and operating values config-backed instead of hardcoded?
- Are domain-specific claims evidence-honest instead of overstated?
- Is the proposal operationally sane for a real production environment, not just code-clean?

### 4. Production Wiring Honesty

For every claimed safeguard, feature, or agent output, classify it as exactly one of:

- `EXISTS_ONLY`
- `CODE_WIRED`
- `LIVE_PATH_WIRED`
- `FINAL_OUTPUT_VERIFIED`
- `RUNTIME_VALIDATION_PENDING`

Prefer the lowest honest classification. "Implemented" is not enough.

### 5. Runtime Performance and Cost Discipline

- Does this add blocking work to a latency-sensitive path?
- Could it increase loop time, queue lag, stale-state risk, or time-to-effect?
- Does it introduce N+1 queries, repeated scans, repeated model loads, or repeated remote calls?
- Should the work live in a background worker, cached service, or lower-frequency cadence instead?
- If the feature uses LLMs, external APIs, or heavy compute, is there:
  - a rate budget
  - timeout and fallback behavior
  - bounded call frequency
  - a degraded-mode path when limits are hit
- Does the design scale without linear cost explosion as tenants, entities, cycles, or events grow?

## Phase A - Adversarial Review

### Step 1: Codex Review

1. Send the plan or changed code to Codex via `codex:codex-rescue`.
2. Set Codex effort from risk:
   - HIGH -> `--effort xhigh`
   - MEDIUM -> `--effort high`
   - LOW -> `--effort medium`
3. Instruct Codex to focus on:
   - live runtime path
   - divergence across claimed execution paths
   - final output or integration-payload parity
   - auth, tenant, or entity-boundary correctness
   - supported DB backend or dialect behavior
   - hardcoded thresholds or defaults vs config
   - latency-sensitive path impact and queue lag risk
   - repeated query, model, API, or LLM call cost
   - whether cadence and placement are appropriate for the work
   - tests that hit the changed path
4. Require structured findings:
   - severity: `P0/P1/P2/P3`
   - title
   - file:line
   - why it matters
   - evidence type
   - wiring status classification
   - performance or rate-cost note if relevant
5. Limit Codex to the minimum relevant files. For post-implementation reviews, list only the changed files that actually matter.
6. Wait for Codex to complete before proceeding.

### Step 2: Claude Validation

7. For each Codex finding:
   - read the cited file or line
   - mark it `CONFIRMED`, `PARTIAL`, or `DISPUTED`
   - cite counter-evidence when disputing
   - do not rubber-stamp
8. For wiring, safeguard, or boundary claims, run the four-grep check:
   - imported and called in the live path?
   - same instance or same state source?
   - enabled by default?
   - silent failure path or bare `except`?
9. Add NEW Claude findings if Codex missed them.

### Step 3: Resolve Disputes

10. Send only `DISPUTED` or `PARTIAL` findings back to Codex for rebuttal.
11. Use the same effort tier as the initial pass unless the dispute is still ambiguous, then escalate one level.
12. If the same dispute repeats twice, escalate to the user immediately.

## Phase B - Fix or Plan Correction

13. Fix the code or correct the plan for all confirmed or agreed findings.
14. Run the minimum targeted verification needed:
   - focused tests
   - grep or call-site traces
   - payload or path inspection
   - schema or query verification
15. Do not mark a finding resolved based only on reasoning.

## Phase C - Mandatory Verification Loop

16. Send the updated code or plan back to Codex.
17. Tell Codex which findings were fixed and how.
18. Ask Codex to verify the fixes and scan for regressions introduced by the fixes.
19. Use the same effort tier as the initial pass; escalate one level only if the fix itself affects safety, tenant boundaries, or hot-path performance.
20. End the prompt with:
   `If zero issues remain, respond exactly: LGTM - zero findings.`
21. If Codex returns new findings, go back to Step 7.
22. If Codex returns `LGTM - zero findings.`, proceed to Summary.
23. If the stagnation guard triggers, stop and escalate with the blocker.

## Summary Format

Present to the user:

```
## Plan Review Summary

### Review Stats
- Risk level: [HIGH/MEDIUM/LOW]
- Rounds: [N]
- Confirmed findings: [N]
- Disputed and closed: [N]
- Escalated: [N]

### Runtime Paths Reviewed
- [path 1]
- [path 2]

### Confirmed Findings
- [P1] [title] - [one-line issue]
  Evidence: [proof]
  Status: [EXISTS_ONLY/CODE_WIRED/LIVE_PATH_WIRED/FINAL_OUTPUT_VERIFIED/RUNTIME_VALIDATION_PENDING]

### Boundary Risks
- [tenant/auth/entity/API risks]

### Performance / Cost Risks
- [latency, queueing, query count, API/LLM rate, or scaling concerns]

### Residual Risks / Runtime Validation
- [what still needs real runtime validation]

### Final Verdict
[APPROVE / REQUEST CHANGES / NEEDS DECISION]
[Codex verdict]
```

## Rules

1. Never claim "live" or "wired" without tracing the live path.
2. Never approve integration-affecting work without checking the final output path.
3. Keep tenant and entity identifiers explicit. Avoid silent fallback behavior.
4. If DB writes change, verify behavior across supported backends or dialects.
5. Prefer config-backed thresholds over hardcoded operating values.
6. Do not present hypotheses as evidence-locked domain truth.
7. Report what you found so far. Do not claim completeness.
8. If Codex times out, retry with a narrower file set and more specific line ranges.
9. If Codex returns empty output twice, proceed with Claude-only review and note the gap.
10. Do not add slow, remote, or LLM-backed work to a latency-sensitive path without bounded fallbacks and an explicit rate budget.
11. Prefer targeted review passes over full-file resend loops; only widen scope when the current evidence is insufficient.
