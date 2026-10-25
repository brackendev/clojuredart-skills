# Exceptions, Resource Cleanup, and Numbers

Part of the `clojuredart` skill. It applies to ClojureDart and overrides the host-neutral `clojure` skill in this domain.

## Catch Types

`(catch Type e body...)` compiles to Dart's `on Type catch (e)`. Dart can throw any non-null object, and its two base types do not overlap:

- `Exception`: a failure the caller is meant to handle, such as `FormatException` from `int/parse`, `IOException`, `TimeoutException`, or `IntegerDivisionByZeroException` from `(quot 1 0)`.
- `Error`: a bug in the program, such as `ArgumentError` (thrown by `(nth [1 2] 5)`), `RangeError`, `StateError`, `TypeError` (a nil passed to a `^String` hint, Iron Law CLJD-3), `UnsupportedError`, or the assertion error from a failed `assert`.

`ex-info` creates a `cljd.core/ExceptionInfo`, which is neither an `Exception` nor an `Error`. `(catch Exception e ...)` does not catch it. Catch it by its full name. The bare `ExceptionInfo` fails to compile with `Can't resolve type ExceptionInfo.`:

```clojure
(try
  (load-profile id)
  (catch cljd.core/ExceptionInfo e
    (handle-domain-failure (ex-data e)))
  (catch FormatException e
    (handle-bad-payload (ex-message e))))
```

The catch-all is `(catch Object e ...)` or `(catch dynamic e ...)`. Upstream `doc/differences.md` documents `dynamic` as the ClojureDart form of Dart's catch clause with no type. `(catch :default e ...)`, the ClojureScript catch-all, fails to compile with `Can't resolve type :default.` In shared `.cljc` code, write the catch-all as `#?(:cljd Object :clj Exception :cljs :default)`, keeping `:clj` last (see Conditional Reading in `references/project-structure.md`).

Effective Dart says "DON'T explicitly catch Error or types that implement it" and "AVOID catches without on clauses". Catch the `Exception` types and `cljd.core/ExceptionInfo` that the code can handle. Keep `Object` and `dynamic` for the outer boundary of an event handler, a background task, or `main`, where the error is reported and then rethrown or turned into an error state.

`(throw e)` on the caught value compiles to Dart's `rethrow`, which keeps the original stack trace. Dart does not attach the stack trace to the exception, so bind it with a second symbol when it is needed: `(catch Exception e st (log e st))`. The second symbol is a stack trace binding only when more body forms follow it. In `(catch Exception e fallback)`, `fallback` is the return value.

`ex-message` returns the message of an `ExceptionInfo`, a `FormatException`, an `ArgumentError` or `RangeError`, a `StateError`, an `UnsupportedError`, an `UnimplementedError`, a `TimeoutException`, an `OSError`, and an `IsolateSpawnException`. For any other `Exception` it returns `(str e)`, such as `"Exception: boom"`. For any other `Error`, such as a `TypeError`, and for thrown values that are not exceptions, it returns nil.

`/` on two integers returns a double, so `(/ 1 0)` returns `##Inf` and throws nothing. Use `quot` for integer division. It throws `IntegerDivisionByZeroException` on a zero divisor.

## Testing Thrown Exceptions

`(is (thrown? Type body...))` wraps the body in `(try ... (catch Type e nil))`. It passes when the body throws a `Type`. It fails when the body returns, and the report shows the value (`Actual: Result:<3>`). An exception of another type escapes the `catch`, and the test reports it as an error (`Actual: Result:<has thrown Exception: boom>`).

Name Dart types. JVM classes such as `ArithmeticException`, `IllegalArgumentException`, and `clojure.lang.ExceptionInfo` do not exist in Dart, although the `cljd.test` docstring still shows `ArithmeticException`. Test `ex-info` with `cljd.core/ExceptionInfo`, and test a thrown non-exception value with `Object`:

```clojure
(deftest parse-amount-test
  (is (thrown? FormatException (parse-amount "abc")))
  (is (thrown? cljd.core/ExceptionInfo (withdraw account 500)))
  (is (thrown-with-msg? cljd.core/ExceptionInfo #"insufficient" (withdraw account 500))))
```

