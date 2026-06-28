# Audit Report Generator

This reference teaches the AI agent how to produce professional, structured security audit reports for Solana and Anchor smart contract programs. Follow every section precisely — do not omit sections, do not use placeholder text.

---

## Report Structure Overview

A production-quality audit report must contain the following sections in order:

1. Report Header (metadata block)
2. Table of Contents
3. Executive Summary
4. Scope
5. Findings Summary Table
6. Detailed Findings (one sub-section per finding)
7. Security Metrics
8. Positive Observations
9. Final Recommendation

---

## Section 1 — Report Header

Every report opens with a metadata block. Use a Markdown table or bold key-value pairs:

```markdown
# Security Audit Report — [Program Name] v[Version]

**Report Classification:** Confidential
**Prepared For:** [Client Name]
**Prepared By:** Solana Security Auditor Skill
**Audit Date:** [Start Date] → [End Date]
**Report Version:** 1.0 — Final
```

---

## Section 2 — Table of Contents

Include a linked table of contents referencing all major sections and each individual finding by its ID.

---

## Section 3 — Executive Summary

The executive summary is written for founders, investors, and non-technical stakeholders. It must be:

- **Concise** — no more than 3–4 short paragraphs.
- **Decisive** — state the overall security posture clearly.
- **Actionable** — end with a clear deployment recommendation.

### Required Elements

| Element | Description |
|---|---|
| Program description | One sentence on what the program does |
| Overall assessment | Plain-English security posture |
| Security score | Integer 0–100 (see scoring below) |
| Total findings | Count of all findings |
| Severity distribution | Critical / High / Medium / Low / Info counts |
| Deployment recommendation | One of: ✅ Ready · ⚠️ Conditional · ⛔ Do Not Deploy |

### Security Score Rubric

| Score Range | Interpretation |
|---|---|
| 90–100 | Excellent — minor informational notes only |
| 75–89 | Good — Low/Informational findings only; safe to deploy with monitoring |
| 60–74 | Acceptable — Medium findings present; remediate before mainnet |
| 40–59 | Poor — High findings present; significant rework required |
| 0–39 | Critical — Critical findings present; do not deploy |

**Deductions:**

- −20 per Critical finding
- −10 per High finding
- −5 per Medium finding
- −2 per Low finding
- −0 per Informational finding (noted but not penalized)

Start from 100 and subtract accordingly.

---

## Section 4 — Scope

List exactly what was audited. Be precise — vague scopes undermine the report's credibility.

### Required Fields

| Field | Example |
|---|---|
| Program Name | SolVault Protocol |
| Program ID | `VLT7xqB3...` (devnet) |
| Framework | Anchor v0.29.0 |
| Language | Rust 1.75 |
| Network | Devnet / Mainnet-beta |
| Audit Date Range | 2025-01-13 → 2025-01-15 |
| Auditor | Solana Security Auditor Skill |
| Commit Hash | `a3f8c21d...` |
| Files Reviewed | List each file path |
| Out of Scope | Explicitly list anything NOT covered |

---

## Section 5 — Findings Summary Table

Produce a compact table listing all findings. Use consistent IDs (F-01, F-02, …).

```markdown
| ID   | Severity     | Title                                      | Status |
|------|--------------|--------------------------------------------|--------|
| F-01 | 🔴 Critical  | Missing Signer Validation on Withdraw      | Open   |
| F-02 | 🟠 High      | Integer Overflow in Reward Calculation     | Open   |
| F-03 | 🟡 Medium    | Unvalidated Remaining Accounts             | Open   |
| F-04 | 🟢 Low       | Missing Event Emission on State Change     | Open   |
```

**Status values:** `Open` | `Acknowledged` | `Resolved` | `Wont Fix`

---

## Section 6 — Detailed Findings

Each finding is a sub-section under `## F-XX — [Title]`. Follow this exact template for every finding:

