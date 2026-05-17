---
name: clojuredart-lenses
description: >-
  Translate code-lenses design philosophies (grug, APOSD, Tidy First, Parse Don't
  Validate, Honest Code, Legacy Code) into ClojureDart and Flutter-specific
  patterns. Auto-triggers when working in ClojureDart alongside code-lenses
  skills. Layers on top of clojure-lenses (in clojure-skills) and covers only
  the deltas that come from `cljd.flutter`, Dart interop, widget composition,
  and the Flutter widget lifecycle.
user-invocable: false
---

# Code Lenses for ClojureDart

Layered on top of [clojure-lenses](https://github.com/brackendev/clojure-skills) (in `clojure-skills`), which covers the host-neutral Clojure translations of grug, APOSD, Tidy First, Parse Don't Validate, Honest Code, and Legacy Code. This skill records only the ClojureDart and Flutter deltas that the baseline does not address: widget composition, `cljd.flutter` directives, lifecycle management, type hints at the Dart boundary, and the absence of `with-redefs`.

When the [code-lenses](https://github.com/brackendev/code-lenses) plugin is active in a ClojureDart project, use the baseline `clojure-lenses` translations first; reach for this skill for the Flutter-specific additions below.

## Grug Brain

- Reach for `deftype` only when implementing a Dart interface or extending a Flutter class. The widget tree itself is the primary structure; do not add abstraction layers on top of it.
- For conditional rendering, use the `:when` directive inside `f/widget` rather than wrapping subtrees in `if` / `cond`:
  ```clojure
  ;; Flat: use :when
  (f/widget
    :when show-detail?
    (detail-panel data))
  ```
- Compose widgets through `f/widget` child threading. Do not subclass `StatelessWidget` or `StatefulWidget`; `f/widget` with directives replaces those class hierarchies.
- State complexity escalation order: atom, then cell (`f/$`), then `:bind` / `:get` for widget-tree sharing. Reach for cells or bindings only when a single atom no longer fits.
- Directives (`:watch`, `:managed`, `:bind`, `:get`, `:vsync`) are built-in composition tools that replace Flutter's StatefulWidget boilerplate. They are not complexity demons; use them when they make code shorter.
- Type hints (`^String`, `^int`, `^m/Widget`) prevent dynamic warnings that cascade into runtime failures. They are required ceremony at Dart interop boundaries, not optional.
- `f/widget` produces a widget; `f/build` produces a builder callback. If Flutter asks for a `builder:` parameter, use `f/build`. Do not wrap one in the other.

## A Philosophy of Software Design

- `f/widget` and the directive set are deep modules: a simple declarative surface (interleaved expressions and directives) over powerful implementation (state, lifecycle, context, composition). Preserve that depth; do not reimplement what directives already provide.
- Hide Flutter plumbing behind namespace boundaries: controller lifecycle goes in `:managed`, inherited widget lookups go in `:get`, animation setup goes in `:vsync`. Callers pass data; the widget function handles Flutter mechanics.
- `:bind` and `:get` are the ClojureDart dependency injection mechanism. Shared state, theme data, and services flow through the widget tree without being passed as function arguments.
- Directive-managed lifecycle pushes complexity downward: `:managed` handles `dispose` automatically; `:vsync` provides a `TickerProvider` without exposing the mixin. Widget functions stay focused on rendering.

## Tidy First

- Use `:when` to guard rendering, not nested `if` blocks wrapping entire widget subtrees.
- Use `:let` to introduce intermediate bindings inside `f/widget`:
  ```clojure
  (f/widget
    :let [visible-items (filter :visible items)
          item-count    (count visible-items)]
    (m/Text (str item-count " items")))
  ```
- Extract widget subtrees into named functions. A widget function returns a widget or `f/widget` form. Keep each focused on one visual concern.
- Place `:let`, `:watch`, and `:managed` directives close to where their bindings are first used. Split large `f/widget` forms into smaller widget functions when the directives serve different concerns.
- Chunk directive blocks (state setup) from widget expressions (rendering) with blank lines:
  ```clojure
  (f/widget
    :watch [items app-state]
    :managed [scroll-ctrl (m/ScrollController)]

    (m/ListView.builder
      .controller scroll-ctrl
      .itemCount (count items)
      .itemBuilder (f/build [i] (item-tile (nth items i)))))
  ```
- Keep similar widget branches symmetric so differences stand out:
  ```clojure
  (case status
    :loading (m/Center .child (m/CircularProgressIndicator))
    :error   (m/Center .child (m/Text error-message))
    :loaded  (m/Center .child (content-widget data)))
  ```
- Use `f/widget` child threading and dotted symbols (`.home`, `.body`) instead of manually nesting `.child` parameters.

## Parse, Don't Validate

- Parse at Dart boundaries. The ClojureDart-specific entry points are: text-field input, HTTP responses, platform-channel messages, route parameters, and JSON deserialization. Widget functions downstream receive parsed data and do not re-validate.
- Route parameters arrive as strings (path and query). Parse them into domain types at the route builder boundary, not inside the screen widget.
- `:watch` with `:default` is parsing: the default value defines the initial parsed state.
  ```clojure
  :watch [items (fetch-items) :default []]
  ;; items is always a vector, never nil
  ```
- Type hints at Dart boundaries are parsing: `^String`, `^int`, `^m/Widget` convert dynamic Dart values into typed ClojureDart values. Without them, the compiler inserts dynamic calls. Hints are the parse step between Dart and ClojureDart.
- Do not simulate branded types with `deftype` wrappers; the baseline rule against simulated branded types still applies, and ClojureDart adds the further cost that wrapper types impede Dart interop.

## Honest Code

- Widget configuration is data. Named parameters to Dart constructors are key-value pairs; keep widget builders honest by passing data, not by constructing widgets through side effects. Flutter Widget instances themselves are ephemeral render objects, not state; keep app state in atoms holding maps.
- `:watch` makes dependencies declared rather than hidden:
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
- One atom per piece of state. Cells (`f/$`) derive from atoms; they do not own state. `:bind` and `:get` share state through the widget tree without duplicating it. If two widgets need the same data, share via `:bind` at a common ancestor.
- `f/widget` child threading is flat composition by design. Expressions chain through `.child` sequentially. Dotted symbols (`.home`, `.body`) redirect the chain. Avoid deeply nested widget constructors:
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
- Let It Crash, ClojureDart edition: do not catch errors inside widget build paths. Let Flutter's error handling surface them (the red error screen during development). For production, wrap error-prone operations (HTTP, file I/O, platform channels) at the boundary and update state to an error variant.
- Directives are declarations: `:watch` declares a reactive dependency; `:managed` declares lifecycle ownership; `:bind` declares a shared binding; `:when` declares conditional rendering. Prefer directives over imperative state management code:
  ```clojure
  ;; Declarative (honest)
  (f/widget
    :managed [ctrl (m/TextEditingController)]
    :watch [text ctrl :> .-text]
    (m/Text (str "You typed: " text)))
  ```

## Legacy Code

- ClojureDart seams: higher-order functions (pass a widget function instead of hardcoding it), `:bind` / `:get` (swap bound values for test doubles), and function parameters for dependencies. ClojureDart does not ship `with-redefs`; the baseline assumption that vars can be temporarily rebound at test time does not hold.
- Extract Interface translates to extracting a function parameter. A widget function that accepts a `render-item` callback is more testable than one that hardcodes a specific item widget. Protocols are rarely needed in ClojureDart; function parameters suffice.
- Parameterize Constructor translates to passing a deps map or using `:bind` / `:get`. Widget functions that need services (HTTP clients, storage) should receive them through `:get` bindings, not reach for global defs.
- Sprout method/class translates to sprouting a new widget function. Add new visual behavior in a new `defn` called from the existing widget, rather than modifying an untested `f/widget` form directly.
- Wrap method/class translates to a wrapper widget function. Compose the original widget with additional behavior (padding, error boundaries, loading states):
  ```clojure
  (defn with-loading [loading? content-widget]
    (f/widget
      (if loading?
        (m/Center .child (m/CircularProgressIndicator))
        content-widget)))
  ```
- Scratch refactoring maps to REPL-driven exploration through the ClojureDart socket REPL. Use `pick!` to inspect live widget state before making targeted changes.
- Hard dependencies in ClojureDart include: direct Dart interop calls embedded in widget functions, `defonce` atoms at the namespace level accessed from multiple namespaces, side effects in the widget build path, and `:require` of concrete implementations where a function parameter or `:bind` / `:get` would allow substitution.
- Characterization tests use `cljd.test`. For widget behavior, use `pumpWidget` and `find.text` to capture what the current widget renders before changing it.
