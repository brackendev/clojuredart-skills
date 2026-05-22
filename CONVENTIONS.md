# Skill Conventions

This document defines the argument grammar, scope vocabulary, and mutation defaults that every user-invocable skill in `clojuredart-skills` follows. A reader who learns one skill should be able to predict every other skill. New skills follow this document.

These conventions apply to user-invocable skills (`user-invocable: true` in frontmatter). Model-invocable reference skills with no argument surface (currently `clojuredart`, `clojuredart-lenses`, and `clojuredart-nav`) carry no argument grammar and are exempt from rules 1 and 2.

## The three rules

### Rule 1: argument grammar

User-invocable skills accept natural-language keywords and bare paths. The single sanctioned flag is `--report`. No other `--name` flags exist.

Skill-specific modifiers are bare phrases, not flags. Examples used in this plugin:

- `lint`, `format`, `compile`, `dry`: step keywords for `/cljd-fix`
- `unit`, `widget`, `all`: scope keywords for `/cljd-test`
- `all`, `<path>`, `<glob>`: shared scope keywords listed in rule 2

Each skill documents its own modifiers in its `## Arguments` table.

Exemptions are listed in the [Exemptions](#exemptions) section with a reason. The standard exemption pattern is a scaffolding skill that needs a required positional argument because no useful default exists.

### Rule 2: scope vocabulary

Skills that operate on files, diffs, pull requests, or commit messages share these core rows in their `## Arguments` table:

| Input             | Target                                                          |
|-------------------|-----------------------------------------------------------------|
| (no argument)     | The skill's narrowest useful default                            |
| `all`             | Widen the selected scope to its maximum                         |
| `<path>` `<glob>` | Operate on those files or directories                           |

Each skill states what `all` resolves to in concrete terms (whole project, the full codebase, every detected step). The keyword has one uniform meaning: widen the selected scope to the maximum. The unit varies per skill.

A skill whose narrowest useful default is already maximum scope still accepts `all` for family consistency. The table row reads "Same as (no argument); accepted for family consistency."

Opt-in rows (`#N` or PR URL, `pr`, `commit`) appear only when the skill genuinely supports them. None of the current `clojuredart-skills` skills operate on pull requests or commit messages, so the opt-in rows do not appear in this plugin today.

### Rule 3: mutation is the default

Skills that can mutate the workspace apply changes when invoked. The operator passes `--report` to receive a description of what the skill would do, without modifying any files.

Only the literal token `--report` enables report-only mode. Natural-language phrases ("preview", "dry run", "rehearse") are scope input or step keywords, not mode triggers. A skill that conflates them is wrong.

Command suffixes reinforce the default. The family follows a noun-first `<target>-<verb>` pattern, so the trailing verb signals behavior. Skills with suffix `-fix`, `-new`, `-test`, `-upgrade` (verbs that imply action) mutate by default; in this package, `/cljd-fix`, `/cljd-new`, `/cljd-test`, `/cljd-upgrade`, and the forthcoming `/cljd-smells-fix`. Skills with suffix `-review` (a reading verb) are pure-report; this package currently has none.

## Classification

Every user-invocable skill falls into one of two classes.

**Mutating skill** -- default behavior. May carry `--report` when preview is useful. May omit `--report` when preview is meaningless (the operator inspects `git diff` after the fact) or the action is small and reversible.

**Pure report** -- never mutates. No `--report` flag because there is nothing to invert.

A skill is classified by its actual behavior, not by its name. If the name and behavior disagree, the rename is the fix. The classification appears in the skill's own description so the operator knows what to expect.

### Skills in this plugin

| Skill                  | Class           | `--report` available? | Notes |
|------------------------|-----------------|------------------------|-------|
| `/cljd-fix`           | Mutating        | Yes                    | `format` step runs `cljfmt fix` by default. `--report` swaps it for `cljfmt check`. `lint`, `compile`, and `dry` are pure-read regardless. |
| `/cljd-new`            | Mutating        | No                     | Scaffolds the project tree. Preview the side effects by reading `SKILL.md`. |
| `/cljd-test`           | Mutating        | No                     | Runs the test suite and offers to scaffold tests when none exist. Test runs rewrite generated Dart in `lib/cljd-out/` (build artifacts, not source); scaffolding writes source files only after operator confirmation. |
| `/cljd-upgrade`        | Mutating        | Yes                    | Rewrites the `tensegritics/clojuredart` `:sha` in `deps.edn`. `--report` prints the previous and remote SHA without writing. |
| `/cljd-smells-fix`     | Mutating        | Yes                    | Placeholder until the ClojureDart smells catalog ships. When implemented, mirrors the mutation contract of `/clj-smells-fix`: Stage 1 mechanical and Stage 2 DEFECT-band findings auto-applied; `--report` disables all writes. |

Model-invocable skills (`clojuredart`, `clojuredart-lenses`, `clojuredart-nav`) have no argument surface and are not classified here.

## Section structure

User-invocable skills with an argument surface use these section conventions:

- `## Arguments` -- required. Contains the canonical scope table.
- `## Scope` -- optional. Add only when detection order, fallback, or base-branch resolution exceeds what the table can express.
- `## Mutation` -- optional. Add only when mutation behavior needs clarification beyond a single table row (for example, when only one of several steps writes, or when a "scaffold when missing" path writes source files while the main path only writes build artifacts).

The section name `## Customization` is retired.

## Worked examples

### `/cljd-fix` -- mutating skill with `--report`

```
/cljd-fix                    # all four steps; format writes
/cljd-fix lint               # lint only (pure-read; no writes anywhere)
/cljd-fix format             # format step; writes via cljfmt fix
/cljd-fix compile dry        # combined step keywords
/cljd-fix --report           # all four steps; format reads via cljfmt check
/cljd-fix format --report    # format step; no writes
/cljd-fix all                # synonym for (no argument)
```

The skill writes when the `format` step runs without `--report`. With `--report`, the format step runs `cljfmt check`, which reports diffs without writing. The other three steps (`lint`, `compile`, `dry`) are pure-read of source regardless. The `compile` step writes generated Dart under `lib/cljd-out/`; that is a build artifact, not a source mutation, and is unaffected by `--report`. The `## Mutation` section in the skill body documents this asymmetry.

### `/cljd-new` -- mutating skill, exemption from `all`/path rows

```
/cljd-new my-app             # creates project tree at ./my-app
/cljd-new                    # prompts the operator for a project name
```

`<project-name>` is a required positional argument. The skill omits the `all` and `<path>` rows because they would not be meaningful for scaffolding. It also omits `--report` because preview is meaningless: the operator reads `SKILL.md` to see what files will be written. The exemption is listed below.

### `/cljd-upgrade` -- mutating skill with `--report`

```
/cljd-upgrade                # upgrade deps.edn :sha and run clj -M:cljd compile
/cljd-upgrade --report       # print current :sha and remote HEAD; no writes
/cljd-upgrade all            # synonym for (no argument)
```

The default rewrites `:sha` in `deps.edn` and verifies with a compile. With `--report`, the skill prints the current `:sha`, the remote `HEAD` from `git ls-remote`, and the would-be diff. No writes. No compile.

### `/cljd-test` -- mutating skill, no `--report`

```
/cljd-test                   # run all tests
/cljd-test unit              # run unit-tagged tests only
/cljd-test widget            # run widget-tagged tests only
/cljd-test all               # synonym for (no argument)
/cljd-test test/my_app/x_test.cljd   # restrict to that file
```

The default runs the suite. Compilation of `.cljd` to `.dart` test files is a build-artifact write, not source mutation. The "offer to scaffold when no tests exist" path writes source files only after operator confirmation, which is captured in the `## Mutation` section.

### `/cljd-smells-fix` -- mutating skill with `--report` (placeholder)

```
/cljd-smells-fix                         # fix changed files
/cljd-smells-fix src/my_app              # fix files under directory
/cljd-smells-fix src/my_app/core.cljd    # fix specific file
/cljd-smells-fix all                     # fix the full codebase
/cljd-smells-fix --report                # produce the report only; no writes
```

The body remains a placeholder until the ClojureDart smells catalog ships. When implemented, the skill will mirror the mutation contract of `/clj-smells-fix`: Stage 1 mechanical findings and Stage 2 `DEFECT`-tier findings within a defined safety band are auto-applied; `SMELL` and `HINT` findings remain report-only; `--report` disables all writes.

## Exemptions

`/cljd-new` accepts a required positional `<project-name>` because no useful default exists for scaffolding. It omits the `all` and `<path>` scope rows and the `--report` flag for the same reason. The `## Arguments` table in its `SKILL.md` documents the positional grammar.

## Ambiguity notes

**`(no argument)` outside a git worktree.** A skill whose narrowest useful default depends on git state (for example, "review changed files") must define the fallback when no git worktree is present. The expected fallback is to ask the operator what to review rather than to widen silently to `all`. In this plugin, `/cljd-smells-fix` will document this fallback when its body lands; `/cljd-fix`, `/cljd-test`, and `/cljd-upgrade` derive scope from `deps.edn` and the project tree rather than from a diff, so the question does not apply.

**`commit` versus staged-and-unstaged state.** When a future skill accepts `commit` as a scope keyword, it must state whether `commit` means the most recent commit, the staged tree, or the staged-plus-unstaged working tree. The expected default is the most recent commit. None of the current skills carry this scope.

**`--report` versus natural-language synonyms.** "Preview", "dry run", "rehearse", and similar phrases are scope input or step keywords (or operator chatter), never mode triggers. Only the literal `--report` token disables writes. The `/cljd-fix` step keyword `dry` is the dry4clj duplicate-form scan, not a dry-run mode; the conflict is named here so operators do not read `dry` as "dry run".

**Step keywords versus scope keywords.** A skill like `/cljd-fix` accepts step keywords (`lint`, `format`, `compile`, `dry`) that select work to run, and scope keywords (`all`) that widen scope. Step keywords are skill-specific and listed in the skill's own table. Scope keywords are shared and listed here. When a skill has both, the `## Arguments` table lists both with clearly distinct rows.

**Tool-level flags the skill calls internally.** A skill may invoke a tool that itself uses POSIX flags (for example, `clj-kondo --lint`, `cljfmt fix`, `cljfmt check`, `git ls-remote`, `flutter pub add --dev`). Those are tool-level flags, not skill flags, and do not count against rule 1. The skill body should disambiguate when a tool flag could be mistaken for a skill flag.

## Author checklist

When adding or modifying a user-invocable skill, confirm each item before committing.

- [ ] Skill has a `## Arguments` section (or is listed under [Exemptions](#exemptions)).
- [ ] Scope rows match the canonical table; opt-in rows appear only where the skill genuinely supports them.
- [ ] If the skill mutates, the command suffix signals it (`-fix`, `-new`, `-upgrade`, `-deploy`, `-test`, `-sync`, `-prune`, `-rebuild`, `-create`, `-apply`).
- [ ] If the skill mutates and preview is useful, `--report` is documented.
- [ ] If the skill is pure-report, the suffix signals it (`-review`, `-audit`, `-check`) and the skill has no `--report` flag.
- [ ] No `## Customization` section.
- [ ] No `--name` flags other than `--report`. Tool-level flags the skill calls internally (for example, `cljfmt check`, `clj-kondo --lint`) are not skill flags and do not count.
- [ ] Frontmatter `name` matches the skill's directory name.
- [ ] OpenCode mirror under `.opencode/skills/<name>/SKILL.md` is byte-identical to the canonical source.
- [ ] `agents/openai.yaml` `default_prompt` references the current command name.
- [ ] `CHANGELOG.md` records the change under `[Unreleased]` when the change is user-facing.