```markdown
## F-XX · [Title]

### Severity
[Severity emoji + label]

### Location
[File path] — [struct/function name], line [N]–[M]

### Description
[2–4 sentences explaining the vulnerability class, where it appears,
and what is missing or incorrect. No speculation — only describe
what the code does and what it fails to do.]

### Vulnerable Code Pattern
[Fenced Rust code block illustrating the vulnerability.
Add ❌ comments to mark the dangerous lines.]

### Impact
[Describe the attack step-by-step as a numbered list.
Be specific: who can exploit it, what preconditions are needed,
what is the worst-case outcome (e.g., "full vault drain with no privileges").]

### Recommendation
[Describe the correct mitigation in plain English before showing code.
Explain WHY the fix works, not just what to change.]

### Secure Code Pattern
[Fenced Rust code block showing the corrected implementation.
Add ✅ comments to mark the key safe lines.]

### References
- [Link or name of relevant Anchor/Solana documentation]
- [Link to sealevel-attacks example if applicable]
```

---

## Section 7 — Security Metrics

Produce a metrics table summarizing the finding counts by severity, plus the final score.

```markdown
| Severity         | Count | % of Total | Description                          |
|------------------|-------|------------|--------------------------------------|
| 🔴 Critical      | X     | X%         | Direct, unconditional fund-loss      |
| 🟠 High          | X     | X%         | Exploitable under realistic conditions|
| 🟡 Medium        | X     | X%         | Moderate impact or specific conditions|
| 🟢 Low           | X     | X%         | Minor issues                          |
| ℹ️ Informational | X     | X%         | Best-practice notes                  |
| **Total**        | **N** | **100%**   |                                       |
```

Then include a short **Scoring Rationale** paragraph explaining what drove the score up or down.

---

## Section 8 — Positive Observations

This section prevents the report from reading as purely adversarial. It must:

- Acknowledge at least 3 security practices that were done correctly.
- Be specific — cite the file, struct, or pattern that was correct.
- Be honest — do not invent positives that do not exist.

**Example format:**

```markdown
1. **Descriptive Custom Errors** — `errors.rs` defines X specific error variants,
   making security assertions unambiguous and debuggable.

2. **Correct Rent-Exempt Initialization** — All `init` constraints include explicit
   `space` calculations, ensuring no account is created below the rent-exempt threshold.
```

---

## Section 9 — Final Recommendation

This is the closing section — written for the client's leadership. It must:

- Restate the overall risk level in one sentence.
- List the top 2–3 actions the team must take immediately.
- Provide an estimated remediation effort table (finding ID, effort, priority).
- Recommend next steps (e.g., re-review, fuzz testing, formal verification).
- End with a clear statement on deployment readiness.

---

## Severity Level Definitions

Use these definitions consistently throughout all reports:

| Severity | Label | Definition |
|---|---|---|
| 🔴 | **Critical** | An attacker can directly steal funds, mint tokens, or permanently corrupt state with no prerequisites. Immediate action required. |
| 🟠 | **High** | An attacker can cause significant financial or protocol damage under realistic conditions (e.g., specific timing, moderate capital). Must fix before mainnet. |
| 🟡 | **Medium** | The vulnerability requires specific conditions or produces limited financial impact. Fix before mainnet; may be deferred in testnet. |
| 🟢 | **Low** | Minor issue with negligible impact. Code quality or defensive improvement. |
| ℹ️ | **Informational** | Best-practice recommendation with no direct security impact. |

---

## Finding ID Naming Convention

- Prefix all finding IDs with `F-` followed by a zero-padded two-digit number: `F-01`, `F-02`, …
- Order findings from most severe to least severe within each severity level.
- Never reuse an ID within the same report.

---

## Language and Tone Guidelines

- Write in third-person technical prose (not "you" or "we").
- Be precise and specific — name the exact file, line, struct, and variable.
- Never speculate about intent — describe only what the code does and what it fails to do.
- Do not soften Critical findings with hedging language like "may potentially possibly allow."
- Do use measured language for Low and Informational findings to avoid alarming the client unnecessarily.

---

## Checklist Before Finalizing a Report

Before delivering any report, verify the following:

- [ ] All finding IDs are sequential and unique.
- [ ] Every finding has a severity, location, description, impact, recommendation, and both code blocks.
- [ ] The security score matches the severity counts using the deduction formula.
- [ ] The deployment recommendation matches the score range.
- [ ] No placeholder text ("TODO", "Lorem Ipsum", "Example") appears anywhere.
- [ ] All Rust code blocks are syntactically plausible (no pseudocode unless labeled).
- [ ] References link to real, publicly accessible documentation.
- [ ] Positive observations mention at least 3 specific correct practices.
- [ ] The final recommendation includes a remediation effort table.
