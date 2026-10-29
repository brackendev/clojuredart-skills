# ClojureDart rules from clojuredart-skills

This is a ClojureDart project. The Iron Laws below come from the `clojuredart` skill and add to the `clojure` baseline laws. Never write code that breaks one. Before writing, reviewing, or refactoring code in this project, load the `clojuredart` skill along with the `clojure` skill. The skills hold the full laws, their exceptions, and the references.

## Iron Laws

- CLJD-1: Never edit generated Dart in `lib/cljd-out/`. Make every change in the `.cljd` sources.
- CLJD-2: Treat every dynamic warning as a build error. Fix the first warning first, and add `{:no-dynamic true}` to a namespace to make its warnings compile errors.
- CLJD-3: Hint `^Type?` when a value can be nil, because `^String x` in ClojureDart does not allow nil.
