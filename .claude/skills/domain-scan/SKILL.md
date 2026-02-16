---
name: domain-scan
description: Scan code to classify logic into four domain knowledge layers - Physical (invariants), Operational (tradeoffs), Strategic (preferences), Historical (legacy). Use when analyzing unfamiliar codebases, doing inner-source work, or understanding why code exists.
argument-hint: [file-or-directory-path]
allowed-tools: Read, Grep, Glob, Bash, Task, LSP
---

# Domain Knowledge Layer Scanner

You are analyzing code to classify every meaningful piece of logic into one of four domain knowledge layers. This helps engineers working in inner-source understand what AI can safely change vs what requires human judgment.

## Target

Scan: `$ARGUMENTS`

If no argument is provided, ask the user which file or directory to scan.

## The Four Layers

### Layer 1: Physical / Mathematical (AI can own)

Logic that derives from math, formal rules, data integrity, or hard regulatory requirements. These are **non-negotiable invariants**.

Detection signals in code:
- Assertions, validations, guards that enforce correctness (e.g., `assert`, `validate`, `require`)
- Transaction boundaries, rollback logic
- Checksum / hash verification
- State machine transitions with exhaustive checks
- Double-entry / balance checks (sum must equal zero)
- Idempotency enforcement (unique constraints, dedup checks)
- Type constraints, schema validation
- Rounding rules, precision handling
- Referential integrity checks
- Cryptographic operations

### Layer 2: Operational / Tradeoff (AI can measure, humans decide tolerance)

Logic shaped by incidents, performance constraints, cost, scalability, or reliability concerns. These are **optimization choices under constraints** -- not mathematical truths.

Detection signals in code:
- Thresholds, limits, rate limits, circuit breakers
- Timeout values, retry counts, backoff strategies
- Batch sizes, queue depths, pool sizes
- Fallback / degradation paths
- Cache TTLs, expiry windows
- Monitoring/alerting metric names and thresholds
- SLA/SLO enforcement values
- Async vs sync choices (especially with comments explaining why)
- Feature flags guarding rollout percentages

### Layer 3: Strategic / Political (Humans must own)

Decisions shaped by business strategy, OKRs, priorities, power structures, or market conditions. This is where "truth" becomes **contextual preference**.

Detection signals in code:
- Priority ordering in scoring/ranking logic
- A/B test configurations and variant weights
- Pricing logic, discount rules, tier definitions
- Routing rules that choose between strategies
- Approval/rejection thresholds for business decisions
- KPI/metric definitions (what counts as "success")
- Notification/escalation rules (who gets alerted for what)
- Partner-specific or customer-segment-specific branching
- Conversion/fraud/cost tradeoff parameters
- Comments referencing OKRs, quarterly goals, business requirements

### Layer 4: Historical / Path-Dependent (Humans must label source)

Constraints that exist because of legacy systems, backward compatibility, migration cost, or external integration lock-in. These are **inherited, not optimal**.

Detection signals in code:
- Hard-coded IDs, magic numbers, special-case partner/customer lists
- `TODO`, `HACK`, `WORKAROUND`, `LEGACY`, `DEPRECATED` comments
- Compatibility shims, version-specific branches
- Dead code paths kept "just in case"
- Schema compromises (redundant fields, denormalization with comments)
- Format conversions for external systems
- Wrapper functions that exist only to adapt old interfaces
- Config that exists because "another system requires it"
- Conditional logic based on dates (migration cutovers)
- Multiple implementations of the same thing (old + new paths)

## Scan Process

### Step 1: Map the module structure

Use Glob and Read to understand the file/directory layout. Identify:
- Entry points
- Core business logic files
- Configuration files
- Test files (these reveal invariants)
- External integration points

### Step 2: Identify and classify each significant logic block

For each file, scan for the detection signals above. Classify each meaningful logic block.

When uncertain between layers, flag it as **"ambiguous"** -- do NOT guess.

### Step 3: Produce the report

Output a structured report in this exact format:

---

## Domain Knowledge Scan: [module/file name]

### Summary

| Layer | Count | Confidence |
|-------|-------|------------|
| Physical/Mathematical | N | high/medium |
| Operational/Tradeoff | N | high/medium |
| Strategic/Political | N | high/medium |
| Historical/Path-Dependent | N | high/medium |
| Ambiguous | N | -- |

### Layer 1: Physical / Mathematical

For each finding:
- **Location**: `file:line`
- **What**: Brief description of the invariant
- **Evidence**: The code pattern that indicates this layer
- **AI Actionable**: Yes -- AI can safely test, refactor, optimize this

### Layer 2: Operational / Tradeoff

For each finding:
- **Location**: `file:line`
- **What**: Brief description of the tradeoff/threshold
- **Current Value**: The threshold or config value
- **Evidence**: Why this is operational (metric name, timeout, limit, etc.)
- **Question for domain expert**: One specific question to understand the tolerance (e.g., "What incident drove this timeout to 5000ms?")

### Layer 3: Strategic / Political

For each finding:
- **Location**: `file:line`
- **What**: Brief description of the business decision
- **Evidence**: Why this is strategic (scoring, routing, pricing, priority logic)
- **Unknown**: What AI cannot determine (e.g., "Why is partner X prioritized?")
- **Decision owner**: Who likely needs to approve changes here (inferred from code ownership/comments)

### Layer 4: Historical / Path-Dependent

For each finding:
- **Location**: `file:line`
- **What**: Brief description of the legacy constraint
- **Evidence**: Why this appears historical (TODO, shim, magic number, dual path)
- **Risk if changed**: What might break if this is modified
- **Question for domain expert**: "Is this still needed? What depends on it?"

### Ambiguous

For each finding:
- **Location**: `file:line`
- **What**: What the code does
- **Possible layers**: Which layers it could belong to and why
- **Evidence needed**: What information would resolve the ambiguity

### Recommendations

1. **Safe for AI**: List specific areas where AI can confidently generate code, tests, or refactors
2. **Needs human input before changing**: List specific decisions that require human confirmation
3. **Suggested questions for domain expert**: A prioritized list of 5-10 questions that would unlock the most understanding (not generic -- specific to the code scanned)

---

## Important Rules

- **Never fabricate domain reasoning.** If you don't know why something exists, say "Unknown" and suggest how to find out.
- **Prefer "Ambiguous" over wrong classification.** Being uncertain is more valuable than being confidently wrong.
- **Be specific.** Reference exact file:line locations. Quote code snippets when they illustrate the classification.
- **Keep questions actionable.** Every question for domain experts should be answerable in 1-2 sentences, not require an essay.
- **Look at tests.** Test files often reveal which invariants the team considers critical.
- **Look at git blame / comments.** If available, comments and commit context help distinguish layers.
