# Project Research Summary

**Project:** FunLexYacc — funlex/funyacc lexer/parser generators for LangThree
**Domain:** Compiler tooling — bootstrapping a functional language's own lexer/parser generators
**Researched:** 2026-03-23
**Confidence:** HIGH

## Executive Summary

FunLexYacc is a bootstrapping project: write funlex (a lexer generator) and funyacc (a parser generator) in LangThree (.fun), then use them to regenerate LangThree's own Lexer.fun and Parser.fun, replacing the existing F# implementations. This is the classic staged bootstrapping pattern used by Alex/Happy, ocamllex/ocamlyacc, and Menhir. The key constraint is strict format compatibility — the tools must consume LangThree's existing Lexer.fsl and Parser.fsy files without modification and produce output that passes all 641 existing integration tests.

The algorithm choices are well-established and non-negotiable: Thompson's NFA construction followed by subset construction and Hopcroft minimization for funlex; LALR(1) via DeRemer/Pennello lookahead propagation for funyacc. The implementation language (LangThree) imposes no unusual constraints — the standard library provides Maps, Sets, and list operations sufficient for all required data structures. No external dependencies are needed. The highest-confidence reference implementation for format compatibility is FsLexYacc itself; for clean LALR(1) implementation, Lemon (SQLite's parser generator) is ~5,000 lines of well-commented C.

The dominant risks are algorithmic correctness (LALR(1) lookahead propagation bugs produce silent wrong-behavior, not crashes), IndentFilter fidelity (7-context state machine with intricate edge cases), and bootstrapping discipline (cutting over to generated output before 100% test parity destroys the test harness itself). All three risks have well-defined mitigations: test against LangThree's actual grammar from day one, port IndentFilter mechanically before refactoring, and maintain a strict side-by-side comparison phase before any cutover.

## Key Findings

### Recommended Stack

Both tools are pure algorithmic implementations in LangThree with zero external dependencies. The algorithm pipeline for funlex is: parse .fsl → Thompson's NFA → subset construction (NFA→DFA) → Hopcroft minimization → emit DFA transition table as .fun source. The pipeline for funyacc is: parse .fsy → compute FIRST sets → construct LR(0) item sets → LALR(1) lookahead propagation (DeRemer/Pennello) → build action/goto tables → resolve conflicts via %left/%right/%nonassoc → emit .fun source. Output format is always LangThree source text, never binary tables.

The bootstrap phase is managed with a hand-written recursive descent parser for .fsl/.fsy during Phase 0; once funlex/funyacc can process their own grammars, the hand-written parsers are replaced. This is the exact strategy used by Alex (which ships pre-generated Parser.hs) and Happy (which uses a parser-combinator bootstrap).

**Core technologies:**
- LALR(1) via DeRemer/Pennello (1982): parser table generation — matches fsyacc's algorithm exactly, sufficient for LangThree's grammar
- Thompson's NFA construction: regex-to-NFA — compositional, maps cleanly to recursive functional implementation
- Powerset/subset construction: NFA-to-DFA — standard, tractable for token-level regexes
- Hopcroft's partition refinement: DFA minimization — O(n log n), reduces table size, optional for v1
- LangThree standard Map/Set: all set-based computations — built-in, no external deps
- .fsl/.fsy input format: FsLexYacc-compatible — non-negotiable constraint
- .fun module output: LangThree module system — non-negotiable constraint

### Expected Features

Features are driven entirely by what LangThree's actual Lexer.fsl and Parser.fsy exercise. Everything in the table-stakes list is directly observed in those files. The indentation system is handled by a separate IndentFilter pipeline stage (not by funlex/funyacc), which transforms NEWLINE(col) tokens into INDENT/DEDENT tokens.

**Must have (table stakes for bootstrapping):**

funlex:
- Named regex definitions (`let ident = regexp`) — used throughout Lexer.fsl
- Character class literals, ranges, union, negation — foundational
- Quantifiers `*`, `+`, `?` — used in every named regex
- String and character literal patterns — all keywords and operators
- `eof` special pattern — terminal rule requirement
- Multiple named rules via `and` — four separate rules in Lexer.fsl
- Parameterized rules (state threading via extra args) — block_comment, read_indent, read_string
- Longest-match + first-match-wins semantics — critical for multi-char operator disambiguation
- `lexbuf` implicit last argument and `LexBuffer<_>` position mutation — line tracking

funyacc:
- `%token` (no payload) and `%token <type>` (with payload) declarations — ~40 tokens
- `%start` and `%type` with multiple start symbols — `start` and `parseModule`
- `%left`, `%right`, `%nonassoc` precedence declarations (10+ levels) — operator precedence
- Grammar rules with `$1`-`$N` semantic value access — all productions
- `parseState` / `IParseState` in actions (source span tracking) — ruleSpan, symSpan helpers
- LALR(1) table generation with conflict resolution via precedence — core requirement
- Empty productions (epsilon rules) — TypeParams, TypeDeclContinuation, etc.
- INDENT/DEDENT as normal tokens in grammar rules — 62+ occurrences in Parser.fsy

