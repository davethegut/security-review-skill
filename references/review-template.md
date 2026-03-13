# Review Report Template

Use this template for all security review output. Copy the structure and fill in findings. Adapt
sections based on the review mode (Skill Review, Deployment Plan, Post-Deployment Audit).

Write the report to `work_docs/reviews/security-review-{name}-{YYYY-MM-DD}.md`.

---

```markdown
# Security Review: {artifact-name}

**Reviewed**: {YYYY-MM-DD}
**Artifact**: {path to skill, plan, or config}
**Mode**: Skill Review | Deployment Plan Review | Post-Deployment Audit
**Reviewer**: skill-security-review (first pass) + independent agent (second pass)

## Risk Summary

| Metric | Value |
|--------|-------|
| Overall Risk | CRITICAL / HIGH / MEDIUM / LOW |
| Write Operations Identified | N |
| Credential Exposure Points | N |
| Critical Findings | N |
| High Findings | N |
| Medium Findings | N |
| Low Findings | N |

## First-Pass Findings

### [CRITICAL] {finding title}

- **Checklist ref**: {e.g., 1.1, 6.1}
- **Location**: {file path}:{section or line reference}
- **Description**: {what the issue is}
- **Risk**: {what could go wrong and the impact}
- **Remediation**: {specific, actionable fix}

### [HIGH] {finding title}

- **Checklist ref**: {e.g., 2.3}
- **Location**: {file path}:{section or line reference}
- **Description**: {what the issue is}
- **Risk**: {what could go wrong and the impact}
- **Remediation**: {specific, actionable fix}

### [MEDIUM] {finding title}

...

### [LOW] {finding title}

...

## Failure Mode Assessment

Evaluate against the common failure modes. For each that applies:

| # | Failure Mode | Applies? | Assessment |
|---|-------------|----------|------------|
| 1 | Alert fatigue amplification | Yes/No/Partial | {brief assessment} |
| 2 | Automation without circuit breakers | Yes/No/Partial | {brief assessment} |
| 3 | Implicit trust in LLM output | Yes/No/Partial | {brief assessment} |
| 4 | Security theater | Yes/No/Partial | {brief assessment} |
| 5 | Credential sprawl | Yes/No/Partial | {brief assessment} |
| 6 | Log poisoning / prompt injection | Yes/No/Partial | {brief assessment} |
| 7 | Orphaned state | Yes/No/Partial | {brief assessment} |
| 8 | Blast radius ignorance | Yes/No/Partial | {brief assessment} |
| 9 | Drift between docs and implementation | Yes/No/Partial | {brief assessment} |
| 10 | Insufficient audit trail | Yes/No/Partial | {brief assessment} |

## Second-Pass Review (Independent Agent)

> This section is populated by an independent agent with no prior context. It validates the
> first-pass findings and checks for gaps.

### Coverage validation

- [ ] All 7 checklist categories were evaluated
- [ ] All 10 failure modes were assessed
- [ ] Write operations were correctly identified
- [ ] Credential exposure points were correctly identified

### Additional findings not caught in first pass

{List any findings the first pass missed, or state "None identified."}

### Remediation validation

{For each first-pass remediation, assess whether it is sufficient, incomplete, or could introduce
new issues.}

| Finding | Remediation Adequate? | Notes |
|---------|----------------------|-------|
| {title} | Yes / Partial / No | {explanation} |

### Disagreements with first pass

{List any findings where the second-pass reviewer disagrees with the severity, assessment, or
remediation. State "None." if fully aligned.}

## Consolidated Recommendations

Priority-ordered list of actions. Merge first-pass and second-pass findings.

1. **[CRITICAL]** {action item}
2. **[HIGH]** {action item}
3. **[MEDIUM]** {action item}
4. **[LOW]** {action item}
```
