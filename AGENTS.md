# Rustledger Development Guidelines

This document provides context for AI assistants working on the rustledger codebase.

## Project Overview

Rustledger is a pure Rust implementation of Beancount, the double-entry bookkeeping language. It provides a 10-30x faster alternative to Python beancount with full syntax compatibility.

## Architecture

The project is a Cargo workspace with these crates:

| Crate | Purpose |
|-------|---------|
| `rustledger-core` | Core types (Amount, Position, Inventory, Directives) |
| `rustledger-parser` | Lexer and parser with error recovery |
| `rustledger-loader` | File loading, includes, options |
| `rustledger-booking` | Interpolation and booking engine (7 methods) |
| `rustledger-validate` | Validation with 26 error codes |
| `rustledger-query` | BQL query engine |
| `rustledger-completion` | Editor-agnostic completion logic (shared by LSP + WASM) |
| `rustledger-plugin` | Native and WASM plugin system (30 plugins) |
| `rustledger-plugin-types` | Shared plugin type definitions |
| `rustledger-importer` | Import framework for bank statements |
| `rustledger-ops` | Pure operations on directives — dedup, categorize, reconcile |
| `rustledger` | CLI tool (`rledger check`, `rledger query`, etc.) |
| `rustledger-wasm` | WebAssembly library target |
| `rustledger-lsp` | Language Server Protocol implementation |
| `rustledger-ffi-wasi` | FFI via WASI (wasip1) JSON-RPC for embedding — current shipping surface |
| `rustledger-ffi-component` | FFI via WASI Preview 2 / Component Model (typed WIT contract); successor to `-ffi-wasi`, dual-shipped during #1384 |

## Rust-Specific Patterns

### Error Handling Strategy

This project uses a layered error handling approach:

| Layer | Crate | Pattern |
|-------|-------|---------|
| Library crates | `rustledger-*` | `thiserror` with typed errors |
| CLI binaries | `rustledger` | `anyhow` for ergonomic error chains |
| Tests | All | `anyhow` or `.unwrap()` with clear context |

**Error type conventions:**

```rust
// In library code - use thiserror
#[derive(Debug, thiserror::Error)]
pub enum ParseError {
    #[error("unexpected token at {location}: expected {expected}, found {found}")]
    UnexpectedToken { location: Span, expected: String, found: String },

    #[error("invalid amount: {0}")]
    InvalidAmount(#[from] DecimalError),
}

// In CLI code - use anyhow
fn main() -> anyhow::Result<()> {
    let ledger = load_file(&path).context("failed to load ledger")?;
    Ok(())
}
```

**Rules:**

- Never use `.unwrap()` in library code unless mathematically impossible to fail
- Use `.expect("reason")` only when failure indicates a bug
- Propagate errors with `?`, add context with `.context()` or `.with_context()`
- User-facing errors must include file location (path, line, column)

### Async Runtime

This project is **synchronous** - no async runtime is used. Reasons:

- File I/O is the only blocking operation, and it's fast enough sync
- Simpler mental model for accounting calculations
- WASM target doesn't support async well

**If async is ever needed:**

- Use `tokio` (not async-std)
- Prefer `tokio::fs` over `std::fs` in async contexts
- Use `#[tokio::main]` for CLI, `#[tokio::test]` for async tests

### Memory Management Guidelines

| Type | When to Use |
|------|-------------|
| `&str` | Borrowed string, no ownership needed |
| `String` | Owned string, needs modification or storage |
| `Cow<'a, str>` | May or may not need to own (parsing, transformations) |
| `Arc<str>` | Shared ownership across threads (interned strings) |
| `Box<T>` | Single ownership of heap data, rare in this codebase |
| `Rc<T>` | **Avoid** - not thread-safe, use `Arc` instead |

**String interning:**
The project uses string interning for frequently repeated values:

- Account names (`Assets:Bank:Checking`)
- Currency codes (`USD`, `EUR`)
- Payee names

Interned strings are stored as `Arc<str>` in a global interner. Use `intern_string()` to intern.

**Allocation guidelines:**

- Prefer `SmallVec<[T; N]>` for small, bounded collections (e.g., posting legs)
- Use `Vec::with_capacity()` when size is known
- Avoid `clone()` in hot paths - prefer borrowing
- Use `Cow` for functions that sometimes need to modify strings

### Type-First Development

When implementing new features:

