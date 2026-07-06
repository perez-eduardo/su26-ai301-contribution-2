# Contribution #2: Add SendGrid API Key Detection Pattern

**Contribution Number:** 2
**Student:** Eduardo Perez
**Issue:** https://github.com/Raftersecurity/rafter-cli/issues/24
**Status:** Phase I Complete

---

## Why I Chose This Issue

Rafter is a secret scanning CLI, so this issue sits directly in the security tooling space I want more depth in. Issue #24 is labeled `good first issue` and `secret-pattern`, and the maintainer spelled out the exact SendGrid key format and the files involved, which makes it well scoped for a focused contribution rather than an open ended investigation.

What drew me to it specifically is that Rafter ships parallel Node/TypeScript and Python implementations that the project requires to stay identical. A single small pattern add still means working carefully across two languages, two pattern definitions, and two test suites, and keeping them in exact parity. After my first contribution, where I added word timestamp support to a Python voice framework, I wanted a second issue that kept me in a real production codebase while stretching me on cross language parity and on writing secret detection regex correctly. This issue fits that goal.

---

## Understanding the Issue

### Problem Description

Rafter scans code and files for leaked secrets by matching a library of provider specific patterns. It currently has no pattern for SendGrid API keys. As a result, a leaked SendGrid key sitting in a repository or a scanned file passes through the tool undetected. Issue #24 asks for a new detection pattern that recognizes the SendGrid key format and flags it the same way the tool already flags other third party API keys.

### Expected Behavior

When Rafter scans content that contains a SendGrid API key, meaning the `SG.` prefixed format, it should report a match and identify it as a SendGrid API key at `critical` severity, consistent with how every other provider API key in the tool is classified.

### Current Behavior

No SendGrid pattern exists in either the Node scanner or the Python scanner. A SendGrid key is therefore never matched, and a scan reports nothing for it, leaving a real credential type unmonitored.

### Affected Components

The change spans both implementations, which the project requires to remain identical, plus the changelog:

- `node/src/scanners/secret-patterns.ts`: the `DEFAULT_SECRET_PATTERNS` array where Node patterns are defined. Each entry follows the `Pattern` interface in `node/src/core/pattern-engine.ts`, which has the fields name, regex as a string, severity, and optional description.
- `python/rafter_cli/scanners/secret_patterns.py`: the Python pattern list, described in the file itself as a verbatim port of the Node patterns, using a `Pattern` dataclass with the same four fields.
- `node/tests/secret-patterns.test.ts`: the Vitest suite for the Node patterns.
- `python/tests/test_regex_scanner.py`: the pytest suite for the Python patterns.
- `CHANGELOG.md`: a changelog entry, following the precedent set by the recent DigitalOcean pattern PR.

The target pattern is `SG\.[a-zA-Z0-9_-]{22}\.[a-zA-Z0-9_-]{43}` at `critical` severity. The severity is not stated in the issue. Every provider specific API key already in the scanner (AWS, GitHub, Google, Stripe, Twilio, HashiCorp, npm, PyPI, DigitalOcean, and the rest) is classified `critical`, and a SendGrid key has an equally distinctive exact format, so it belongs in that group. The only patterns the tool rates below critical are the fuzzy, higher false positive ones such as Generic API Key and Bearer Token.

---

## Reproduction Process

To be completed in Phase II, where I set up the dev environment in both implementations and demonstrate that a SendGrid key currently goes undetected by a scan.

---

## Solution Approach

To be completed in Phase II, covering the root cause and a full UMPIRE implementation plan for adding the pattern and its tests to both languages.

---

## Testing Strategy

To be planned in Phase II and executed in Phase III, using one positive and one negative test in each suite, with fake tokens built programmatically so nothing trips push protection.

---

## Implementation Notes

To be completed in Phase III as the pattern, tests, and changelog entry are written and committed.

---

## Pull Request

To be completed in Phase IV, including the PR link, the Summary, Test plan, and Spec sections the project requires, the parity checklist, the AI assistance disclosure with a Co-Authored-By trailer, and any maintainer feedback and how I addressed it.

---

## Learnings & Reflections

To be completed in Phase IV, once the contribution is submitted and reviewed.

---

## Resources Used

- Issue #24, which defines the SendGrid format, regex, and the files to change: https://github.com/Raftersecurity/rafter-cli/issues/24
- The repository `CONTRIBUTING.md`, for the contribution process, the hard parity rule, the PR body and checklist requirements, and the AI assistance disclosure rule.
- `CLAUDE.md` and `AGENTS.md`, for the four step recipe for adding a new secret pattern.
- `docs/adding-a-platform.md`, which states the parity rule in its strongest form, that both languages ship in the same PR.
- The recent DigitalOcean pattern PR (commit `582ee46`), which serves as a near exact template for a new provider key pattern, including its changelog entry.
