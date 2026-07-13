# Contribution #2: Add SendGrid API Key Detection Pattern

**Contribution Number:** 2
**Student:** Eduardo Perez
**Issue:** https://github.com/Raftersecurity/rafter-cli/issues/24
**Status:** Phase III Complete

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

### Environment Setup

I set up a Linux native development environment inside WSL (Ubuntu on Windows 11), which matches the Linux CI the maintainers gate on. The fork is cloned at `~/CodePath/AI301/rafter-cli` inside the WSL filesystem, not across the Windows mount, so file permissions and symlinks behave natively. The toolchain is Node 24 via nvm, pnpm via Corepack, Poetry via uv, and Python 3.12 in an isolated uv virtual environment.

Several setup challenges were worth noting and were all solved without changing any tracked file. I first attempted the setup natively on Windows and hit 187 test failures that were all environmental: symlink creation needing elevation, POSIX file mode assertions, hardcoded `/tmp` paths, and path separator mismatches. Moving to WSL cleared every one of them. The pnpm 11 build approval for esbuild is required to run the Node suite, and it can only live in the tracked `pnpm-workspace.yaml`, so it is deliberately kept out of every commit rather than staged. On the Python side, Poetry 2.x refused to install because the repo's `pyproject.toml` uses the older `[tool.poetry]` layout, which Poetry reads as a lock mismatch. Rather than regenerate the tracked `poetry.lock`, I created an isolated uv virtual environment, installed the dependency set, pinned Click below 8.2 and Typer to the 0.15 line to match the repo's ranges, and installed the package itself editable with `--no-deps`. That produced a fully green suite touching no tracked files.

Baseline before any change: the Node suite passes 1890 tests with zero failures, and the Python suite passes cleanly on the files relevant to this work. The working tree is clean apart from the intentional pnpm build approval.

### Steps to Reproduce

Because SendGrid detection is a missing feature rather than a broken behavior, reproduction means demonstrating that a correctly shaped SendGrid key currently goes undetected, and confirming the scanners otherwise work by detecting a pattern they already know.

1. Create a file containing a correctly shaped but obviously fake SendGrid key, built as `SG.` followed by 22 characters, a dot, then 43 characters, all repeated `A` so the value is clearly synthetic.
2. Scan the file with the Node CLI: `node node/dist/index.js secrets <path> --json`.
3. Scan the same file with the Python CLI: `python -m rafter_cli secrets <path> --json`.
4. Observe that both scans return an empty results list.
5. As a control, create a file with a fake DigitalOcean key (`dop_v1_` plus 64 hex characters), a pattern that already exists, and scan it with both CLIs.
6. Observe that both scans detect the DigitalOcean key, confirming the scanners are live and reading files correctly.

### Reproduction Evidence

The fake SendGrid key scanned to an empty result in both implementations. Both the Node and Python CLI returned a local scan payload with `"results": []`, meaning no SendGrid match and no match of any kind.

The control confirmed the scanners work. The fake DigitalOcean key was detected in both implementations, each reporting a pattern named `DigitalOcean Personal Access Token` at `critical` severity.

Taken together, the empty SendGrid results are proof of a missing pattern rather than a broken scan. The gap the issue describes is real and identical in Node and Python.

---

## Solution Approach

### Analysis

The root cause is simply that no SendGrid pattern is defined. The scanner architecture, the file reading, and the matching engine all work, as the DigitalOcean control proves. The fix is therefore additive: introduce one new provider pattern in each implementation, with matching tests, and record it in the changelog. No engine or spec change is required, and CLI behavior does not change.

### Proposed Solution

Add a SendGrid detection pattern with the regex `SG\.[a-zA-Z0-9_-]{22}\.[a-zA-Z0-9_-]{43}` at `critical` severity to both the Node and Python scanners, using identical regex, then add a positive and a negative test in each suite and a changelog entry. The recently merged DigitalOcean pattern (commit `582ee46`) serves as a near exact template for every part of this.

### Implementation Plan

Using the UMPIRE framework:

**Understand:** Rafter's scanner has no SendGrid pattern, so a SendGrid key passes undetected. Add the pattern to both implementations with identical regex at `critical` severity, plus tests in each suite and a changelog entry. Reproduction confirmed the gap is real and that the scanners otherwise work.

**Match:** The DigitalOcean Personal Access Token pattern is the template. It touches the same files the same way: a pattern object in the Node `DEFAULT_SECRET_PATTERNS` array, a mirrored `Pattern` dataclass entry in Python, positive and negative tests in each suite using programmatically built fake tokens, and a CHANGELOG line. Severity `critical` follows the precedent that every provider specific API key in the scanner is critical. Test conventions to mirror are `scanString()` in Node's `secret-patterns.test.ts` and the `scanner` fixture in Python's `test_regex_scanner.py`.

