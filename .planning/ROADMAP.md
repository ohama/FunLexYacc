# Roadmap: FunLexYacc

## Overview

FunLexYacc implements funlex and funyacc — a lexer generator and parser generator — written in LangThree, so that LangThree can generate its own lexer and parser without depending on F# fslex/fsyacc. The journey runs from a safety-first test harness through IndentFilter, through funlex, through funyacc (the algorithmic peak), to a final bootstrap cutover that makes LangThree self-hosting on its lexing and parsing pipeline. The acceptance criterion is 641 passing tests with generated code producing identical output to the F# reference.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Side-by-side test harness and project structure — bootstrapping safety before any generator code
- [ ] **Phase 2: IndentFilter** - Port IndentFilter.fs to IndentFilter.fun — the fragile 7-context state machine, no generator dependencies
- [ ] **Phase 3: funlex** - Full lexer generator: .funl parsing, NFA/DFA construction, and .fun lexer emission
- [ ] **Phase 4: funyacc Input** - .funy format parsing into grammar AST — isolated from LALR algorithm complexity
- [ ] **Phase 5: funyacc LALR Core** - LR(0) items, LALR(1) lookahead propagation, action/goto table construction with conflict resolution
- [ ] **Phase 6: funyacc Output** - Parser code emission: LALR tables + semantic actions + inline shift/reduce interpreter → Parser.fun
- [ ] **Phase 7: Bootstrap** - Run 641 tests against generated pipeline; prove parity; cut over from F# to .fun

## Phase Details

### Phase 1: Foundation
**Goal**: The test harness exists and comparing generated output to F# reference output is a single command
**Depends on**: Nothing (first phase)
**Requirements**: TEST-01, TEST-03
**Success Criteria** (what must be TRUE):
  1. A side-by-side comparison command runs generated lexer/parser output against F# reference output and reports any token or AST difference
  2. The test harness can run each of lexer, IndentFilter, and parser in isolation without the others being complete
  3. A rollback build target exists that restores the F# pipeline if generated output is substituted
  4. Project directory structure matches the intended src/ module layout with clear boundaries between funlex, funyacc, and IndentFilter
**Plans**: 3 plans