1. **Define types first** - structs, enums, traits
1. **Write trait bounds** - what capabilities are needed?
1. **Let the compiler guide you** - fix errors iteratively
1. **Implement logic last** - once types compile, logic is constrained

This approach works well with AI assistance because:

- Rust's compiler provides structured feedback
- Type errors are specific and actionable
- The AI can iterate quickly on compiler output

## Code Standards

### Rust Idioms

- Use `Result<T, E>` for fallible operations, not panics
- Prefer `?` operator over `.unwrap()` in production code
- Use `thiserror` for error types, `anyhow` in CLI/tests only
- Prefer iterators over explicit loops where idiomatic
- Use `#[must_use]` on functions returning important values
- Prefer `&str` over `String` when possible
- Use `Cow<'a, str>` for potentially-owned strings

### Testing

| Test Type | Location | Framework |
|-----------|----------|-----------|
| Unit tests | `#[cfg(test)]` in source files | `#[test]` |
| Integration tests | `crates/*/tests/*.rs` | `#[test]` |
| Snapshot tests | Parser output, error messages | `insta` |
| Property tests | Invariants, roundtrips | `proptest` |
| Fuzzing | Parser, untrusted input | `cargo-fuzz` (nightly) |

**Test naming:** `test_<function>_<scenario>`

```rust
#[test]
fn test_parse_amount_with_commodity() { }

#[test]
fn test_parse_amount_negative_value() { }

#[test]
fn test_parse_amount_missing_currency_returns_error() { }
```

### Documentation

- All public items must have doc comments
- Include examples in doc comments where helpful
- Use `# Errors` section to document error conditions
- Use `# Panics` section if function can panic (should be rare)

## Build Commands

```bash
# Quick feedback loop (run after every change)
cargo check --all-features --all-targets

# Full test suite
cargo test --all-features

# Lint with warnings as errors
cargo clippy --all-features --all-targets -- -D warnings

# Format all files
treefmt

# Security audit
cargo deny check

# Coverage report
cargo llvm-cov --html
```

## Common Tasks

### Adding a New Plugin

1. Create struct implementing `NativePlugin` trait in `rustledger-plugin/src/native/`
1. Register in `NativePluginRegistry::new()`
1. Add tests in `tests/native_plugins_test.rs`

### Adding a BQL Function

1. Add case to `evaluate_function()` in `rustledger-query/src/executor.rs`
1. Add completion in `rustledger-query/src/completions.rs`
1. Add tests and documentation

### Adding a Validation Error

1. Add variant to `ValidationError` enum in `rustledger-validate/src/lib.rs`
1. Implement detection in `validate_*` function
1. Add tests covering the error case

## Git Workflow

