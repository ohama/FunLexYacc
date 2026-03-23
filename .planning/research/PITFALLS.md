# Pitfalls Research

**Domain:** Lexer/parser generator — bootstrapping LangThree with funlex/funyacc
**Researched:** 2026-03-23
**Confidence:** HIGH

---

## Critical Pitfalls

### Pitfall 1: Lookahead Propagation Bugs in LALR(1) Table Generation

**What goes wrong:**
LALR(1) is constructed by merging LR(1) states that share the same LR(0) core. During lookahead propagation, the algorithm must correctly identify which lookaheads are spontaneously generated (derived from FIRST sets) versus which are propagated (inherited from predecessor items). A bug in this phase produces a table that is either more restrictive than it should be (spurious reduce-reduce conflicts) or more permissive (silently accepting invalid programs).

The most common implementation bug: failing to include epsilon-reachable items in the closure. If item `A -> alpha . B beta` is in a set and `B -> . gamma` is in the closure, the lookaheads for `B -> . gamma` depend on `FIRST(beta)`. If `beta =>* epsilon`, then the lookaheads for `A ->` also propagate to `B -> . gamma`. Forgetting the epsilon case causes incomplete lookahead sets, producing spurious reduce-reduce conflicts that appear nowhere in the .funy grammar.

**Why it happens:**
Implementers often treat the closure operation as simple BFS over items without correctly threading the epsilon reachability check. The Dragon Book algorithm is correct but has subtle index-off-by-one errors when translated from pseudocode.

**How to avoid:**
Implement closure incrementally with a worklist. For each item `A -> alpha . B beta, LA`, compute `FIRST(beta LA)` (not just `FIRST(beta)`) as the lookahead set added to all `B -> . gamma` items. Test each nonterminal for nullability before skipping the outer lookahead.

Build a regression test: take the actual LangThree Parser.funy and verify the generated table has zero conflicts. Parser.funy is known to parse cleanly under fsyacc; any conflict in the funyacc-generated table is a bug.

**Warning signs:**
- Any reduce-reduce conflict on a grammar that fsyacc accepts cleanly
- Shift-reduce conflicts on tokens where precedence rules should resolve them but don't
- Parser accepts inputs it should reject, or rejects inputs it should accept, with no obvious grammar change

**Phase to address:** LALR(1) table generation phase (core algorithm milestone)

---

### Pitfall 2: Bootstrapping Chicken-and-Egg: Testing Before Self-Hosting Works

**What goes wrong:**
The goal is for funlex/funyacc (written in LangThree) to regenerate LangThree's own Lexer.fun and Parser.fun, which then replace the F# Lexer.fs and Parser.fs. If you try to self-host too early — before the generated output is functionally equivalent — you break the LangThree interpreter needed to run funyacc itself. This creates a state where you cannot test, cannot run, and cannot easily revert.

The specific failure pattern: funyacc generates a Parser.fun that is nearly correct but has one wrong action in a match clause. You copy it into the LangThree pipeline. LangThree now fails on 400 of 641 tests. You cannot run funyacc to regenerate because LangThree itself is broken. You have destroyed your test harness.

