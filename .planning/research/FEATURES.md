# Feature Research

**Domain:** Lexer/parser generator (fslex/fsyacc-compatible, targeting bootstrapping of LangThree)
**Researched:** 2026-03-23
**Confidence:** HIGH — based on direct inspection of LangThree/src/LangThree/Lexer.fsl, Parser.fsy, and IndentFilter.fs, plus FsLexYacc official docs

---

## Scope Note

This research answers two interleaved questions:

1. What do fslex/fsyacc actually support? (Establishes compatibility baseline)
2. What does LangThree's grammar actually use? (Establishes what funlex/funyacc must implement for bootstrapping)

The bootstrapping constraint is the hard constraint. "Table stakes" in this context means "bootstrapping fails without it", not "users expect it in general".

---

## Feature Landscape

### Table Stakes (Bootstrapping Fails Without These)

These are features directly observed in Lexer.fsl, Parser.fsy, or IndentFilter.fs that LangThree requires.

#### funlex Table Stakes

| Feature | Why Required | Complexity | Evidence |
|---------|--------------|------------|----------|
| Header block `{ ... }` | Opens F# modules, defines helper functions (`lexeme`, `classifyOperator`, `setInitialPos`) | LOW | Lexer.fsl lines 1–33 |
| Named regex definitions (`let ident = regexp`) | `digit`, `whitespace`, `newline`, `letter`, `ident_start`, `ident_char`, `type_var`, `op_char` — all used | LOW | Lexer.fsl lines 36–43 |
| Character class literals `['a'-'z']`, `[' ' '\t']` | Used in every named regex | LOW | Lexer.fsl lines 36–43 |
| Character class union (space-separated inside `[...]`) | `['a'-'z' 'A'-'Z']`, `[' ' '\t']` | LOW | Lexer.fsl lines 40, 41 |
| Character class negation `[^ ...]` | `[^ '\n' '\r']*` in single-line comment rule | LOW | Lexer.fsl line 114 |
| Quantifiers `*`, `+`, `?` | `digit+`, `ident_char*`, `' '+ ` | LOW | Lexer.fsl lines 36–44, 47 |
| Alternation `|` inside rule arm | Multiple keyword/operator alternatives | LOW | Lexer.fsl throughout |
| Concatenation and grouping `(...)` | `('\n' \| '\r' '\n')` for newline | LOW | Lexer.fsl line 38 |
| String literal patterns `"let"`, `"->"` | All keywords and multi-char operators | LOW | Lexer.fsl lines 53–144 |
| Character literal patterns `'+'`, `'\n'` | All single-char operators and escape chars | LOW | Lexer.fsl lines 117–144 |
| `eof` special pattern | Terminal rule `\| eof { EOF }` | LOW | Lexer.fsl line 145 |
| Multiple named rules via `and` | `tokenize`, `block_comment`, `read_indent`, `read_string` — four rules | MEDIUM | Lexer.fsl lines 46, 148, 158, 163 |
| Parameterized rules | `block_comment depth`, `read_indent col`, `read_string buf` — pass state between rules | MEDIUM | Lexer.fsl lines 148, 158, 164 |
| `lexbuf` implicit argument in actions | `lexeme lexbuf`, `lexbuf.EndPos <- ...`, `lexbuf.EndPos.NextLine` | LOW | Lexer.fsl throughout |
| Actions as arbitrary F# expressions | `failwith`, `Int32.Parse`, string building, recursive calls | LOW | Lexer.fsl throughout |
| `LexBuffer<_>` position mutation (`EndPos <- EndPos.NextLine`) | Required for correct line tracking across newlines | MEDIUM | Lexer.fsl lines 49, 152 |
| Mutual recursion between rules | `tokenize` calls `block_comment`, `read_indent`, `read_string`; each returns to `tokenize` | MEDIUM | Lexer.fsl design |
| Longest-match semantics | Multi-char ops (`|>`, `>>`, `<<`, `<-`, `..`) must beat single-char ops | HIGH | Lexer.fsl lines 97–144 (ordering matters) |
| First-match-wins ordering | Keywords before `ident_start ident_char*`; `_` before identifier | MEDIUM | Lexer.fsl lines 53–93 |

#### funyacc Table Stakes