- `thrown-with-msg?` matches the regular expression against `ex-message`. When `ex-message` returns nil, as it does for a `TypeError`, the match itself throws `type 'Null' is not a subtype of type 'String'` and the test reports an error. Use `thrown?` for those types.
- With release `0.9.20261002`, a literal `(throw ...)` as the body of `thrown?` or `thrown-with-msg?` inside `is` fails to compile with `Cannot invoke "clojure.lang.IFn.invoke(Object, Object)" because "aliases" is null`. Call a function that throws instead, which is what a real test does.
- To test a failing `Future`, await it inside the assertion: `(is (thrown? StateError (await (load-state))))`. Without `await`, the body returns the `Future`, and the assertion fails with `Actual: <Instance of 'Future<dynamic>'>`.

## Resource Cleanup with with-open

ClojureDart's `with-open` binds each value, runs the body, and disposes the values in reverse order in a `finally`, so a value is disposed even when the body throws. It calls `.dispose` by default, not `.close`. Follow a binding with `:dispose <form>` to dispose through `(-> value <form>)` instead. The binding form can destructure:

```clojure
(ns my-app.files
  (:require ["dart:io" :as io]))

(defn file-length [^String path]
  (with-open [raf (.openSync (io/File path)) :dispose .closeSync]
    (.lengthSync raf)))

(with-open [{:flds [stream] :as controller} (async/StreamController) :dispose .close]
  ...)
```

Every binding is disposed, so destructure the resource in its own binding, as above. A second binding such as `{:flds [stream]} controller` disposes `controller` again with the default `.dispose`, which prints `DYNAMIC WARNING: can't resolve member dispose on target type StreamController` and throws `NoSuchMethodError` at runtime.

In widgets, give controllers to `:managed` in `f/widget` instead (`references/flutter.md`), because their lifetime is the widget's, not one block's.

The value escapes the block in two cases, and then the resource is disposed before it is used:

- **Lazy sequences.** `(with-open [r (open)] (map f (items r)))` returns an unrealized sequence, and `with-open` disposes `r` first. Realize the result inside the block with `doall`, `vec`, or `into`.
- **Futures.** A body that returns a `Future` without awaiting it, such as `(.then (fetch r) handle)`, disposes `r` before the `Future` completes. Write `(await ...)` inside the block. The function then becomes async, and `with-open` disposes after the awaited work completes.

ClojureDart core has no `clojure.java.io`, `line-seq`, `slurp`, `spit`, or `with-precision`. Read files with `dart:io`. `(.readAsLinesSync (io/File path))` and `(.readAsStringSync (io/File path))` open and close the file themselves, and `(.openRead (io/File path))` returns a `Stream` for large files (`references/async.md`).

## Numbers: M, N, and Ratio Literals

Dart has no `BigDecimal` and no ratio type. The compiler reads number literals with the Clojure reader on the JVM and writes them into Dart with `str`, so they become plain Dart numbers:

| Literal | Dart value | Type |
|---------|------------|------|
| `0.1M` | `0.1` | `double` |
| `1M` | `1` | `int` |
| `1N` | `1` | `int` |
| `1/3` | the expression `1/3`, which is `0.3333333333333333` | `double` |

`(+ 0.1M 0.2M)` is `0.30000000000000004`, so an `M` suffix gives no decimal arithmetic. Do not write `M` literals in ClojureDart code or in shared `.cljc` code that ClojureDart compiles and rely on exact decimals.

At runtime, `cljd.edn/read-string` and `cljd.reader/read-string` throw `FormatException` with the message `BigDecimal not supported yet.` on `"0.1M"`, and `Invalid token: 1/3` on `"1/3"`. They read `"1N"` as the integer `1`. EDN from a JVM service that writes `BigDecimal` values therefore fails to parse. Have the service send such amounts as strings or integers.

Keep money in integer minor units (cents) as `int`. A native `int` holds 64 bits. On the web, Dart represents an `int` as a JavaScript double, which keeps 53 bits of integer precision, so a Flutter web build loses precision above `2^53` (the Dart "Numbers in Dart" documentation).