**Should have (P2 — add after bootstrapping validation):**
- Conflict reporting (`-v` verbose output with state/token info) — grammar debugging
- `%prec TOKEN` rule-level precedence override — resolves dangling-else style conflicts

**Defer (v2+):**
- Error recovery (`error` token) — LangThree has no error productions
- Unicode mode (`--unicode`) — LangThree only needs ASCII currently
- Verbose conflict output with .dot graph — useful but not blocking
- Binary table serialization — premature optimization

### Architecture Approach

The architecture follows the canonical two-phase generator pattern: parse spec → build automaton/tables → emit source. Both funlex and funyacc are file-in/file-out CLI tools. The generated .fun modules are self-contained and have no runtime dependency on the generator. IndentFilter is a standalone port of the existing F# module — it is not generated, it is shipped as handwritten .fun code alongside the generators.

The recommended build order is driven by dependency: IndentFilter first (no deps, testable immediately), then funlex (FslParser → NFA → DFA → LexEmit), then funyacc (FsyParser → LrItems → Lookahead → Tables → ParserEmit), then end-to-end bootstrap validation.

**Major components:**
1. FslParser — parse .fsl spec into typed AST (hand-written recursive descent)
2. NfaBuild + DfaBuild — Thompson's construction then subset/Hopcroft (core lexer algorithm)
3. LexEmit — serialize DFA as .fun source (template generation, one state per line)
4. FsyParser — parse .fsy spec into grammar AST (hand-written recursive descent)
5. LrItems + Lookahead + Tables — LR(0) items, LALR(1) lookahead propagation, action/goto tables with conflict resolution
6. ParserEmit — serialize tables + semantic actions as .fun source
7. IndentFilter.fun — port of IndentFilter.fs; NEWLINE(col) → INDENT/DEDENT/IN stateful fold

### Critical Pitfalls

1. **LALR(1) lookahead propagation bugs** — Forgetting epsilon reachability in closure produces spurious reduce-reduce conflicts that look like grammar problems. Prevention: implement closure with `FIRST(beta LA)` not just `FIRST(beta)`, test all LangThree nullability cases. Verification: zero conflicts on Parser.fsy.

2. **Premature bootstrapping cutover** — Copying generated files into the LangThree pipeline before 100% test parity destroys the test harness needed to debug. Prevention: mandatory side-by-side comparison phase before any cutover; never proceed below 641 passing tests.

3. **IndentFilter context stack reconstruction** — The 7-context stack (TopLevel, InMatch, InTry, InFunctionApp, InLetDecl, InExprBlock, InModule) has intricate edge cases. Prevention: port mechanically one-for-one before any refactoring; write token-level unit tests for each grammar production consuming INDENT/DEDENT.

4. **Embedded F# action code verbatim passthrough** — `{ ... }` blocks in .fsl/.fsy contain F# syntax, not LangThree. Prevention: lex action blocks by counting brace depth, never parse contents. Test with Lexer.fsl's complex header block.

5. **LALR conflict resolution incompleteness** — Implementing LALR table generation without precedence resolution means `%nonassoc` comparisons silently parse wrong. Prevention: implement precedence resolution as required step, verify `%nonassoc` produces error actions, compare against fsyacc verbose output.

## Implications for Roadmap

Based on research, the dependency graph drives a clear 9-phase structure. The architecture research explicitly lays out the build order; the pitfalls research identifies which phases are highest-risk and need the most verification.

### Phase 1: Foundation and Test Infrastructure
**Rationale:** Before writing any generator code, establish the side-by-side comparison harness that makes bootstrapping safe. This addresses the highest-severity pitfall (premature cutover) before it can occur. Also set up the project structure with module boundaries per ARCHITECTURE.md.
**Delivers:** Git repo structure matching src/ layout; test harness that runs generated output against all 641 tests and compares against F# reference; rollback build target.
**Addresses:** Bootstrapping safety (Pitfall 2)
**Avoids:** The catastrophic scenario where broken generated code destroys the test harness

### Phase 2: IndentFilter Port
**Rationale:** IndentFilter has no dependencies on funlex or funyacc output. It is the most fragile component (7-context state machine) and is exercised by 442 of 641 integration tests. Completing it first gives a solid testing foundation before any generator work begins.
**Delivers:** IndentFilter.fun that passes all 442 fslit integration tests at token level, including all offside and implicit-in edge cases
**Implements:** Architecture component: IndentFilter (Pattern 4: pure stream transformer)
**Avoids:** IndentFilter context stack reconstruction bugs (Pitfall 3); INDENT/DEDENT injection mismatch (Pitfall 5)

