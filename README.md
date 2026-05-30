# temper-issue-interp-stackoverflow

Repro for an interpreter crash in Temper `fbcc92f`.

## The bug

`temper test --backend interp` throws `java.lang.StackOverflowError` and exits
without reporting any test results when Temper code recurses to about 330 call
frames. The same tests pass on `--backend js` at depths of 500 and above.

## Root cause, maybe?

**Primary file:** `interp/src/commonMain/kotlin/lang/temper/interp/UserFunction.kt`, line 91
**Supporting file:** `interp/src/commonMain/kotlin/lang/temper/interp/Interpreter.kt`

Every Temper-to-Temper call traverses this chain of JVM frames:

```
interpretCall          (Interpreter.kt:1204)
  -> interpretEdge     (Interpreter.kt:301)
  -> dispatchCall      (Interpreter.kt:925)
    -> invokeUnpositioned (UserFunction.kt:82)
      -> interpretFunctionBody (Interpreter.kt:2052)
        -> interpretBlock      (Interpreter.kt:434)
          -> interpretChild    (Interpreter.kt:574)
            -> interpretTree   (Interpreter.kt:354)
              -> doMaybeSpammy lambda
                -> interpretTreeNotSpammy (Interpreter.kt:363)
                  -> interpretCall (Interpreter.kt:1204)  <-- recurse
```

That is 9–12 JVM frames per Temper call depth. There is no trampoline or
continuation-passing mechanism. The JVM default thread stack holds a few thousand
frames before overflow, which places the effective Temper recursion limit at
approximately 330 calls (non-deterministic ±5 depending on JIT and GC state).

`UserFunction.invokeUnpositioned` at `UserFunction.kt` line 82 is the single
chokepoint: every Temper-to-Temper call passes through it. It holds the complete
information needed to represent a pending call (body, formals, actuals, `cb`, `im`),
and there is one site rather than multiple `interpretFunctionBody` call sites.

## The fix, maybe?

`kotlinx-coroutines` is already declared in `settings.gradle` at version 1.10.2 and
is pulled in by several modules. Adding it to `interp/build.gradle` is the only
build change:

```kotlin
// interp/build.gradle — add to commonMain dependencies
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:$kotlinx_coro_version"
```

With that in place, convert `invokeUnpositioned` to drive a `runBlocking` loop and
`yield()` before each recursive Temper call. A `yield()` releases the current JVM
frame back to the coroutine scheduler, bounding JVM stack depth to O(scheduler
overhead) regardless of Temper call depth:

```kotlin
// UserFunction.kt — replace invokeUnpositioned body
internal fun invokeUnpositioned(
    args: Actuals,
    cb: InterpreterCallback,
    interpMode: InterpMode,
): PartialResult = runBlocking {
    invokeUnpositionedSuspend(args, cb, interpMode)
}

private suspend fun invokeUnpositionedSuspend(
    args: Actuals,
    cb: InterpreterCallback,
    interpMode: InterpMode,
): PartialResult {
    yield()  // releases JVM frame; each Temper call depth costs O(1) heap instead of O(stack)
    // ... rest of existing body unchanged ...
}
```

## How to reproduce

```
git clone https://github.com/notactuallytreyanastasio/temper-issue-interp-stackoverflow
cd temper-issue-interp-stackoverflow
temper test --backend interp   # StackOverflowError, no results reported
temper test --backend js       # Tests passed: 4 of 4
```

## Observed threshold

~330–340 Temper call frames. Non-deterministic ±5 (JIT/GC). Measured on macOS
Apple M-series with the default JVM stack size.

## Version

Temper `fbcc92f`. Reproduced 2026-05-30.

## Context

Found running tests for a [chess9 Temper port](https://github.com/notactuallytreyanastasio/chess9).
The reducer module recurses through the move tree. The full suite crashed the
interpreter immediately, requiring manual test splitting to keep individual test
depths below ~300.