- Use conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`
- PRs require CI to pass before merge
- Use merge queue for main branch

## Security Considerations

- **Parser**: Must handle malformed input gracefully (no panics)
- **Loader**: Must prevent path traversal in `include` directives
- **WASM**: Must be sandboxed, no file system access
- **Dependencies**: Check for known vulnerabilities with `cargo deny`

## Performance Guidelines

- Profile before optimizing - correctness first
- Avoid unnecessary allocations in hot paths
- Use `SmallVec` for small, stack-allocated collections
- String interning is used for accounts, currencies, payees

## What NOT to Do

- Don't add features beyond what's requested
- Don't refactor code that isn't related to the task
- Don't add unnecessary error handling for impossible cases
- Don't create abstractions for one-time operations
- Don't add backwards-compatibility shims - just change the code

## AI Commenting Policy

**An AI assistant must not post a comment or reply into any Issue, Pull
Request, or Discussion where a human other than the maintainer (`robcohen`) is
involved.** It drafts the text to a file and hands over the path; the
maintainer reads it and posts it himself.

The point is that no AI-written text reaches another person under the
maintainer's name without him having read it first.

### Scope

Every repository in the `rustledger` organization: `rustledger`, `rustfava`,
`pta-standards`, `scoop-rustledger`, `homebrew-rustledger`, `oss-fuzz`, and
`.github`.

### What is still permitted

| Action | Allowed | Why |
|--------|---------|-----|
| Commenting on a thread where `robcohen` is the only human | Yes | Nobody else is being written to |
| Replying to and resolving a **bot** review thread | Yes | Bots are not people, and `main` requires conversation resolution — refusing to resolve would stall every PR |
| Opening a new Issue | Yes | The restriction is comments and replies on existing threads |
| Writing or editing a PR body / description | Yes | Same |
| Commit messages, code, branches | Yes | These reach the maintainer through review before they reach anyone else |
| Commenting on a thread any other human has touched | **No** | Draft it instead |

### Checking before commenting

Participation is checked, never assumed. The check below covers Issues, Pull
Requests, and Discussions, and it **fails closed**: if a thread cannot be
positively identified and fully read, it reports that rather than reporting
nobody.

```bash
# Humans other than the maintainer involved in Issue, PR, or Discussion <N>.
#
#   empty output  => the maintainer's alone => an AI may comment directly
#   anything else => draft only
#
# Fails CLOSED: if the thread cannot be identified and fully read, it prints
# `!unchecked: <reason>` on stdout and returns non-zero. Stdout, not stderr, so
# that a caller testing "is the output empty?" treats a failed check as
# "someone is involved" rather than as permission.
others() {
  repo="${1:-rustledger/rustledger}"; n="$2"
  if kind=$(gh api "repos/$repo/issues/$n" --jq 'if .pull_request then "pr" else "issue" end' 2>/dev/null); then :
  elif gh api "repos/$repo/discussions/$n" --jq '.number' >/dev/null 2>&1; then kind=discussion
  else echo "!unchecked: could not read $repo#$n as an issue, PR, or discussion"; return 2
  fi
  case "$kind" in
    issue)      eps="issues/$n issues/$n/comments" ;;
    pr)         eps="issues/$n issues/$n/comments pulls/$n/comments pulls/$n/reviews" ;;
    discussion) eps="discussions/$n discussions/$n/comments" ;;  # comments include replies, flattened
  esac
  names=""
  for ep in $eps; do
    case "$ep" in
      */comments|*/reviews) q='.[] | select(.user.type=="User") | .user.login' ;;
      *)                    q='select(.user.type=="User") | .user.login' ;;
    esac
    # Every endpoint here exists for this kind of thread, so ANY failure means
    # the check did not happen. No 404 is expected, so none is swallowed.
    if ! out=$(gh api --paginate "repos/$repo/$ep" --jq "$q" 2>/dev/null); then
      echo "!unchecked: $ep failed"; return 2
    fi
    names="$names
$out"
  done
  printf '%s\n' "$names" | sort -u | grep -v '^robcohen$' | grep -v '^$'
  return 0
}
```

Why it is shaped like this — each point was a wrong answer in an earlier
version:

- **It fails closed.** An earlier version swallowed failures to avoid tripping
  on 404s, so an expired token made it return nothing — permission — on a
  thread with two other humans on it. A check that cannot run must not grant.
- **It identifies the thread first.** Issues, PRs, and Discussions share one
  number sequence, and a Discussion number 404s on `issues/<N>`. Knowing the
  type up front means no 404 is ever expected, so none has to be ignored.
- **Discussions have their own endpoints.** `discussions/<N>/comments` returns
  replies flattened alongside top-level comments, so a person who only ever
  replied inside a thread is still found.
- **A PR needs all four endpoints.** Review-thread comments are a separate API,
  and an inline human reviewer appears only in `pulls/<N>/comments`.
- **Bots are identified by `.user.type`, never by name.** The same reviewer is
  `Copilot` on one endpoint and `copilot-pull-request-reviewer[bot]` on another.
- **Every call paginates.** Threads here reach 197 comments against a default
  page of 30.

Verified against this repo:

| Thread | Result |
|--------|--------|
| Issue #1387 | `bkuhn caesar` → draft only |
| Issue #923 | `alensiljak` → draft only |
| Discussion #2268 | `petemounce vqv` → draft only |
| Discussion #881, #1420 | `alensiljak`, `zacchiro` — found only in replies → draft only |
| Discussion #2293 | `apearson`, the author, who never commented → draft only |
| PR #2300, Issue #2302 | nothing → may comment |
| Expired token, mistyped repo, nonexistent number | `!unchecked` → draft only |
| An endpoint failing after the thread is identified | `!unchecked` → draft only |

### Where this policy is enforced

The policy covers the whole organization, but only this repository carries it
in `CLAUDE.md` and `AGENTS.md`; no other org repository has either file. An
assistant working in another org repository will not read this document, so do
not rely on it as the enforcement there.

### When the policy blocks a comment

Write the intended comment to a file, then say plainly that it was not posted
and where it is. Do not post a shortened version, and do not route the message
through a bot thread.