### Phase 3: funlex Input Parsing (FslParser)
**Rationale:** FslParser depends only on LangThree itself. It produces the typed FslAst that all downstream NFA/DFA work depends on. Starting here allows validating .fsl format handling (including verbatim action block passthrough) before writing algorithmic code.
**Delivers:** FslParser.fun that parses Lexer.fsl and prints a correct typed AST; FslAst.fun module
**Uses:** Hand-written recursive descent (no external dependencies)
**Avoids:** Embedded F# action code parsing (Pitfall 4)

### Phase 4: funlex Automaton (NFA + DFA)
**Rationale:** With FslParser complete, the NFA and DFA builders can be developed and tested in isolation against small known-correct regex inputs before touching LangThree's actual Lexer.fsl.
**Delivers:** NfaBuild.fun (Thompson's construction) + DfaBuild.fun (subset construction + optional Hopcroft minimization)
**Uses:** LALR(1) algorithm pipeline for lexer: regex AST → NFA → DFA → transition table
**Implements:** Architecture components: NFA Builder, DFA Builder (Patterns 1 and 2)

### Phase 5: funlex Output (LexEmit + Integration)
**Rationale:** With the DFA complete, LexEmit serializes it to Lexer.fun. This is the first integration test: run generated Lexer.fun on LangThree source in side-by-side mode.
**Delivers:** Working Lexer.fun that tokenizes LangThree source identically to Lexer.fs; LexEmit.fun module
**Addresses:** Longest-match + first-match semantics; NEWLINE(col) with column payload; LexBuffer position mutation
**Implements:** Architecture component: Lexer Code Emitter (Pattern 2: DFA-as-data, emitter-as-template)

### Phase 6: funyacc Input Parsing (FsyParser)
**Rationale:** Mirrors Phase 3 for the parser side. FsyParser depends only on LangThree. Completing it separately allows validating .fsy format handling before writing the complex LALR algorithm.
**Delivers:** FsyParser.fun that parses Parser.fsy and prints a correct grammar AST; FsyAst.fun module
**Uses:** Hand-written recursive descent; brace-depth action block extraction
**Avoids:** Embedded F# action code parsing (Pitfall 4); grammar normalization skipping (Architecture Anti-Pattern 5)

### Phase 7: funyacc LALR(1) Core (LrItems + Lookahead + Tables)
**Rationale:** This is the highest-algorithmic-complexity phase. Must be developed with continuous verification against Parser.fsy — any deviation from fsyacc's table produces behavioral differences. Develop LrItems, Lookahead, and Tables as separately testable sub-components per the architecture's module boundary design.
**Delivers:** LrItems.fun, Lookahead.fun, Tables.fun with correct LALR(1) action/goto tables for LangThree's grammar; zero conflicts on Parser.fsy; precedence resolution for %left/%right/%nonassoc including correct %nonassoc semantics
**Uses:** DeRemer/Pennello (1982) LALR(1) lookahead propagation
**Avoids:** Lookahead propagation bugs (Pitfall 1); silent conflict resolution (Pitfall 6)

### Phase 8: funyacc Output (ParserEmit + Integration)
**Rationale:** With correct LALR tables, ParserEmit serializes them plus semantic actions to Parser.fun. This is the critical integration test: run generated Parser.fun against all 641 tests in side-by-side mode.
**Delivers:** Working Parser.fun that parses LangThree source identically to Parser.fs; ParserEmit.fun module; 100% side-by-side test parity
**Implements:** Architecture component: Parser Code Emitter (Pattern 3: LALR table as encoded arrays)
**Addresses:** All funyacc table-stakes features including multiple start symbols, $N references, parseState access

### Phase 9: Bootstrap Cutover + Validation
**Rationale:** Only after 100% side-by-side parity is proven does the cutover occur. Replace Lexer.fs/Parser.fs with Lexer.fun/Parser.fun in LangThree pipeline. Run all 641 tests to confirm behavioral equivalence. This completes the bootstrapping goal.
**Delivers:** Self-hosting LangThree — its own lexer and parser are generated by tools written in LangThree itself
**Avoids:** Bootstrapping chicken-and-egg catastrophe (Pitfall 2) — side-by-side parity was required in Phase 8 before this phase runs

### Phase Ordering Rationale

- IndentFilter before generators: zero dependencies, highest fragility, exercises the most tests — de-risk it first
- FslParser before NFA/DFA: clean boundaries; parsing bugs are easier to fix than algorithmic bugs mixed together
- funlex before funyacc: lexer algorithm (Thompson's + subset) is simpler than LALR; getting end-to-end lexer output validates the code generation infrastructure before the harder parser phase
- FsyParser before LALR core: same isolation principle as funlex; grammar bugs surface before table-construction bugs
- Tables split into LrItems + Lookahead + Tables: each is independently testable on toy grammars before running against Parser.fsy
- Cutover only after 100% parity: mandated by Pitfall 2; no exceptions

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 7 (LALR core):** DeRemer/Pennello lookahead propagation has subtle epsilon-reachability cases. Recommend reading Lemon source (lemon.c) and the Dragon Book sections on LALR construction before implementation begins. Comparing generated table against fsyacc's `--log` output is essential.
- **Phase 2 (IndentFilter):** The 7-context state machine has edge cases not visible in the source code. Recommend mapping every context transition against all 30+ offside/implicit-in test cases before writing code.

Phases with standard patterns (deeper research not needed):
- **Phase 3 and Phase 6 (FslParser, FsyParser):** Hand-written recursive descent for simple DSLs is well-documented. FsLexYacc docs provide complete format specification.
- **Phase 4 (NFA/DFA):** Thompson's construction and subset construction are textbook algorithms with multiple reference implementations (Alex, ocamllex). No research phase needed.
- **Phase 5 and Phase 8 (LexEmit, ParserEmit):** Code generation is template string construction. No research needed beyond studying the structure of existing generated Parser.fs.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All algorithm choices verified against FsLexYacc docs, DeRemer/Pennello paper, and multiple reference implementations. Zero ambiguity. |
| Features | HIGH | Based on direct inspection of Lexer.fsl, Parser.fsy, and IndentFilter.fs — not documentation inference. Feature list is ground-truth. |
| Architecture | HIGH | Component boundaries and build order derived from actual dependency graph. Matches established patterns in FsLexYacc, Alex, Happy. |
| Pitfalls | HIGH | Pitfalls derived from known FsLexYacc bugs (issue #39), LALR algorithm known failure modes, and direct analysis of IndentFilter's complexity. Not speculation. |

**Overall confidence:** HIGH

### Gaps to Address

- **fsyacc `%type` on non-start nonterminals:** FsLexYacc documentation is incomplete on whether `%type` is required for all nonterminals or only start symbols. Research noted this as MEDIUM confidence. During FsyParser implementation, validate against fsyacc's actual behavior with Parser.fsy.
- **LangThree v1.8 module syntax details:** Generated .fun files must use correct LangThree module declaration syntax. If module syntax has evolved, generated output may not compile. Verify against current LangThree source before writing LexEmit/ParserEmit.
- **DFA state explosion in Hopcroft minimization:** For v1, Hopcroft minimization is optional. If the unminimized DFA for Lexer.fsl exceeds a tractable state count, equivalence classes (grouping chars with identical transitions) must be added. This is unlikely given the grammar size but should be monitored.

## Sources

### Primary (HIGH confidence)
- LangThree/src/LangThree/Lexer.fsl — direct inspection, all funlex feature requirements
- LangThree/src/LangThree/Parser.fsy — direct inspection, all funyacc feature requirements
- LangThree/src/LangThree/IndentFilter.fs — direct inspection, IndentFilter architecture
- LangThree/src/LangThree/Parser.fs — direct inspection, generated parser structure reference
- [FsLexYacc GitHub](https://github.com/fsprojects/FsLexYacc) — format specification
- [FsLexYacc issue #39](https://github.com/fsprojects/FsLexYacc/issues/39) — confirmed %nonassoc bug
- [DeRemer & Pennello 1982](https://www.semanticscholar.org/paper/Efficient-Computation-of-LALR(1)-Look-Ahead-Sets-DeRemer-Pennello/4337e7504e0d43a0c21802d5301fdbb1c3950d0f) — LALR(1) lookahead propagation algorithm
- [LALR parser Wikipedia](https://en.wikipedia.org/wiki/LALR_parser) — algorithm overview
- [OCamllex manual](https://ocaml.org/manual/5.4/lexyacc.html) — lexer generator design reference
- [Menhir project](https://gallium.inria.fr/~fpottier/menhir/) — LR(1) reference

### Secondary (MEDIUM confidence)
- [Happy GitHub](https://github.com/haskell/happy) — bootstrap parser-combinator strategy (inferred from search results)
- [FsYacc docs](https://fsprojects.github.io/FsLexYacc/content/fsyacc.html) — incomplete documentation; some details inferred
- [Self-hosted parser design — Drew DeVault](https://drewdevault.com/2021/04/22/Our-self-hosted-parser-design.html) — bootstrapping lessons
- [Lemon parser generator](https://sqlite.org/lemon.html) — clean LALR(1) reference implementation

### Tertiary (LOW confidence)
- [Derw bootstrapping post](https://derw.substack.com/p/writing-a-bootstrapping-compiler) — general bootstrapping example, not specific to this domain

---
*Research completed: 2026-03-23*
*Ready for roadmap: yes*
