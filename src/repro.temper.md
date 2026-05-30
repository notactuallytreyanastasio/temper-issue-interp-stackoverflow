# Bug: Interpreter backend crashes with `StackOverflowError` at ~330 recursive calls

Temper `fbcc92f` — confirmed 2026-05-30

## Summary

`--backend interp` throws `java.lang.StackOverflowError` and exits without
reporting any test results when Temper code recurses to approximately 330
call frames. The JS backend handles the same code at depth 500+ without
issue.

Root cause: `Interpreter.kt` uses direct JVM recursion. Each Temper function
call consumes ~10–15 JVM stack frames (interpretCall → interpretTree →
dispatchCall → withMacroEnvironment …). At ~330 Temper frames that is
~3,300–5,000 JVM frames, which exhausts the default JVM thread stack.

## Repro

    export let count(n: Int): Int {
      if (n <= 0) { 0 } else { 1 + count(n - 1) }
    }

    test("depth 100 — passes on all backends") {
      assert(count(100) == 100) { "depth 100" };
    }

    test("depth 300 — passes on all backends") {
      assert(count(300) == 300) { "depth 300" };
    }

    test("depth 400 — passes on js, crashes interp process") {
      assert(count(400) == 400) { "depth 400" };
    }

    test("depth 500 — passes on js, crashes interp process") {
      assert(count(500) == 500) { "depth 500" };
    }

## How to reproduce

Run `temper test --backend interp` and observe StackOverflowError.
Run `temper test --backend js` and observe all 4 tests pass.

## Observed threshold

Non-deterministic around 330–340 Temper frames (varies ±5 with JIT/GC state).
Measured on Apple M-series with the default JVM stack size.

## Impact

- The interp runner process exits with an unhandled JVM exception.
  No test results are reported — not even for tests that completed before
  the overflow.
- No way to configure the recursion limit or raise the JVM stack via
  the `temper` CLI.
- Any recursive engine (tree traversal, minimax, recursive descent) cannot
  be validated with the interpreter backend.

## Suggested fix

Trampoline the interpreter's recursive dispatch in `Interpreter.kt` to
avoid consuming JVM stack frames per Temper call. Alternatively, expose
a `--jvm-stack-size` flag that passes `-Xss` to the underlying JVM.

## Context

Found running the test suite for a chess9 Temper port. The reducer and
legality modules recurse through a move tree; the full suite crashed the
interpreter and required manual test splitting. See:
https://github.com/notactuallytreyanastasio/chess9