| Feature | Why Required | Complexity | Evidence |
|---------|--------------|------------|----------|
| Header block `%{ ... %}` | Opens `Ast`, `FSharp.Text.Lexing`, `FSharp.Text.Parsing`; defines helpers `ruleSpan`, `symSpan`, `desugarAnnotParams` | LOW | Parser.fsy lines 1–20 |
| `%token` declarations (no payload) | ~40 keyword/punctuation tokens | LOW | Parser.fsy lines 26–67 |
| `%token <type>` declarations (with payload) | `NUMBER` (int), `IDENT` (string), `STRING` (string), `TYPE_VAR` (string), `NEWLINE` (int), `INFIXOP0-4` (string) | LOW | Parser.fsy lines 23–25, 62–64 |
| `%start` and `%type` declarations | Two start symbols: `start` and `parseModule` | LOW | Parser.fsy lines 85–88 |
| Multiple start symbols | `start : Ast.Expr` and `parseModule : Ast.Module` — both used in production | MEDIUM | Parser.fsy lines 85–88 |
| Precedence declarations `%left`, `%right`, `%nonassoc` | 10 precedence levels: PIPE_RIGHT, COMPOSE_RIGHT, COMPOSE_LEFT, OR, AND, comparison set, INFIXOP0, INFIXOP1, CONS, INFIXOP2, INFIXOP3, INFIXOP4 | MEDIUM | Parser.fsy lines 71–82 |
| Action blocks `{ ... }` with F# expressions | All grammar rules produce AST nodes | LOW | Parser.fsy throughout |
| `$1`, `$2`, ... semantic value access | Standard yacc convention | LOW | Parser.fsy throughout |
| `parseState` access in actions | `ruleSpan parseState 1 4` — accesses `IParseState` | MEDIUM | Parser.fsy lines 8–9 |
| `InputStartPosition` / `InputEndPosition` | Source span construction for AST nodes | MEDIUM | Parser.fsy lines 8–9 |
| Left-recursive rules | `Term STAR Factor`, `AppExpr Atom`, `Term STAR TupleTypeList` | MEDIUM | Standard LALR requirement |
| Right-recursive rules | `ExprList`, `SemiExprList`, `PatternList`, `ParamList`, `Constructors` | LOW | Parser.fsy lines 279–294 |
| LALR(1) table generation | The grammar has real shift/reduce situations resolved by precedence | HIGH | Core requirement |
| Operator precedence via `%left`/`%right` on token classes | `INFIXOP0`–`INFIXOP4` each occupy their own precedence level | MEDIUM | Parser.fsy lines 77–82 |
| `%%` section separator | Standard fsyacc format | LOW | Parser.fsy line 90 |
| Empty productions (epsilon rules) | `TypeParams :` (empty), `TypeDeclContinuation :` (empty), `LetRecContinuation :` (empty) | LOW | Parser.fsy lines 405, 426, 600+ |
| Named nonterminals reused across rules | Grammar has ~30 nonterminals; shared across declaration and expression contexts | LOW | Standard |
| Mutually recursive grammar rules | `Expr → AppExpr → Atom → Expr` (parenthesized); pattern rules reference each other | LOW | Standard LALR |
| `INDENT`/`DEDENT` tokens consumed in grammar | Rules explicitly pattern-match on `INDENT ... DEDENT` blocks | HIGH | Parser.fsy lines 105–107, 150, 227–229, etc. |

#### IndentFilter (funlex pipeline component, not part of .fsl format)

The IndentFilter is a separate F# module that post-processes the raw token stream. It is NOT generated by funlex — it is handwritten F# code that consumes `Parser.token` values. However, funlex must produce the `NEWLINE col` token (a NEWLINE with an integer payload carrying the column position) that IndentFilter uses as input.

| Feature | Why Required | Complexity | Evidence |
|---------|--------------|------------|----------|
| `NEWLINE <int>` token with payload | IndentFilter reads column position from NEWLINE | LOW | Lexer.fsl line 161: `NEWLINE col` |
| `INDENT` and `DEDENT` tokens declared but not emitted by lexer | Emitted by IndentFilter, consumed by Parser | LOW | Parser.fsy lines 65–66 |
| Tab-rejection in lexer | `'\t' { failwith "Tab character not allowed" }` — tabs break indent counting | LOW | Lexer.fsl lines 48, 160 |
| Spaces-only indentation model | `read_indent` counts spaces with `' ' { read_indent (col + 1) }` then emits `NEWLINE col` | LOW | Lexer.fsl lines 158–161 |

