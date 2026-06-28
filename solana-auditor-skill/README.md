# Solana Security Auditor Skill

> A production-grade AI skill for auditing Solana/Anchor smart contracts — built for the [Superteam Solana AI Kit](https://superteam.fun) Skills bounty.

---

## Overview

The **Solana Security Auditor Skill** transforms any compatible AI agent into a senior Solana smart contract security auditor. It provides structured, token-efficient knowledge bases that guide the agent through:

- Identifying critical vulnerabilities in Anchor programs
- Generating professional, structured audit reports
- Applying production-grade remediation patterns

This skill is designed to work inside the [Solana AI Kit](https://github.com/solana-foundation/solana-ai-kit) skill routing system. It follows the `SKILL.md` specification for lazy, context-aware loading so the agent only reads what it needs for each task.

---

## Features

- **16 Vulnerability Classes** — Covers every major Solana/Anchor exploit vector including missing signer checks, PDA validation, CPI reentrancy, integer overflow, type cosplay, and Token-2022 security issues.
- **Anchor Constraints Reference** — Complete reference for all 18 `#[account(...)]` constraints (`mut`, `signer`, `init`, `seeds`, `bump`, `has_one`, `close`, `realloc`, `zero`, and more) with security notes and examples.
- **Severity Taxonomy** — Every finding is tagged Critical / High / Medium / Low / Informational with a rationale.
- **Production Code Examples** — Each vulnerability includes a vulnerable snippet and a secure, production-grade fix in Rust/Anchor.
- **Detection Tips** — Pattern-match guidance so the AI can locate vulnerabilities by scanning account structs and instruction handlers.
- **Audit Report Generator** — A structured template for producing professional security audit reports with executive summaries, findings tables, and risk scoring.
- **Realistic Example Report** — A fully worked example audit report for a vulnerable Anchor vault program.
- **Token-Efficient Routing** — `SKILL.md` routes the agent to only the relevant reference file, avoiding unnecessary context consumption.

---

## Folder Structure

```
solana-auditor-skill/
├── README.md                          # This file
├── LICENSE                            # MIT License
└── skill/
    ├── SKILL.md                       # Skill entrypoint & routing logic
    ├── vulnerability-checklist.md     # 16-class vulnerability knowledge base
    ├── anchor-constraints-ref.md      # Complete Anchor constraint reference
    ├── report-generator.md            # Audit report template & instructions
    └── example-audit-report.md        # Realistic worked example audit report
```

---

## Installation

This skill is consumed by an AI agent runtime that supports the Solana AI Kit skill format. No build step is required — all content is plain Markdown.

### Option A — Clone directly into your agent's skill directory

```bash
git clone https://github.com/Dhruv26052005/solana-auditor-skill.git
```

Then register the skill by pointing your agent runtime at `skill/SKILL.md`.

### Option B — Reference via the Solana AI Kit registry

If the Solana AI Kit supports a skills registry, add this repository URL to your agent configuration. Consult the [Solana AI Kit documentation](https://github.com/solana-foundation/solana-ai-kit) for the exact configuration format.

---

## Usage

Once loaded, the skill activates on prompts related to Solana/Anchor security. The agent will:

1. Read `skill/SKILL.md` to understand its role and routing logic.
2. Load the appropriate sub-reference (`vulnerability-checklist.md` or `report-generator.md`) based on the user's intent.
3. Apply the loaded knowledge to audit the supplied program code or generate a formatted report.

### Trigger Phrases

The skill is designed to activate on prompts such as:

- `"Audit this Anchor program for vulnerabilities"`
- `"Find security issues in this Solana smart contract"`
- `"Generate a security report for this program"`
- `"Check this instruction handler for missing signer checks"`
- `"Is this PDA validation correct?"`

---

## Example Prompt

```
You are a senior Solana security auditor. Audit the following Anchor program
and generate a professional security report with all findings, severity levels,
and recommended fixes.

[paste Anchor program code here]
```

---

## Example Output

```
# Security Audit Report — Vault Program v1.0

**Auditor:** AI Security Agent (Solana Auditor Skill)
**Date:** 2025-01-15
**Severity Summary:** 2 Critical · 1 High · 2 Medium · 1 Low

---

## Executive Summary

The Vault program exhibits two critical account validation failures that
allow an attacker to drain funds without authorization. Immediate remediation
is required before mainnet deployment.

---

## Findings

| ID  | Title                        | Severity | Status |
|-----|------------------------------|----------|--------|
| F-01 | Missing signer check on withdraw | Critical | Open |
| F-02 | No owner validation on vault account | Critical | Open |
| F-03 | Unchecked integer arithmetic | High     | Open |
...

[Full structured report — see skill/example-audit-report.md]
```

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Contributing

Pull requests are welcome. Please ensure any new vulnerability entries follow the established format in `skill/vulnerability-checklist.md` and include:

- A clear description and danger explanation
- Both a vulnerable and a secure Rust/Anchor code example
- Detection tips
- A severity rating with rationale
