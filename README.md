# rust-coverage-analysis

An agent skill that analyzes Rust test coverage gaps and produces an actionable report classifying uncovered lines and branches.

Works with Claude Code, Codex, and other agents supported by the [skills CLI](https://github.com/vercel-labs/skills).

## What it does

- Runs `cargo llvm-cov --text -p <crate>` to get instrumented coverage data
- Identifies uncovered lines and branches
- Classifies each gap: business logic, error path, edge case, unreachable, or deliberately untested
- Checks existing tests — including tests whose names suggest coverage but whose bodies miss the path
- Produces a structured report with a concrete mitigation for every must-cover gap

## Prerequisites

```bash
cargo install cargo-llvm-cov
```

The skill will not proceed without this tool installed.

## Installation

### Option 1: Skills CLI

```bash
npx skills add keruch/rust-coverage-analysis
```

### Option 2: Claude Code Plugin

```bash
/plugin marketplace add keruch/rust-coverage-analysis
/plugin install rust-coverage-analysis
```

### Option 3: Manual

```bash
cp -r skills/rust-coverage-analysis ~/.claude/skills/
```

## Usage

The skill triggers automatically in these contexts:

- After implementing logic or writing tests
- After a code review session
- After a security audit of Rust code
- When an implementation plan has just been fully executed
- When you suspect tests are missing important logic paths

Or invoke it explicitly:

> "Run coverage analysis on the `my-crate` crate and report any gaps."

## Available Skills

| Skill | Purpose |
|-------|---------|
| [rust-coverage-analysis](skills/rust-coverage-analysis/SKILL.md) | Analyze Rust test coverage gaps and produce an actionable report |

## License

MIT
