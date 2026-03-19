---
name: rust-coverage-analysis
description: Analyzes Rust test coverage gaps and produces an actionable report classifying uncovered lines and branches. Only use this skill when cargo-llvm-cov is available — do not trigger if the tool is not installed. Use after implementing logic, after writing tests, after a code review session, or after a security audit of Rust code. Also trigger proactively when an implementation plan has just been executed (all steps done, tests pass), when code review feedback has been received and addressed, when a coverage report shows uncovered lines or branches, when the user suspects tests are missing important logic paths, or when asked to verify test completeness for a Rust crate. Applies only to Rust code.
compatibility: "Requires Rust toolchain with Cargo. Requires cargo-llvm-cov. Applies to Rust crates only."
---

# Rust Coverage Analysis

Identify meaningful gaps in a Rust test suite, classify them, and produce an actionable report. This skill is **read-only** — it surfaces findings but does not write tests.

## Prerequisite Check

Before anything else, verify `cargo-llvm-cov` is installed:

```bash
cargo llvm-cov --version
```

If the command is not found, tell the user:

> `cargo-llvm-cov` is required. Install with:
> ```bash
> cargo install cargo-llvm-cov
> ```
> Then re-run.

Do not proceed until the tool is available. Do not fall back to static analysis — instrumented coverage data is what makes this skill useful.

## Workflow

### 1. Identify the Crate

If the user didn't specify a crate name, check `Cargo.toml` for `[package] name`. In a Cargo workspace, list members to pick the right one — don't guess.

### 2. Run Coverage

```bash
cargo llvm-cov --text -p <crate-name>
```

`--text` produces a line-by-line report. What to look for:
- **`0`** next to a line → that line was never executed during tests
- **`^0`** → a branch arm was never taken

If the crate has features that affect which code is compiled (e.g., `export-abi`), run without them unless the user asks otherwise — test coverage typically applies to default features.

### 3. Collect Gaps

Scan the output and list every uncovered location: file, line range, and the enclosing function or block. Group by file. This forms the raw gap list you'll classify next.

### 4. Read Context

For each gap, read the surrounding code. A single uncovered line rarely tells the whole story — understanding the function it's in, the type it handles, and the invariants it enforces is necessary to classify it correctly.

### 5. Classify Each Gap

Assign one category per gap. When in doubt between "edge case" and "unreachable", prefer "edge case" — err on the side of covering it.

| Category | Meaning | Action |
|---|---|---|
| **Business logic** | Core behavior — a function's main purpose | Must cover |
| **Error path** | `Result::Err` arm, rejected input, guard clause | Must cover |
| **Edge case** | Empty input, boundary value, overflow, off-by-one | Must cover |
| **Unreachable / dead code** | `unreachable!()`, exhaustive-match fallthrough, code gated behind a condition that can never be true | Note it, skip |
| **Deliberately untested** | Panic in an infallible path, proc-macro artifact, generated boilerplate | Note it, skip |

"Must cover" means: if a bug is introduced here, no test will catch it. That's the key question — what goes undetected if this gap stays open?

### 6. Check Existing Tests

Before recommending any fix, look at what's already there. The right mitigation is often a small change to an existing test, not a new one. For every must-cover gap:

1. Search for tests whose name or structure is close to the gap
2. Read those test bodies — do they actually set up the right preconditions to reach the uncovered line?
3. If a test almost covers the gap, name it in the mitigation and describe the specific change needed
4. Only recommend a new test when no existing test is close

Pay special attention to tests whose name suggests they *should* cover the gap but whose body does not set up the required state. This is a common source of false confidence — the test is green because it silently exercises a fallback path rather than the intended one. When you find this, call it out explicitly in the mitigation: name the test, explain why it misses the path, and describe the setup that is missing.

### 7. Produce the Report

Use the format below. Keep it factual and actionable. Every must-cover gap should end with a concrete mitigation that either names an existing test to fix or describes a new one to write.

---

## Report Format

```
# Coverage Analysis: <crate-name>

## Summary
- Total gaps found: N
- Must-cover: N  (Business logic: X | Error paths: Y | Edge cases: Z)
- Skipped: N  (Unreachable: A | Deliberately untested: B)

---

## Must-Cover Gaps

### Gap 1 — <file>:<line-range>
**Category**: Error path
**Code**: `Err(Error::InsufficientFunds(_))` arm in `transfer()`
**Why it matters**: If this arm is never exercised, no test verifies that the contract rejects underfunded calls. A regression here is completely silent.
**Mitigation**: `test_transfer_insufficient_funds` nearly reaches this — add `assert_matches!(result, Err(Error::InsufficientFunds(_)))` and pass an amount one unit below the balance.

---

### Gap 3 — <file>:<line-range>
**Category**: Edge case
**Code**: `if amount == 0 { return Err(...) }` guard in `deposit()`
**Why it matters**: Zero-amount deposits may bypass accounting logic downstream. No test currently exercises this guard.
**Mitigation**: New test `test_deposit_zero_amount` — call `deposit(0)` and assert the expected error is returned.

---

## Skipped Gaps

### <file>:<line-range>
**Category**: Unreachable
**Reason**: `unreachable!()` following an exhaustive match on a closed enum. Cannot be reached at runtime.

---

## Overall Assessment

<1-3 sentence verdict: which gaps are most urgent, what the coverage story looks like overall, and whether the suite gives adequate confidence in the contract's correctness.>
```

If there are no must-cover gaps, say so explicitly — that's a valid and useful result.