Plans:
- [ ] 01-01-PLAN.md — src/ directory skeleton, valid .fun module stubs, tests/fixtures symlink, run.sh
- [ ] 01-02-PLAN.md — compare.sh: single-command side-by-side comparison (tokens + AST, file + --expr mode)
- [ ] 01-03-PLAN.md — run-isolated.sh (per-component passthrough runner) and rollback.sh (F# pipeline restore + baseline verify)

### Phase 2: IndentFilter
**Goal**: IndentFilter.fun correctly transforms NEWLINE(col) tokens into INDENT/DEDENT/IN tokens across all 7 syntax contexts, passing all relevant integration tests
**Depends on**: Phase 1
**Requirements**: IFLT-01, IFLT-02, IFLT-03
**Success Criteria** (what must be TRUE):
  1. IndentFilter.fun produces token-for-token identical output to IndentFilter.fs across all 442 fslit integration tests
  2. All 7 SyntaxContext variants (TopLevel, InMatch, InTry, InLetDecl, InExprBlock, InModule, InFunctionApp) handle offside transitions correctly
  3. JustSawMatch single-line match case handling produces no regressions in single-line-match test cases
  4. IndentFilter.fun compiles and links as a standalone LangThree module consumable by the LangThree module system
**Plans**: TBD

Plans:
- [ ] 02-01: SyntaxContext ADT and NEWLINE→INDENT/DEDENT/IN core transformation
- [ ] 02-02: JustSawMatch handling and all 7 context transitions
- [ ] 02-03: Integration against fslit tests via side-by-side harness

### Phase 3: funlex
**Goal**: funlex accepts any .funl file (including LangThree's Lexer.funl) and emits a correct .fun lexer module with DFA-driven longest-match semantics
**Depends on**: Phase 1
**Requirements**: FLEX-01, FLEX-02, FLEX-03, FLEX-04, FLEX-05, FLEX-06, CGEN-01
**Success Criteria** (what must be TRUE):
  1. funlex parses LangThree's Lexer.funl without error, including verbatim F# action blocks extracted by brace-depth counting without parsing their contents
  2. The generated Lexer.fun tokenizes LangThree source files with token-for-token identical output to the F# Lexer.fs when run through the side-by-side harness
  3. Parameterized rules (block_comment, read_indent, read_string) with extra arguments are emitted correctly in the generated module
  4. The generated .fun module compiles and integrates with LangThree's module system without modification
**Plans**: TBD

Plans:
- [ ] 03-01: FunlParser — recursive descent parser for .funl format, FunlAst types, action block extraction
- [ ] 03-02: NfaBuild — Thompson's construction from regex AST to NFA
- [ ] 03-03: DfaBuild — subset construction (NFA→DFA) and Hopcroft minimization
- [ ] 03-04: LexEmit — serialize DFA as .fun source with transition table and longest-match driver; CGEN-01 module output format

### Phase 4: funyacc Input
**Goal**: funyacc correctly parses LangThree's Parser.funy into a typed grammar AST, handling all token declarations, precedence levels, grammar rules, and verbatim semantic action blocks
**Depends on**: Phase 1
**Requirements**: FYAC-01
**Success Criteria** (what must be TRUE):
  1. FunyParser.fun parses Parser.funy without error and produces a typed grammar AST reflecting all ~40 token declarations, 10+ precedence levels, multiple start symbols, and all grammar productions
  2. Embedded F# semantic action blocks are extracted verbatim by brace-depth counting without the parser attempting to interpret their contents
  3. The grammar AST accurately represents epsilon productions, multiple start symbols, and all %left/%right/%nonassoc declarations
**Plans**: TBD

Plans:
- [ ] 04-01: FunyAst type definitions and FunyParser recursive descent implementation
- [ ] 04-02: Precedence, start symbol, and type annotation handling; AST round-trip validation against Parser.funy

### Phase 5: funyacc LALR Core
**Goal**: The LALR(1) table builder generates correct action/goto tables for LangThree's grammar with zero conflicts, using DeRemer/Pennello lookahead propagation and full precedence resolution
**Depends on**: Phase 4
**Requirements**: FYAC-02, FYAC-03, FYAC-04
**Success Criteria** (what must be TRUE):
  1. LR(0) item set construction correctly computes closures including epsilon reachability for all nullable nonterminals in Parser.funy
  2. LALR(1) lookahead propagation produces tables that match fsyacc's output — zero spurious reduce/reduce conflicts on Parser.funy
  3. %left/%right/%nonassoc conflict resolution produces correct action table entries; %nonassoc produces error actions (not silent acceptance), matching fsyacc's behavior
  4. Both start symbols (start: Ast.Expr and parseModule: Ast.Module) are represented in the goto table with correct initial states
**Plans**: TBD

Plans:
- [ ] 05-01: LrItems — LR(0) item set construction, closure with epsilon reachability, goto function
- [ ] 05-02: Lookahead — DeRemer/Pennello LALR(1) lookahead propagation
- [ ] 05-03: Tables — action/goto table construction, conflict resolution, %nonassoc error actions, multiple start symbol support

### Phase 6: funyacc Output
**Goal**: funyacc emits a Parser.fun that embeds the LALR tables, all semantic actions verbatim, and a complete inline shift/reduce interpreter, producing a self-contained LangThree parser module
**Depends on**: Phase 5
**Requirements**: FYAC-05, CGEN-02, CGEN-03
**Success Criteria** (what must be TRUE):
  1. The generated Parser.fun compiles as a LangThree module without modification
  2. Semantic action blocks from Parser.funy ($1, $2, $N references; parseState / IParseState; ruleSpan and symSpan helpers) are embedded correctly and compile without errors
  3. The inline shift/reduce interpreter in Parser.fun correctly drives the LALR tables through a stack-based parse without runtime dependency on funyacc
  4. Parser.fun produces AST output for at least one representative LangThree source file that can be compared to the F# parser's output
**Plans**: TBD

Plans:
- [ ] 06-01: ParserEmit — LALR table serialization and $N / parseState semantic action embedding
- [ ] 06-02: Inline shift/reduce interpreter emission (CGEN-03) and end-to-end integration test

### Phase 7: Bootstrap
**Goal**: LangThree's full pipeline (Lexer.fun + IndentFilter.fun + Parser.fun) passes all 641 tests with output identical to the F# reference, and the F# fslex/fsyacc dependency is removed
**Depends on**: Phases 2, 3, 6
**Requirements**: BOOT-01, BOOT-02, BOOT-03, TEST-02
**Success Criteria** (what must be TRUE):
  1. funlex processes Lexer.funl and funyacc processes Parser.funy without errors, producing Lexer.fun and Parser.fun
  2. All 641 tests (199 F# unit + 442 fslit integration) pass when run against the generated Lexer.fun + IndentFilter.fun + Parser.fun pipeline
  3. The generated pipeline produces byte-identical ASTs to the F# reference for every test input (verified by side-by-side harness from Phase 1)
  4. The LangThree build removes the FsLexYacc NuGet dependency and compiles without Lexer.fs/Parser.fs
**Plans**: TBD

Plans:
- [ ] 07-01: End-to-end run: funlex on Lexer.funl, funyacc on Parser.funy, side-by-side 641-test validation
- [ ] 07-02: Cutover — replace Lexer.fs/Parser.fs with .fun equivalents, remove FsLexYacc dependency, confirm 641 passing

## Progress

**Execution Order:**
Phases execute in dependency order: 1 → 2 → 3 → 4 → 5 → 6 → 7
(Phases 2 and 3 both depend on Phase 1 only and can run in parallel)

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/3 | Not started | - |
| 2. IndentFilter | 0/3 | Not started | - |
| 3. funlex | 0/4 | Not started | - |
| 4. funyacc Input | 0/2 | Not started | - |
| 5. funyacc LALR Core | 0/3 | Not started | - |
| 6. funyacc Output | 0/2 | Not started | - |
| 7. Bootstrap | 0/2 | Not started | - |
