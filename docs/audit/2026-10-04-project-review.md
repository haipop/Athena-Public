# Project Review Audit - 2026-10-04

## Metadata

| Field | Value |
|---|---|
| Audited repository | `haipop/Athena-Public` fork of `winstonkoh87/Athena-Public` |
| Audit base | commit `120942eb2c32e562f80c422c8dd96fc3b1ad5ca8` |
| Audit branch | `audit/project-review` |
| Review scope | Repository structure, packaging, initialization/boot behavior, test evidence, retrieval architecture, governance hooks, public documentation accuracy, release state |
| Out of scope | Production Supabase deployment, private memory corpus evaluation, live IDE integration for every supported target, dependency vulnerability adjudication |
| Change type | Documentation-only audit; no runtime source changes |

## Executive Summary

Project Athena is a Python 3.10+ CLI/SDK that provides a portable context and memory layer for AI coding agents. The public repository combines:

- Markdown workspace scaffolding through `athena init`.
- Session lifecycle management through `/start`, `/save`, and `/end`.
- Hybrid retrieval using canonical text, vectors, BM25, filenames, framework documents, and optional web search, followed by weighted RRF fusion and reranking.
- MCP tools with permission, governance, and secret-mode behavior.
- Claude Code hooks for secret access, destructive-command, meta-awareness, and output checks.
- Optional Supabase pgvector storage.

The project has unusually strong hygiene for a solo personal-knowledge system: 637 tests were collected, 630 passed and 7 skipped on Python 3.12; Ruff passed; internal links and privacy scans passed; and upstream GitHub workflows were green at the audited commit.

The principal concern is that the public repository behaves like a filtered slice of a larger private workspace rather than a fully self-contained product. This is most visible in the broken standalone `init` to boot path: initialization creates `.framework/modules/Core_Identity.md`, while boot requires `.framework/v8.2-stable/modules/Core_Identity.md` and a fixed SHA-384 hash. Documentation and inventory files also report private-workspace counts that are not present in this checkout.

Overall assessment: strong reference implementation and experimental framework, but not yet a turnkey, independently installable product. Treat code-enforced governance claims as scoped primarily to Claude Code, and treat retrieval benchmark claims as dependent on an unavailable private corpus.

## Verification Performed

All local checks were run at the audit base in a temporary Python 3.12 environment.

| Check | Result | Interpretation |
|---|---|---|
| `pytest tests -q` | Exit 0; 630 passed, 7 skipped; 637 collected | Core test suite is green |
| Coverage | 45.32% | Moderate coverage; above the current 15% gate |
| `ruff check src scripts tests` | Exit 0 | Lint clean for source, scripts, and tests |
| `athena --version` | `athena-cli v10.0.2` | CLI entry point installed and executable |
| Standalone `athena init ... --ide claude` | Exit 0 | Workspace scaffolding succeeds |
| Standalone `athena check` | Exit 0 | Basic health check succeeds |
| Standalone default `athena` boot | Failure: `Core_Identity.md NOT FOUND` | Critical init/boot contract mismatch |
| `athena doctor` in generated workspace | 24 passed, 7 warnings, 2 skipped | Doctor does not catch the boot-blocking identity mismatch |
| Internal link checker | 707 links checked, no failures | Public Markdown links are internally valid |
| Privacy scan | 1,024 files clean with committed blocklist | No committed blocklist violations |
| `mypy src --ignore-missing-imports --no-error-summary` | Exit 1 with multiple errors | Type errors exist; CI currently marks mypy non-blocking |
| `pip-audit` | Could not complete because PyPI connection timed out | Audit gap, not evidence of a vulnerability |
| Upstream GitHub workflows at audit base | Quality Gate, Tests, Privacy, and Link Checker successful | External CI corroborates local quality checks |
| Final local Git state | Clean before documentation edit | Audit did not modify runtime source |

## Findings

### A-001 - High: Standalone initialization does not produce a bootable workspace

**Evidence**