---

### Differentiators (Nice to Have, Not Required for Bootstrapping)

Features that existing fslex/fsyacc support and that make funlex/funyacc more usable, but that LangThree does not currently exercise.

#### funlex Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Unicode mode (`--unicode`) | Non-ASCII identifiers, emoji in source, internationalization | HIGH | fslex supports via `--unicode` flag; LangThree only needs ASCII |
| Character set difference (`regexp1 # regexp2`) | Expressive negation beyond `[^...]` | MEDIUM | fslex feature; LangThree doesn't use it |
| Named regex validation (cycle detection) | Catch `let a = b` and `let b = a` compile-time | MEDIUM | Not required but prevents confusing errors |
| Source-map output | Map generated F# back to .fsl line numbers for debugging | HIGH | None of the standard tools do this well |
| Error on overlapping rules warning | Warn when a rule arm can never be reached | MEDIUM | Useful DX but not blocking |
| `?` optional quantifier | LangThree doesn't use it, but common in other lexers | LOW | Already in fslex spec |

#### funyacc Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| `%prec IDENT` rule-level precedence override | Resolves dangling-else and similar conflicts without grammar duplication | MEDIUM | LangThree resolves via grammar structure but this is standard yacc |
| Conflict reporting (`-v` verbose output) | Shows shift/reduce and reduce/reduce conflicts with state info | MEDIUM | Critical for grammar debugging |
| Parser table serialization | Pre-generate tables into binary for fast startup | HIGH | Not needed for bootstrapping |
| GLR extension | Handle ambiguous grammars | VERY HIGH | LangThree is deliberately LALR(1); do not build this |
| Error production (`error` token) | Panic-mode error recovery | HIGH | LangThree has none; don't prioritize |
| `--module` / `--internal` flags | Control generated module name and visibility | LOW | Useful for clean output but not blocking |

---

### Anti-Features (Deliberately Do NOT Build)

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| GLR / Earley parsing | LangThree grammar is LALR(1) by design; GLR adds complexity with no benefit and makes bootstrapping harder to reason about | Stick to LALR(1); report conflicts and fix grammar |
| Error recovery (`error` token) | Requires significant grammar annotations LangThree doesn't have; creates confusing partial-parse states | Emit a clear error message and stop; panic-mode recovery is a v2+ feature |
| Full yacc C-style compatibility (`yylval`, `YYSTYPE`) | The target is F# output, not C; emulating C conventions creates translation friction | Generate idiomatic F# discriminated unions and pattern matching |
| Interactive REPL integration in the generator | Generators are build-time tools; runtime REPL concerns belong in LangThree's evaluator | Keep generator as a pure file-in → file-out CLI tool |
| Semantic predicates / context-sensitive lexing via attributes | LangThree handles context via IndentFilter post-processing, not lexer state predicates | Preserve the IndentFilter pattern: raw lexer emits NEWLINE col, filter transforms to INDENT/DEDENT |
| Separate lexer/parser binary format (`.tables` files) | Premature optimization; LangThree is a bootstrapping compiler, not a production parser with hot startup requirements | Generate F# source directly; keep it simple |
| Auto-generated AST types | LangThree has a carefully crafted `Ast.fs`; generating AST types creates a conflict | The parser actions populate user-defined `Ast` types; funyacc generates only the parser machinery |

---

## Feature Dependencies

```
[Named regex definitions]
    └──requires──> [Named regex reference in rules]

[Multiple rules via `and`]
    └──requires──> [Parameterized rules]  (rules without params work but are less useful)
    └──enables──> [Recursive lexing contexts] (block_comment, read_string)

[`%token <type>` declarations]
    └──requires──> [Multiple start symbols / %type declarations]

[Multiple start symbols]
    └──requires──> [LALR table generation handles multiple entry points]

[Precedence declarations (%left/%right/%nonassoc)]
    └──requires──> [LALR conflict detection] (precedence only resolves known conflicts)
    └──enables──> [INFIXOP0-4 user-defined operators at distinct levels]

[INDENT/DEDENT token consumption in grammar]
    └──requires──> [IndentFilter post-processor] (external to funlex/funyacc)
    └──requires──> [NEWLINE <int> token emission in lexer]

[`parseState` / `IParseState` in actions]
    └──requires──> [funyacc generates parseState parameter in action wrapper]
    └──enables──> [Source span tracking (ruleSpan, symSpan)]

[Longest-match semantics]
    └──conflicts──> [First-match-wins ordering]
    (Resolution: standard lex semantics = longest match wins; ties broken by first rule.
     LangThree relies on this: `|>` beats `|` because longer match wins.)
```