**Why it happens:**
Eagerness to prove bootstrapping works leads to swapping in the generated files before they pass 100% of the existing test suite in "side by side" mode (generated code called alongside the existing F# code, with outputs compared).

**How to avoid:**
Establish a strict two-phase test protocol:
1. **Side-by-side phase**: funlex/funyacc generates output files to a temp directory. A test harness runs the generated lexer/parser against all 641 test inputs and compares output against the F# reference. Do NOT copy into the live pipeline until 100% match.
2. **Cutover phase**: Only after side-by-side parity, replace F# files with generated .fun files.

Keep the F# reference implementation checked in and never delete it during development. Maintain a "rollback" build target that forces the F# path.

**Warning signs:**
- Temptation to "try it and see" with partial test coverage
- Generating code that passes 95%+ of tests and declaring it ready
- Any state where the LangThree test suite count drops below 641 while funyacc is still being developed

**Phase to address:** Must be addressed in project architecture design (Phase 1), enforced throughout every subsequent milestone

---

### Pitfall 3: IndentFilter Context Stack Reconstruction

**What goes wrong:**
The LangThree IndentFilter maintains a 7-context stack (`TopLevel`, `InMatch`, `InTry`, `InFunctionApp`, `InLetDecl`, `InExprBlock`, `InModule`) with intricate context-switching rules. The most fragile transition is `InLetDecl` with the offside rule: when the next token's column is at or below `offsideCol`, an implicit `IN` is emitted, and the `InLetDecl` frame is popped. This logic has three separate trigger points (same-level newline, DEDENT newline, EOF), and implementing even one of them wrong produces silent misparses.

When rewriting IndentFilter in LangThree (.fun), the temptation is to simplify the context stack into a simpler state machine. Any simplification that collapses these contexts will fail on the 30+ offside/implicit-in integration tests.

**Why it happens:**
The IndentFilter is highly stateful (mutable state with 7 fields in the original F# implementation). When rewriting in a functional style, developers restructure the state machine and miss edge cases, especially:
- Single-line `match`/`try` on same line as keyword (JustSawMatch flag consumed before a PIPE on the same line)
- `InFunctionApp` suppressing the offside rule inside multi-line applications
- Nested `let` inside `InMatch` still triggering offside
- Explicit `IN` after a block-let silently popping the `InLetDecl` frame

**How to avoid:**
Port the IndentFilter mechanically first — not idiomatically. Translate each match branch in the original `filter` function one-for-one before any refactoring. Run all 442 fslit integration tests (not just the 30 offside tests) because IndentFilter bugs surface in module, let, match, and exception tests too.

Define the filter's output as a deterministic function of `(token_stream, config)` and write property tests that check specific known-good token sequences against expected INDENT/DEDENT/IN output.

**Warning signs:**
- Any `implicit-in` integration test failing
- Any `offside` test failing
- Module-level lets getting spurious implicit IN tokens (they should NOT trigger offside)
- `let rec f x =\n    match x with` producing wrong INDENT depth

**Phase to address:** IndentFilter port milestone (before any other integration testing)

---

### Pitfall 4: .funl/.funy Compatibility — Embedded F# Code Blocks

**What goes wrong:**
The .funl and .funy formats embed arbitrary F# code in `{ ... }` header blocks and in semantic actions (`{ $1 }`, `{ Let($2, $4, $6) }`). These code blocks contain F# syntax, not LangThree syntax. A funlex/funyacc parser that tries to parse these blocks as LangThree will fail on valid .funl/.funy files.

More specifically: the Lexer.funl opens with a `{ ... }` block containing F# module opens and helper functions. The Parser.funy opens with a `%{ ... %}` block. These must be treated as opaque strings — extracted and emitted verbatim into the output, not parsed.

**Why it happens:**
It is tempting to parse semantic action blocks to validate or transform them. This is unnecessary for the compatibility goal and breaks immediately on F# syntax that LangThree cannot understand.

**How to avoid:**
Define a clear boundary: funlex/funyacc parses the structural directives (patterns, token names, rule names, precedence) and treats all `{ ... }` action code as opaque lexed strings. The lexer for .funl/.funy must count curly-brace depth to correctly delimit action blocks without parsing their contents. Test specifically with LangThree's actual Lexer.funl (which has complex F# helpers in the header) to verify verbatim passthrough.

**Warning signs:**
- Trying to validate or "understand" semantic action code
- Any attempt to parse `{ ... }` blocks as LangThree expressions
- Failure to handle nested curly braces in action code (e.g., `{ let x = { field = 1 } in x }`)

**Phase to address:** funlex/funyacc parser implementation (early milestone)

---

### Pitfall 5: Grammar Ambiguity Introduced by INDENT/DEDENT Injection

**What goes wrong:**
The existing Parser.funy grammar is written assuming the IndentFilter has already preprocessed the token stream. It contains production rules like:
```
| LET IDENT EQUALS INDENT Expr DEDENT
| LET IDENT EQUALS INDENT Expr DEDENT IN Expr
| MATCH Expr WITH MatchClauses
```
These rely on INDENT/DEDENT being injected at exactly the right positions. If the IndentFilter emits one extra INDENT, or emits a DEDENT at the wrong column boundary, the parser will hit a shift-reduce or reduce-reduce conflict at runtime (not at table-generation time) — it will either silently take the wrong branch or throw a parse error.

This is distinct from a grammar conflict: the grammar itself is unambiguous given correct INDENT/DEDENT placement. The problem is behavioral — the filter's output must precisely match what the grammar expects.

**Why it happens:**
When reimplementing IndentFilter in LangThree, developers test with simple inputs (single let bindings) and miss complex cases. The Parser.funy grammar has 8 different production rules that mix INDENT/DEDENT with other tokens. Each one has a specific token sequence expectation.

**How to avoid:**
Before porting IndentFilter, document each grammar production that consumes INDENT/DEDENT and the exact token sequence it expects. Write unit tests at the token level (input: `LET IDENT EQUALS NEWLINE(4) IDENT`, expected output after filter: `LET IDENT EQUALS INDENT IDENT DEDENT`). These token-level tests are independent of the full parser and catch filter bugs before running 641 end-to-end tests.

**Warning signs:**
- Parse errors on inputs that should parse cleanly
- Wrong AST produced for indented let bodies (missing `in` desugaring)
- `| LET IDENT EQUALS INDENT Expr DEDENT` branch taken when `| LET IDENT EQUALS INDENT Expr DEDENT IN Expr` was intended

**Phase to address:** IndentFilter port + integration testing

---

### Pitfall 6: LALR(1) Cannot Handle the Grammar — Silent Conflicts Resolved Wrong

**What goes wrong:**
Fsyacc resolves shift-reduce conflicts silently using precedence declarations or default resolution (shift preferred). If funyacc does not implement the same conflict resolution strategy, it will produce a different parse table for the same grammar, causing behavioral differences even when there are no "errors." The existing Parser.funy uses `%left`, `%right`, `%nonassoc` declarations for 15+ precedence levels. These are not just annotations — they directly determine which action is chosen in conflict states.

A funyacc that ignores precedence declarations and always shifts will parse arithmetic correctly (shift is right for left-recursive operators) but will fail on `%nonassoc` uses like comparisons (`1 < 2 < 3` should be a parse error but may become `(1 < 2) < 3`).

**Why it happens:**
Precedence-based conflict resolution is a non-trivial post-processing step on the parse table. Implementers who get the LALR table generation right often skip precedence resolution because "it only affects conflicted states" — but those states include critical ones.

**How to avoid:**
Implement precedence and associativity resolution as a required step, not an optional optimization. After generating the raw LALR(1) action table, any remaining shift-reduce conflict must be resolved by comparing the precedence of the production (determined by its rightmost terminal) with the precedence of the lookahead token. Implement `%nonassoc` as "neither shift nor reduce — generate an error action."

Verify by comparing the action table generated by funyacc against the reference fsyacc table for Parser.funy (fsyacc can emit a verbose debug output with `--log`).

**Warning signs:**
- Chained comparison `1 < 2 < 3` not producing a parse error
- Operator precedence tests failing (arithmetic parsing in wrong order)
- Any shift-reduce conflict in the generated table that does NOT appear in fsyacc's output for the same grammar

**Phase to address:** LALR(1) table generation, conflict resolution subphase

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcode IndentFilter config (4 spaces, non-strict) | Simpler implementation | Cannot handle alternative indent widths in .funl/.funy files | Acceptable for Phase 1 if documented |
| Skip LALR table serialization (regenerate from grammar each run) | Avoid file I/O complexity | Generator runs on every LangThree startup (slow) | Never — table must be cached or precomputed |
| Use LangThree's runtime hashtables for LALR goto/action tables | Simple to implement | Table lookup performance degrades for large grammars | Acceptable initially, profile before optimizing |
| Skip conflict warnings, always silently resolve | Faster iteration | Hidden grammar bugs; cannot detect when grammar changes break assumptions | Never — emit warnings even in MVP |
| Copy-paste IndentFilter logic into funyacc output | No abstraction overhead | Two codebases to maintain; module boundary unclear | Never — IndentFilter should be a module |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| .funl regex dialect | Assuming standard POSIX regex — .funl uses its own precedence hierarchy (# for set difference, \* + ? for repetition, concatenation, then |) | Parse .funl regex syntax explicitly; do not use a general regex engine for the .funl meta-format itself |
| .funy semantic actions | Parsing `{ ... }` blocks as LangThree code | Lex action blocks by counting curly brace depth; emit verbatim |
| fsyacc `%type` declarations | Omitting `%type` annotations on non-start nonterminals | `%type` is optional in fsyacc but funyacc should validate that `%start` symbols have `%type` |
| LangThree module system | Generating a flat .fun file when the module system expects `module Lexer = ...` wrapper | Generated files must use LangThree's module declaration syntax |
| NEWLINE token with column payload | Treating NEWLINE as a unit token | `NEWLINE col` carries an integer column; the filter consumes it, the parser never sees it |
| EOF handling | Forgetting to emit pending implicit INs and DEDENTs before EOF | Must flush the context stack on EOF exactly as IndentFilter.fs does |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Building LALR item sets as immutable F# lists | Generator takes minutes on LangThree's grammar | Use sets or sorted arrays; LangThree's grammar has ~400+ items across ~200 states | During development — LangThree Parser.funy is not trivial |
| Re-running closure from scratch for every state | Exponential behavior during state construction | Memoize closure computation; use worklist algorithm | Any grammar with more than ~50 productions |
| Token-stream-as-list in IndentFilter | Materializing entire token list before filtering begins | The IndentFilter.fs already uses list-based lookahead but requires only 1 token lookahead; port should use seq with limited lookahead | Large source files with thousands of tokens |
| Tree-walking evaluator for LALR table construction | funyacc itself is slow because LangThree is interpreted | Acceptable — table generation is a one-time cost per grammar, not per file parsed. Do not prematurely optimize | Only a problem if table construction exceeds ~10 seconds, which requires a very large grammar |

---

## "Looks Done But Isn't" Checklist

- [ ] **IndentFilter: empty lines** — Empty lines (only whitespace/comments) must NOT generate NEWLINE tokens that affect indentation. Verify with tests that have blank lines inside indented blocks.
- [ ] **IndentFilter: single-line match** — `match x with | A -> 1 | B -> 2` (all on one line) must NOT push `InMatch` context. The `JustSawMatch` flag must be consumed by seeing PIPE on the same line before any NEWLINE.
- [ ] **LALR: epsilon productions** — Productions with empty right-hand sides (`|  { [] }`) require correct FIRST/FOLLOW computation. Verify `TypeParams`, `TypeDeclContinuation`, `Decls` (epsilon alternatives) parse correctly.
- [ ] **LALR: reduce-reduce resolution** — If two reductions are possible, funyacc must pick the one that appears first in the grammar file (same as fsyacc default). Document this behavior explicitly.
- [ ] **funlex output: unicode vs byte** — LangThree's existing Lexer.funl uses `LexBuffer<char>` (unicode). Generated funlex output must match — do not silently switch to byte mode.
- [ ] **funyacc output: `$n` references** — Semantic actions in .funy reference `$1`, `$2`, etc. Verify the generated LangThree code maps these to the correct pattern variables in the match expression, accounting for 1-based indexing.
- [ ] **641 tests: all categories** — Do not only run the `expr/` and `file/let/` tests. The `file/module/`, `file/exception/`, `file/offside/`, `file/implicit-in/`, and `file/match/` categories exercise IndentFilter's most complex paths.
- [ ] **Block comments** — The Lexer.funl handles nested `(* ... *)` comments with a depth counter. The generated lexer must replicate this; shallow comment handling fails on `(* (* nested *) *)`.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Lookahead propagation bug discovered after LALR table complete | HIGH | Rewrite closure/propagation; re-run all state construction from scratch |
| IndentFilter rewrite produces wrong token stream | MEDIUM | Roll back to mechanical port; add token-level unit tests before re-attempting idiomatic rewrite |
| Premature cutover broke LangThree pipeline | HIGH | `git checkout` F# reference files; re-run all 641 tests to confirm rollback; do not proceed until side-by-side parity is proven |
| .funl/.funy compatibility break (action code not verbatim) | MEDIUM | Add brace-depth lexer mode; re-scan .funl/.funy with corrected parser |
| Precedence resolution wrong | MEDIUM | Re-implement resolution step; compare against fsyacc verbose log for Parser.funy |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Lookahead propagation bugs | LALR(1) core algorithm | Zero conflicts on Parser.funy; matches fsyacc action table |
| Bootstrapping chicken-and-egg | Architecture design (Phase 1) | Side-by-side test harness exists before any code generation |
| IndentFilter context stack | IndentFilter port | All 30+ offside/implicit-in tests pass at token level |
| .funl/.funy action code verbatim | Parser for .funl/.funy format | Lexer.funl header block emitted verbatim in output |
| INDENT/DEDENT injection mismatch | IndentFilter + integration | All 442 fslit tests pass |
| LALR conflict resolution | LALR table generation | `%nonassoc` comparisons produce parse errors on chained ops |

---

## Sources

- [LALR Parser - Wikipedia](https://en.wikipedia.org/wiki/LALR_parser) — state merging and lookahead propagation mechanics
- [LR Parsing — Rahul Gopinath](https://rahul.gopinath.org/post/2024/07/01/lr-parsing/) — practical LR(0)/SLR/LALR/LR(1) implementation walkthrough
- [Mysterious Conflicts — GNU Bison Manual](https://www.gnu.org/software/bison/manual/html_node/Mysterious-Conflicts.html) — reduce-reduce conflicts from state merging
- [Python Lexical Analysis — Python Docs](https://docs.python.org/3/reference/lexical_analysis.html) — authoritative INDENT/DEDENT specification including parenthesis suspension and empty line rules
- [FsLex Overview](https://fsprojects.github.io/FsLexYacc/content/fslex.html) — .funl format rules, EOF limitation, longest-match behavior
- [FsYacc Documentation](https://github.com/fsprojects/FsLexYacc/blob/master/docs/content/fsyacc.md) — .funy format, `%type` requirements, position tracking requirement
- [Bootstrapping (compilers) — Wikipedia](https://en.wikipedia.org/wiki/Bootstrapping_(compilers)) — staged development and double-compilation verification strategies
- [Principled Parsing for Indentation-Sensitive Languages](https://michaeldadams.org/papers/layout_parsing/LayoutParsing.pdf) — formal treatment of layout rules and edge cases
- LangThree source — `IndentFilter.fs`, `Lexer.funl`, `Parser.funy`, 641 integration tests (direct inspection)

---
*Pitfalls research for: lexer/parser generator bootstrapping (funlex/funyacc for LangThree)*
*Researched: 2026-03-23*
