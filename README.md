# Adversarial Plan Review

A reusable Claude Code skill for adversarial review between two models in a loop:

`review -> validate -> fix -> re-review -> repeat until zero findings`

It is designed for implementation plans and fresh code changes where correctness, runtime behavior, integration boundaries, performance, and review cost all matter.

## What It Adds

- Multi-round review instead of a single pass
- Mandatory fix verification
- Evidence-gated findings
- Risk-based effort selection
- Scoped file reads before widening review
- Stagnation guard for deadlocks
- Performance and rate-cost checks

## How It Works

```text
Claude writes a plan or code
    |
    v
Codex reviews
    |
    v
Claude validates each finding
    |
    v
Disputes go back for rebuttal
    |
    v
Fixes are applied
    |
    v
Codex re-reviews the fixes
    |
    v
If new findings exist, loop again
    |
    v
Stop only at "LGTM - zero findings."
```

## Installation

1. Copy `plan-review.md` into your project's `.claude/commands/` directory:

```bash
mkdir -p .claude/commands
cp plan-review.md .claude/commands/
```

2. Run it in Claude Code:

```text
/plan-review
/plan-review path/to/plan.md
```

## Usage

### Pre-Implementation

Use it to stress test a plan before writing code.

### Post-Implementation

Use it to review the actual code after changes are made.

### Risk Scaling

The protocol scales review effort by risk:

- LOW -> medium effort
- MEDIUM -> high effort
- HIGH -> xhigh effort

It starts narrow, reading only the files and lines most relevant to the claim, then widens only if the evidence requires it.

## Why This Pattern Is Useful

Single-pass review often misses issues that show up only after a fix or only when execution paths, boundaries, or runtime behavior are checked carefully. This protocol makes the re-review step mandatory and keeps the review efficient by scaling depth to risk.

## License

MIT