- `src/athena/cli/init.py:438-465` writes `Core_Identity.md` to `.framework/modules/Core_Identity.md`.
- `src/athena/boot/constants.py:13-15` reads `.framework/v8.2-stable/modules/Core_Identity.md`.
- `src/athena/boot/constants.py:27-28` pins a SHA-384 hash for the immutable stable identity template.
- `src/athena/boot/loaders/identity.py:22-47` fails boot when the file is absent or its hash differs.
- `tests/test_cli_init.py:80-91` verifies broad directory creation, but no test connects a fresh initialization to a successful default boot.
- A fresh temporary workspace reproduced the failure after successful `athena init` and `athena check`.

**Impact**

The documented standalone setup flow is broken. A user who installs the package and creates a new workspace cannot complete the advertised `/start` path. Clone-based use can mask the issue because the repository already contains `.framework/v8.2-stable/modules/Core_Identity.md`.

**Recommended response**

1. Define one path contract for generated workspaces and the stable framework template.
2. Add an end-to-end test that runs `init`, then boot, and asserts successful session creation.
3. Decide whether `Core_Identity.md` is immutable or user-editable. An immutable hash is appropriate for a shipped reference file, but not for a template intended to be customized.
4. Extend `athena doctor` to validate the boot identity path and hash before reporting a healthy workspace.

### A-002 - High: Packaging scope conflicts with the lightweight local installation claim

**Evidence**

- `pyproject.toml:31-43` places `google-genai`, `supabase`, `anthropic`, `flashrank`, `diskcache`, `fastmcp`, `onnxruntime`, and related packages in base dependencies.
- `pyproject.toml:63-66` defines `[local]` as additional dependencies; Python extras do not remove base dependencies.
- The local verification environment resolved 108 packages, including cloud SDKs and ONNX runtime.
- `README.md:41-55` calls `pip install -e ".[local]"` a lightweight install while also acknowledging that cloud SDKs remain in base dependencies.

**Impact**

The minimum installation is heavier than the documentation implies. This increases install time, dependency-conflict surface, security-review surface, and barriers for users who only want Markdown session management.

**Recommended response**

1. Move optional cloud, MCP, and reranking dependencies out of base dependencies.
2. Keep base dependencies limited to the CLI and local Markdown workflow.
3. Use import guards and explicit capability diagnostics when optional features are unavailable.
4. Add CI jobs for at least base, local, MCP, cloud, and full dependency profiles.

### A-003 - Medium: Public documentation reports private-workspace inventory that is absent here

**Evidence**

- `docs/ARCHITECTURE.md:52-59` describes 87 Python files, 35 test files, and private workspace areas such as `FX Trading/` and `media-factory/`.
- The public checkout contains approximately 102 Python source files and 49 test files.
- `.agent/config/CAPS.json:54-57` reports 285 agent scripts and 5,043 memory files.
- In this checkout, `.agent/scripts` contains 19 source files and `.context/memories/` is absent.
- `.agent/config/CAPS.json:59-79` defines paths and recount rules for skills, protocols, and archived inventories that are not present in this public tree.

**Impact**

Readers can mistake private-workspace scale and capability claims for reproducible public-repository state. This also makes architecture docs less useful as an implementation map.

**Recommended response**

1. Separate public architecture documentation from the author's private workspace inventory.
2. Label private-workspace metrics as contextual, non-reproducible evidence.
3. Generate public inventory counts from the public tree in CI.
4. Move unsupported path claims into an explicit "private upstream only" section.

### A-004 - Medium: Code-enforced governance is narrower and less fail-safe than the product narrative suggests

**Evidence**

- `.claude/settings.json:2-49` wires meta-awareness, secret access, ruin, and stop hooks for Claude Code.
- `README.md:106-121` states that other tools receive rules rather than executable hooks and that Claude hook installation from `athena init` is still in progress.
- `.agent/scripts/security/block_secrets.py:60-63` catches parsing errors and exits successfully, allowing the tool call to continue.
- `.agent/scripts/security/block_ruin.py:60-79` also passes through when hook JSON cannot be parsed.
- `src/athena/core/ruin_check.py:13-37` uses regex patterns plus a structured checker and is necessarily not a complete shell-semantics verifier.

**Impact**

The governance system is meaningful within its supported integration, but it should not be treated as a universal security boundary. Fail-open behavior may be intentional for usability, but it weakens the "code-enforced" claim.