### Dependency Notes

- **`INDENT`/`DEDENT` requires IndentFilter**: The parser grammar consumes `INDENT`/`DEDENT` tokens directly (62+ occurrences in Parser.fsy). These tokens are NOT emitted by the funlex-generated lexer — they come from IndentFilter. funlex's only contribution to indentation is the `NEWLINE col` token. This separation must be preserved.

- **Multiple start symbols require coordinated `%start`/`%type`**: LangThree uses `start` for expression-mode parsing and `parseModule` for module-mode parsing. funyacc must generate an entry function for each `%start` declaration.

- **Precedence requires conflict detection**: Declaring `%left INFIXOP0` only makes sense if funyacc detects the shift/reduce conflict it resolves. If funyacc silently ignores conflicts, precedence declarations are useless.

- **Parameterized rules require state threading**: `block_comment depth` and `read_indent col` pass integer state through recursive calls. The generated lexer must allow rule functions to accept extra parameters beyond `lexbuf`.

---

## MVP Definition

### Launch With (v1 — Bootstrapping Milestone)

Minimum needed to parse LangThree's actual Lexer.fsl and Parser.fsy.

**funlex v1:**
- [ ] Header block `{ ... }` copied verbatim to output
- [ ] Named regex definitions (`let ident = regexp`)
- [ ] Character class literals, ranges, union, negation
- [ ] Quantifiers `*`, `+`; optional `?` can be deferred if LangThree doesn't use it
- [ ] String and character literal patterns
- [ ] `eof` pattern
- [ ] Alternation `|` in rules
- [ ] Multiple named rules via `and`
- [ ] Parameterized rules (passes extra args before `lexbuf`)
- [ ] `lexbuf` implicit last argument in generated rule functions
- [ ] Actions as arbitrary F# — copied verbatim with `$1` variable binding for matched text
- [ ] `LexBuffer<_>` type in generated output (from FSharp.Text.Lexing)
- [ ] Longest-match + first-match-wins semantics
- [ ] Generate a `.fun` module (single F# module file)

**funyacc v1:**
- [ ] Header block `%{ ... %}` copied verbatim
- [ ] `%token` (no payload) and `%token <type>` (with payload) declarations
- [ ] `%start` and `%type` declarations (support multiple start symbols)
- [ ] `%left`, `%right`, `%nonassoc` precedence declarations
- [ ] `%%` section separator
- [ ] Grammar rules with actions: `$1`–`$N` semantic value access
- [ ] `parseState` parameter available in actions (for `InputStartPosition`/`InputEndPosition`)
- [ ] LALR(1) table generation (shift/reduce, reduce/reduce)
- [ ] Conflict resolution via precedence declarations
- [ ] Empty productions (epsilon rules)
- [ ] Multiple entry point functions (one per `%start`)
- [ ] Generate a `.fun` module

### Add After Validation (v1.x)

- [ ] Conflict reporting: print shift/reduce and reduce/reduce conflicts with state numbers — trigger: grammar debugging during LangThree evolution
- [ ] `%prec TOKEN` rule-level precedence override — trigger: if a grammar conflict arises that can't be resolved by restructuring
- [ ] Unicode mode for funlex — trigger: if LangThree adds non-ASCII support
- [ ] `?` optional quantifier — trigger: if needed by a grammar extension

### Future Consideration (v2+)

- [ ] Error recovery (`error` token) — defer: LangThree has no error productions; this requires significant grammar work
- [ ] Verbose conflict output (`-v` flag with .dot or state-machine description) — defer: useful but not blocking
- [ ] Source maps from generated F# back to .fsl/.fsy — defer: complex, low immediate value
- [ ] Binary table serialization — defer: premature optimization for bootstrap compiler

---

## Feature Prioritization Matrix

| Feature | Bootstrap Value | Implementation Cost | Priority |
|---------|-----------------|---------------------|----------|
| funlex: named regexes + character classes | HIGH | LOW | P1 |
| funlex: multiple parameterized rules | HIGH | MEDIUM | P1 |
| funlex: longest-match + first-match semantics | HIGH | MEDIUM | P1 |
| funyacc: token declarations + precedence | HIGH | LOW | P1 |
| funyacc: LALR(1) table generation | HIGH | HIGH | P1 |
| funyacc: multiple start symbols | HIGH | MEDIUM | P1 |
| funyacc: `parseState` / `IParseState` in actions | HIGH | MEDIUM | P1 |
| funyacc: conflict resolution via precedence | HIGH | MEDIUM | P1 |
| funlex: unicode mode | LOW | HIGH | P3 |
| funlex: character set difference `#` operator | LOW | LOW | P3 |
| funyacc: `%prec` rule-level override | MEDIUM | MEDIUM | P2 |
| funyacc: conflict reporting | MEDIUM | MEDIUM | P2 |
| funyacc: error recovery | LOW | HIGH | P3 |
| Either: source maps | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for bootstrapping LangThree
- P2: Should have, add when possible / grammar debugging
- P3: Nice to have, future consideration

---

## Competitor Feature Analysis

| Feature | fslex (FsLexYacc) | ocamllex | funlex (our plan) |
|---------|-------------------|----------|-------------------|
| Named regex defs | Yes (`let ident = ...`) | Yes | Yes (P1) |
| Multiple states via `and` | Yes | Yes | Yes (P1) |
| Parameterized rules | Yes | Yes | Yes (P1) |
| Unicode mode | Yes (`--unicode`) | Separate tool | Defer (P3) |
| Character set difference `#` | Yes | No | Defer (P3) |
| `eof` pattern | Yes | Yes | Yes (P1) |

| Feature | fsyacc (FsLexYacc) | menhir (OCaml) | funyacc (our plan) |
|---------|---------------------|----------------|---------------------|
| LALR(1) | Yes | Yes (also LR(1)) | Yes (P1) |
| Multiple start symbols | Yes | Yes | Yes (P1) |
| `%left`/`%right`/`%nonassoc` | Yes | Yes | Yes (P1) |
| `%prec` rule-level override | Yes | Yes | P2 |
| Error recovery | Yes | Yes | P3 |
| Conflict reporting | Yes (`-v`) | Yes (verbose) | P2 |
| IParseState / position tracking | Yes | Position API | Yes (P1) |
| GLR extension | No | No | Anti-feature |
| Incremental parsing | No | No | Anti-feature |

---

## Key Insight: Indentation Is a Pipeline Problem, Not a Generator Problem

The most important architectural insight from studying LangThree is that **indentation-sensitive syntax is NOT a feature of funlex or funyacc**. It is handled by:

1. **funlex-generated lexer**: emits `NEWLINE col` (a raw token with column number as payload) instead of silently skipping newlines
2. **IndentFilter.fs** (handwritten F# module): transforms `NEWLINE col` into `INDENT`/`DEDENT` tokens using an indent-stack algorithm with context awareness (handles match, try, let-offside rule, module blocks, multi-line function application)
3. **funyacc grammar**: consumes `INDENT`/`DEDENT` as first-class tokens in grammar rules

funlex/funyacc do not need to "understand" indentation at all. They just need to:
- funlex: emit `NEWLINE col` when a newline is seen (done in Lexer.fsl's `read_indent` rule)
- funyacc: accept `INDENT`/`DEDENT` as normal tokens in grammar rules

IndentFilter is **not** generated — it is maintained as handwritten code in LangThree and will remain so in FunLexYacc. This is the right separation of concerns.

---

## Sources

- LangThree/src/LangThree/Lexer.fsl — direct inspection (all funlex feature requirements)
- LangThree/src/LangThree/Parser.fsy — direct inspection (all funyacc feature requirements)
- LangThree/src/LangThree/IndentFilter.fs — direct inspection (indentation pipeline architecture)
- [FsLexYacc fslex docs](https://github.com/fsprojects/FsLexYacc/blob/master/docs/content/fslex.md) — feature enumeration for compatibility baseline
- [FsLexYacc fsyacc docs](https://fsprojects.github.io/FsLexYacc/content/fsyacc.html) — feature enumeration for compatibility baseline
- [FsLexYacc GitHub](https://github.com/fsprojects/FsLexYacc) — project overview

---
*Feature research for: funlex/funyacc lexer/parser generator for LangThree bootstrapping*
*Researched: 2026-03-23*
