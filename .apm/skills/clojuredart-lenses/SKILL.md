---
name: clojuredart-lenses
description: >-
  Translate code-lenses design philosophies (grug, APOSD, Tidy First, Parse Don't
  Validate, Honest Code, Legacy Code) to idiomatic ClojureDart and Flutter
  patterns. Auto-triggers when working in ClojureDart alongside code-lenses
  skills. Prevents non-idiomatic translations of language-agnostic design advice.
user-invocable: false
---

# Code Lenses for ClojureDart

When the [code-lenses](https://github.com/brackendev/code-lenses) design philosophy skills are active alongside ClojureDart code, use these translations to prevent non-idiomatic advice. Each section maps a code-lenses philosophy to ClojureDart-native tools and patterns.

ClojureDart inherits most Clojure idioms but adds Flutter-specific dimensions: widget composition, reactive state with directives, lifecycle management, and Dart's type system. These translations account for both.

## Grug Brain

Grug philosophy aligns naturally with Clojure and extends to ClojureDart with Flutter-specific instincts:

- Default building blocks are maps, vectors, and sets. Not deftypes, not protocols. Reach for `deftype` only when implementing a Dart interface or extending a Flutter class. The widget tree itself is the primary structure; do not add abstraction layers on top of it.
- "Guard clauses and early returns" means flattening control flow with `cond`, `when`, `if-let`, `some->`, and `some->>`. ClojureDart has no `return` statement. Use `:when` directive inside `f/widget` for conditional rendering instead of wrapping widgets in `if`.
- "Composition over inheritance" means composing widgets through `f/widget` child threading, not subclassing `StatelessWidget` or `StatefulWidget` in ClojureDart. `f/widget` with directives replaces class hierarchies.
- REPL-driven development works in ClojureDart through the socket REPL. Use it to spike widget expressions before introducing abstractions. Evaluate `(f/widget ...)` forms interactively.
- Atoms are the simple state primitive. Use a single atom with a map for app state before reaching for cells, `:bind`/`:get`, or third-party state management. Complexity escalation order: atom, then cell (`f/$`), then `:bind`/`:get` for widget tree sharing.
- Directives (`:watch`, `:managed`, `:bind`, `:get`, `:vsync`) are not complexity demons. They are built-in composition tools that replace Flutter's StatefulWidget boilerplate. Use them when they make code shorter and clearer.
- Type hints (`^String`, `^int`, `^m/Widget`) are not ceremony. They prevent dynamic warnings, which cascade and cause runtime failures. Add them at Dart interop boundaries.
- `f/widget` and `f/build` are the two widget tools. `f/widget` produces a widget. `f/build` produces a builder callback. If you need a widget, use `f/widget`. If Flutter asks for a `builder:` parameter, use `f/build`. Do not introduce wrapper functions for this distinction.

## A Philosophy of Software Design

APOSD maps well to ClojureDart. Translate module-level thinking to ClojureDart-native structures:

- **Namespaces are modules.** A namespace with a small public API (`defn`) and private helpers (`defn-`) is a deep module. In ClojureDart, a screen namespace that exports one `screen` function backed by private widget helpers is a deep module.
- **`f/widget` and directives are deep modules.** `f/widget` has a simple interface (interleaved expressions and directives) but powerful implementation: it handles state, lifecycle, context, and composition behind a declarative surface. Recognize and preserve this depth. Do not reimplement what directives already provide.
- **Information hiding means hiding Flutter plumbing, not data.** ClojureDart idiomatically passes plain maps as data. Hide controller lifecycle (`:managed`), inherited widget lookups (`:get`), and animation setup (`:vsync`) behind namespace boundaries or widget functions. Callers pass data; the widget function handles Flutter mechanics.
- **`defn-` for privacy.** Use private functions for helper widgets that should not be called from other namespaces. Public widget functions form the namespace's API.
- **`ex-info` and `ex-data` for rich errors.** When errors cannot be defined out of existence, use `ex-info` to attach structured context. Handle at the boundary (route builders, event handlers) with `ex-data` destructuring.
- **`:bind`/`:get` for dependency injection.** Shared state, theme data, and services flow through the widget tree via `:bind` and `:get`. This hides wiring complexity from leaf widgets, which only declare what they need via `:get`.
- **Directive-managed lifecycle hides complexity.** `:managed` handles `dispose` automatically. `:vsync` provides a `TickerProvider` without exposing the mixin. These push lifecycle complexity downward so widget functions stay focused on rendering.

## Tidy First

The 15 tidyings apply to ClojureDart with these translations:

- **Guard clauses** means flattening nested `if`/`when`/`let` with `cond`, `if-let`, `when-let`, `some->`, and `some->>`. For widget trees, use `:when` directive to guard rendering rather than wrapping entire subtrees in conditionals:
  ```clojure
  ;; Flat: use :when
  (f/widget
    :when show-detail?
    (detail-panel data))

  ;; Not: nested conditional wrapper
  (if show-detail?
    (detail-panel data)
    (m/SizedBox))
  ```
- **Explaining variables** means extracting complex expressions into named `let` bindings or `:let` directives that reveal intent:
  ```clojure
  (f/widget
    :let [visible-items (filter :visible items)
          item-count (count visible-items)]
    (m/Text (str item-count " items")))
  ```
- **Extract helper** means pulling widget subtrees into named functions. A widget function returns a widget or `f/widget` form. Keep each function focused on one visual concern:
  ```clojure
  (defn item-tile [item]
    (f/widget
      :let [{:keys [title subtitle]} item]
      (m/ListTile .title (m/Text title) .subtitle (m/Text subtitle))))
  ```
- **Reading order** follows Clojure's top-down compilation rule. Define helper widget functions before the screen function that uses them. ClojureDart files compile top-down, so helpers must precede callers unless `declare` is used.
- **Move declaration and initialization together** means placing `:let`, `:watch`, and `:managed` directives close to where their bindings are first used in the widget tree. Split large `f/widget` forms into smaller widget functions when directives serve different concerns.
- **Chunk statements** means adding visual grouping in `f/widget` bodies. Separate directive blocks (state setup) from widget expressions (rendering) with blank lines:
  ```clojure
  (f/widget
    :watch [items app-state]
    :managed [scroll-ctrl (m/ScrollController)]

    (m/ListView.builder
      .controller scroll-ctrl
      .itemCount (count items)
      .itemBuilder (f/build [i] (item-tile (nth items i)))))
  ```
- **Normalize symmetries** means making similar widget branches follow identical structure. When a `case` or `cond` produces different widgets, keep the pattern consistent so differences stand out:
  ```clojure
  (case status
    :loading (m/Center .child (m/CircularProgressIndicator))
    :error   (m/Center .child (m/Text error-message))
    :loaded  (m/Center .child (content-widget data)))
  ```
- **New interface, old implementation** means writing a new widget function with the desired signature that delegates to existing widgets. Useful for wrapping Dart Flutter widgets with ClojureDart-friendly APIs.
- Threading in `f/widget` (child threading through `.child`, dotted symbols for `.home`, `.body`) is a tidying tool built into the macro. Use it instead of manually nesting `.child` parameters.
- `clj-kondo` catches dead code, unused bindings, and structural issues in `.cljd` files. Use it to identify which tidyings to apply.

## Parse, Don't Validate

The principle applies to ClojureDart with Flutter-specific parsing boundaries:

- **Parse at the boundary, trust downstream.** In ClojureDart, boundaries are: user input from text fields, data from HTTP responses, platform channel messages, route parameters, and JSON deserialization. Parse and validate at these points. Widget functions downstream receive parsed data and do not re-validate.
- **Smart constructor functions.** Write a `parse-item` or `make-order` function that validates input and returns a domain map or throws `ex-info` with structured error data. Widget functions that receive the result know it satisfies the invariants:
  ```clojure
  (defn parse-item [raw]
    (let [title (get raw "title")
          price (get raw "price")]
      (when (or (nil? title) (nil? price))
        (throw (ex-info "Invalid item" {:raw raw})))
      {:item/title title
       :item/price (double price)}))
  ```
- **Namespaced keys as domain markers.** Use `:item/title`, `:order/total`, `:user/email` instead of bare `:title`, `:total`, `:email`. Namespaced keys signal that the data has been parsed and belongs to a specific domain concept.
- **Tagged maps for UI state.** Use a `:type` key to distinguish state variants instead of multiple boolean or nullable fields:
  ```clojure
  ;; State atom holds tagged variants
  (def app-state (atom {:type :loading}))

  ;; Transitions produce new tagged maps
  (reset! app-state {:type :loaded :data items})
  (reset! app-state {:type :error :message "Network failure"})

  ;; Widget dispatches on tag
  (f/widget
    :watch [{:keys [type data message]} app-state]
    (case type
      :loading (m/CircularProgressIndicator)
      :loaded  (item-list data)
      :error   (m/Text message)))
  ```
- **`:watch` with `:default` is parsing.** When watching a Future or Stream, the `:default` value defines the initial parsed state. Use it to provide a typed starting point rather than checking for nil:
  ```clojure
  :watch [items (fetch-items) :default []]
  ;; items is always a vector, never nil
  ```
- **Type hints at Dart boundaries are parsing.** `^String`, `^int`, `^m/Widget` hints convert dynamic Dart values into typed ClojureDart values. Without them, the compiler inserts dynamic calls. Type hints are the parse step between Dart and ClojureDart:
  ```clojure
  ;; Untyped (dynamic, fails at runtime)
  (defn greet [name] (str "Hello " name))

  ;; Typed (parsed at boundary)
  (defn greet [^String name] (str "Hello " name))
  ```
- **Route parameters need parsing.** go_router path and query parameters arrive as strings. Parse them into domain types at the route builder boundary, not inside the screen widget.
- **Do not simulate branded types.** Wrapping strings in deftypes to emulate TypeScript-style branded types is non-idiomatic. Use smart constructors, namespaced keys, and type hints instead.

## Honest Code

Most Honest Code constructs are native to Clojure and extend to ClojureDart. Translate the remaining constructs with Flutter-specific adjustments:

- **Construct 1 (Data Is Data):** Already the default. Clojure maps, vectors, and sets are the honest data structures. In ClojureDart, widget configuration is also data: named parameters to constructors are key-value pairs. Keep app state in atoms holding maps, not in stateful widget classes. The serialization test is EDN-printable for app state; Flutter Widget instances are the exception (they are ephemeral render objects, not data).
- **Construct 2 (Input In, Output Out):** Widget functions should be pure: data in, widget out. Side effects belong in `:watch` callbacks, `:bg-watcher`, or `onPressed`/`onTap` handlers, not in the widget build path. A widget function that reads from an atom via `:watch` is honest because the dependency is declared, not hidden:
  ```clojure
  ;; Honest: dependency declared via :watch
  (f/widget
    :watch [count counter-atom]
    (m/Text (str count)))

  ;; Dishonest: hidden side effect in build
  (f/widget
    :let [_ (println "building")]  ;; side effect during build
    (m/Text "hello"))
  ```
- **Construct 3 (One Source of Truth):** One atom per piece of state. Cells (`f/$`) derive from atoms; they do not own state. `:bind`/`:get` shares state through the widget tree without duplicating it. If two widgets need the same data, share via `:bind` at a common ancestor, not by passing atoms as function arguments:
  ```clojure
  ;; One source: atom bound at root
  (f/widget
    :bind {:cart (atom [])}
    (app-scaffold))

  ;; Consumers get the single source
  (f/widget
    :get [:cart]
    :watch [items cart]
    (m/Text (str (count items) " items")))
  ```
- **Construct 5 (Compose Flat, Never Deep):** `f/widget` child threading is flat composition by design. Expressions chain through `.child` sequentially, not nested. Dotted symbols (`.home`, `.body`) redirect the chain. This is the ClojureDart equivalent of middleware stacks. Avoid deeply nested widget constructors; use `f/widget` threading instead:
  ```clojure
  ;; Flat (honest)
  (f/widget
    m/Scaffold
    .body
    m/Center
    (m/Text "hello"))

  ;; Deep (dishonest equivalent)
  (m/Scaffold .body (m/Center .child (m/Text "hello")))
  ```
- **Construct 6 (Let It Crash):** Use `ex-info`/`ex-data` instead of typed exceptions. In Flutter, unhandled errors surface in the red error screen during development. For production, wrap error-prone operations (HTTP, file IO, platform channels) in `try`/`catch` at the boundary and update state to an error variant. Do not catch errors inside widget build paths; let Flutter's error handling surface them.
- **Construct 8 (Boring Tests):** `(is (= (f input) expected))` is the boring test. `cljd.test` with `deftest` and `is` keeps tests flat and repetitive. Widget tests with `pumpWidget` and `find.text` are the boring widget test. If tests need complex mocking or setup, the design is the issue.
- **Construct 10 (Declare What, Not How):** Directives are declarations. `:watch` declares a reactive dependency. `:managed` declares lifecycle ownership. `:bind` declares a shared binding. `:when` declares conditional rendering. Prefer directives over imperative state management code:
  ```clojure
  ;; Declarative (honest)
  (f/widget
    :managed [ctrl (m/TextEditingController)]
    :watch [text ctrl :> .-text]
    (m/Text (str "You typed: " text)))

  ;; Imperative equivalent (dishonest in ClojureDart)
  ;; Manually creating controllers, adding listeners, disposing...
  ```
- **State is not dishonest when explicit.** Atoms with `:watch` are honest state management: mutability is visible, scoped, and reactive. The test is whether state is declared (`:watch`, `:managed`, `:bind`) or hidden (global defs mutated from arbitrary call sites).

## Legacy Code

The book's techniques are OO-flavored. Translate to ClojureDart-native seams and strategies:

- **Seams in ClojureDart:** Higher-order functions (pass a widget function instead of hardcoding it), `:bind`/`:get` (swap bound values for test doubles), and function parameters for dependencies. Prefer extracting pure data-transformation functions over relying on widget-level seams.
- **Extract Interface** translates to extracting a function parameter. A widget function that accepts a `render-item` callback is more testable than one that hardcodes a specific item widget. Protocols are rarely needed in ClojureDart; function parameters suffice.
- **Parameterize Constructor** translates to passing a deps map or using `:bind`/`:get`. Widget functions that need services (HTTP clients, storage) should receive them through `:get` bindings, not reach for global defs.
- **Sprout method/class** translates to sprouting a new widget function. Add new visual behavior in a new `defn` called from the existing widget, rather than modifying an untested `f/widget` form directly.
- **Wrap method/class** translates to a wrapper widget function. Write a new function that composes the original widget with additional behavior (padding, error boundaries, loading states):
  ```clojure
  ;; Wrapper adds loading state around existing widget
  (defn with-loading [loading? content-widget]
    (f/widget
      (if loading?
        (m/Center .child (m/CircularProgressIndicator))
        content-widget)))
  ```
- **Scratch refactoring** maps to REPL-driven exploration. Connect to the socket REPL, evaluate widget expressions, and use `pick!` to inspect live widget state before making targeted changes.
- **Effect sketching** means tracing which namespaces and atoms a change touches. Use `clj-kondo` to map the dependency graph. In ClojureDart, the narrowest test point is a pure function that transforms data, callable directly in the REPL.
- **Hard dependencies in ClojureDart** include direct Dart interop calls embedded in widget functions, `defonce` atoms at the namespace level accessed from multiple namespaces, side effects in the widget build path, and `:require` of concrete implementations where a function parameter or `:bind`/`:get` would allow substitution.
- **Characterization tests** in ClojureDart use `cljd.test`. Capture existing behavior with `(is (= (f known-input) observed-output))` before making changes. For widget behavior, use `pumpWidget` and `find.text` to capture what the current widget renders.