**Recommended response**

1. Document the hook threat model, including intentional fail-open/fail-closed behavior.
2. Add malformed-input tests that pin the chosen behavior.
3. Prefer fail-closed behavior for secret and destructive-operation hooks unless availability is explicitly prioritized.
4. Separate "instruction-level governance" from "runtime-enforced governance" in capability tables.

### A-005 - Medium: Type checking and dependency auditing are not release-blocking

**Evidence**

- Local `mypy src --ignore-missing-imports --no-error-summary` exited 1 with errors in auditors, GTO code, frame ledger, and dynamic boot-constant synchronization.
- `.github/workflows/ci.yml:63-69` marks both mypy and pip-audit as `continue-on-error`.
- Local `pip-audit` could not finish because the PyPI request timed out.
- The repository has no lock file or constraints file.

**Impact**

Known type errors can merge, and dependency audits can be skipped by network instability. Broad `>=` dependency ranges without a lock make fresh installs less reproducible.

**Recommended response**

1. Fix the current mypy errors and remove `continue-on-error`.
2. Add a dependency lock or constraints file for CI and release verification.
3. Cache or retry pip-audit while still failing on audit-tool errors that are not proven transient.
4. Separate vulnerability failures from infrastructure timeouts in CI reporting.

### A-006 - Medium: Repository version and published package version have drifted

**Evidence**

- `pyproject.toml:5-8` declares version `10.0.2`.
- PyPI metadata at audit time reported latest `athena-agent` version `10.0.1`, uploaded on 2026-09-28.
- The audited repository commit is dated 2026-10-04.

**Impact**

Users installing from PyPI do not receive the current main-branch documentation and code. Debugging differs depending on whether the user installed the package or cloned the repository.

**Recommended response**

1. Release `10.0.2` or reset the repository version until release readiness is confirmed.
2. Include package version and commit in `athena doctor` output.
3. Add a release check that warns when main is ahead of the latest published version by more than a patch drift window.

## Positive Observations

- The test suite is substantial and passed locally on Python 3.12.
- Ruff, internal-link, and privacy gates are enforced and passed.
- README capability tables distinguish shipped, prompt-level, in-progress, untested, and N=1 evidence more honestly than most comparable projects.
- Session state is represented as inspectable Markdown with YAML metadata and cache invalidation.
- Hybrid retrieval is implemented as a real multi-channel pipeline rather than a vector-only demo.
- MCP tools are wrapped by a permission layer and support redaction mode.
- GitHub workflows use narrow default permissions and split quality, privacy, link, and test concerns.

## Evidence Boundaries

**Directly supported by this repository or fresh local execution**

- Package identity, supported Python range, dependency groups, and test configuration.
- Init-created paths and boot-required paths.
- Session creation, checkpoint, close, retrieval, permission, and hook implementation.
- Local test, lint, coverage, link, privacy, CLI, and boot-smoke results.
- Public inventory discrepancies between documented counts and files present in the tree.

**Reasonable inference**

- The standalone boot failure is caused by inconsistent generated-template and boot path contracts, not by the temporary environment.
- The public project is a filtered representation of a larger private workspace.
- Current governance should be considered best-effort and Claude-scoped rather than a universal runtime security boundary.

**Not established by this audit**

- Retrieval Hit@5 and MRR results, because the evaluator requires Supabase credentials and the author's indexed private corpus.
- Long-run personalization quality across users; the repository itself labels this as N=1 evidence.
- Dependency vulnerability status, because pip-audit did not complete.
- Behavior in every supported IDE.
- Production safety or data isolation of a live Supabase deployment.

## Suggested Follow-Up Order

1. Fix and regression-test the standalone `init -> doctor -> boot` path.
2. Split dependency profiles and add an installation matrix to CI.
3. Reconcile public docs and CAPS counts with the actual public tree.
4. Decide and document hook failure policy, then test malformed hook input.
5. Make mypy and dependency-audit failures actionable and blocking.
6. Align the repository version with a published PyPI release.

## Audit Status

This document records findings only. It does not change application behavior and does not mark any finding as remediated.