**Plan:**
1. Add the SendGrid pattern object to `node/src/scanners/secret-patterns.ts` with a service name comment, `name: "SendGrid API Key"`, the regex, `severity: "critical"`, and a description.
2. Add the identical pattern to `python/rafter_cli/scanners/secret_patterns.py` as a `Pattern` dataclass instance, keeping the regex byte for byte identical to the Node version.
3. Add a positive and a negative test to `node/tests/secret-patterns.test.ts`, building the fake key at runtime.
4. Add the mirrored positive and negative tests to `python/tests/test_regex_scanner.py`.
5. Add a `CHANGELOG.md` entry following the DigitalOcean precedent.

**Implement:** Work on a dedicated branch off `main`. Stage only the five intended files explicitly so the pnpm build approval and the Python virtual environment never ride along. Commit with a conventional commit message such as `feat(scan): add SendGrid API key secret pattern`.

**Review:** Check against the contribution checklist: identical regex in both implementations, both suites passing, versions in `node/package.json` and `python/pyproject.toml` untouched, no spec change needed, and no real secrets anywhere since test keys are built programmatically. Confirm the tree holds only the five intended files before pushing.

**Evaluate:** Re-run the reproduction. The same fake SendGrid key that returned zero results must now be detected as `SendGrid API Key` at `critical` severity in both CLIs, and the DigitalOcean control must still pass. Run the full Node and Python suites and confirm the two touched test files are green with no regressions.

---

## Testing Strategy

### Unit Tests

- [x] Node positive: a runtime built fake SendGrid key is detected as `SendGrid API Key` in `node/tests/secret-patterns.test.ts`.
- [x] Node negative: a too-short SendGrid-like value produces no SendGrid match.
- [x] Python positive: the mirrored fake key is detected in `python/tests/test_regex_scanner.py`.
- [x] Python negative: the mirrored too-short value produces no SendGrid match.

### Integration Tests

- [x] CLI reproduction flip: after implementation, `rafter secrets` on the fake key file reports a SendGrid match in both Node and Python, where it previously reported none.
- [x] Control unchanged: the DigitalOcean key is still detected in both implementations.

### Manual Testing

Ran the exact reproduction and control scans from Phase II against the built code. The fake SendGrid key now flags as `SendGrid API Key` at `critical` in both the Node and Python CLI, the flip from the earlier empty result. The DigitalOcean control still detects. Both full suites were run: Node passes 1892 tests with zero failures, and Python passes on all files relevant to this change. The only Python reds in the full run are 19 errors confined to `test_mcp_server_stdio.py`, which pass when that file is run on its own, so they are cross-suite subprocess contention in the local run rather than a defect, and they are unrelated to the secret scanner.

---

## Implementation Notes

### Week 3 Progress

I implemented the full change in both languages, added the tests, and committed it on a dedicated branch. The pattern went into both scanner files right after the DigitalOcean entry, comment headed the same way, using the regex `SG\.[a-zA-Z0-9_-]{22}\.[a-zA-Z0-9_-]{43}` at `critical` severity. The Node string form uses a doubled backslash and the Python raw string uses a single backslash, which compile to the same pattern, so parity holds byte for byte in behavior. I added a positive and a negative test to each suite, mirroring the DigitalOcean and GitHub test shapes, with tokens built programmatically so nothing resembles a real secret. Then I added the CHANGELOG entry in the DigitalOcean format.

The main challenge this phase was an environment split. Claude Code was installed on the Windows side, so its edits landed in the Windows clone at the `/mnt/e` path rather than the WSL clone where my toolchain and green baseline live. I caught this when a scan of the WSL clone still showed no SendGrid match and a grep confirmed the pattern was absent there. The clean fix was to reapply the edits directly in the WSL clone. To avoid multi-line paste corruption in the terminal, I applied each edit through a small reviewable Python script that anchors on the existing DigitalOcean entry and inserts the new block, with assertions that fail safely rather than double insert.

I also confirmed the project's branch and commit conventions from git history rather than assuming. Human contribution branches use a conventional commit style namespace, and the DigitalOcean pattern itself was committed as `feat(scan): add DigitalOcean Personal Access Token secret pattern`. So I branched as `feat/sendgrid-secret-pattern` and used the matching commit subject.

Per the project's contribution rules, the commit includes the AI assistance disclosure: a note that it was written with Claude Code, plus a `Co-Authored-By` trailer.

### Code Changes

- **Files modified:** `node/src/scanners/secret-patterns.ts`, `python/rafter_cli/scanners/secret_patterns.py`, `node/tests/secret-patterns.test.ts`, `python/tests/test_regex_scanner.py`, `CHANGELOG.md`.
- **Key commit:** `7818af9` on branch `feat/sendgrid-secret-pattern`, `feat(scan): add SendGrid API key secret pattern (#24)`, five files changed, 40 insertions.
- **Approach decisions:** Followed the DigitalOcean pattern as a near exact template for placement, naming, severity, and tests. Kept the pnpm build approval out of every commit by staging only the five intended files by name. Kept the Python environment isolated in a uv venv so no tracked lock or config file was ever rewritten.

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
