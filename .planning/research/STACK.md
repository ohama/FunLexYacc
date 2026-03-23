# Stack Research

**Domain:** Lexer/parser generator (bootstrapping a functional language)
**Researched:** 2026-03-23
**Confidence:** HIGH

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| LangThree (.fun) | v1.8 (current) | Implementation language for funlex and funyacc | The entire point — bootstrapping requires the tools be written in the target language itself |
| LALR(1) via LR(0)+lookahead | canonical algorithm (DeRemer/Pennello 1982) | Parser table generation algorithm for funyacc | Matches fsyacc's existing behavior exactly; well-documented; sufficient for real-world grammars including LangThree's own grammar |
| Thompson's NFA construction | classical (1968) | Regex-to-NFA in funlex | Compositional, easy to implement recursively in a functional language; each regex operator maps to a small NFA fragment |
| Powerset/subset construction | classical | NFA-to-DFA conversion in funlex | Standard algorithm with tractable worst case for token-level regexes; produces correct DFA for longest-match |
| Hopcroft's partition refinement | classical (1971) | DFA minimization in funlex | O(n log n) in states × alphabet; reduces generated table size; practically essential for unicode lexers |
| .funl/.funy format | FsLexYacc-compatible | Input grammar format | Non-negotiable constraint — existing LangThree Lexer.funl and Parser.funy must be consumed without modification |
| .fun module output | LangThree module system | Generated code format | Non-negotiable constraint — generated lexer/parser must integrate with LangThree's module system |

### Algorithm Pipeline: funlex

```
.funl file
  → parse .funl header, named regex bindings, rule entry points
  → for each rule: build regex AST per pattern
  → Thompson's NFA construction per regex AST
  → compose all patterns into one NFA with ε-start-transitions
  → powerset construction: NFA → DFA (states = sets of NFA states)
  → Hopcroft minimization: reduce DFA state count
  → assign accept states to token actions by priority (first rule wins)
  → encode DFA as 2D transition table: int array[state][char_code] → state
  → longest-match runtime driver embedded in generated .fun module
  → output .fun source with: transition table literal, accept table, driver function
```

### Algorithm Pipeline: funyacc

```
.funy file
  → parse %token declarations, %type, %start, %left/%right/%nonassoc precedence
  → parse grammar productions with semantic actions
  → compute FIRST sets for all symbols
  → build LR(0) item sets via closure() and goto() functions
  → construct the LR(0) automaton (canonical collection of item sets)
  → compute LALR(1) lookahead sets:
      - spontaneous generation: which lookaheads arise in a state
      - propagation: which lookaheads copy from one item to another
      (DeRemer/Pennello: define READS, INCLUDES, LOOKBACK relations;
       compute transitive closure to get FOLLOW sets per state)
  → build action table: for each state×terminal:
      - shift if item has dot before terminal
      - reduce if dot at end and terminal in lookahead set
      - accept if start symbol reduction
  → build goto table: for each state×nonterminal: target state
  → resolve shift/reduce conflicts using %left/%right/%nonassoc declarations
  → report unresolved conflicts as errors (or warnings with default resolution)
  → encode tables as arrays of integers
  → output .fun source with: action table, goto table, token type union,
    semantic value union, table-driven parse() function
```

### Key Data Structures

| Data Structure | Used In | What It Represents |
|----------------|---------|-------------------|
| `RegexAst` (ADT) | funlex | Parsed regex: Char, CharClass, Seq, Alt, Star, Plus, Opt, Ref |
| `NfaState` (int + transitions) | funlex | NFA node with labeled and ε-edges |
| `DfaState` (set of NFA states) | funlex | DFA node from subset construction |
| `DfaTable` (int array[state][char]) | funlex | Final minimized transition table |
| `AcceptTable` (int array[state]) | funlex | Maps accept states to token action index |
| `Item` (production + dot position + lookahead set) | funyacc | LR(1) item |
| `ItemSet` (set of Items) | funyacc | LR state |
| `ActionTable` (int array[state][terminal]) | funyacc | Shift/reduce/accept/error entries |
| `GotoTable` (int array[state][nonterminal]) | funyacc | State after reduction |
| `GrammarSymbol` (Terminal \| Nonterminal) | funyacc | Token or grammar symbol |
| `Production` (lhs + rhs list + action) | funyacc | Grammar rule with embedded action code |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| LangThree standard Map/Set | built-in | State sets, first/follow sets, item sets | Throughout both tools — ordered maps and sets are fundamental |
| LangThree string functions | built-in | Code generation string building | When emitting .fun source text |
| LangThree list operations | built-in | Closure computation, BFS/DFS over states | Item set construction, NFA ε-closure |

No external library dependencies. Both tools are pure algorithmic implementations using only LangThree's standard library.

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| Existing fslex/fsyacc | Bootstrap phase 0 — compile the .funl/.funy inputs for funlex/funyacc themselves | You need fslex/fsyacc to parse the .funl/.funy format during initial development; this dependency is removed once funlex/funyacc are self-hosting |
| LangThree interpreter/compiler (v1.8) | Run funlex and funyacc | The host for all .fun code execution |
| fslit integration test suite | 641 tests | Primary correctness oracle — generated lexer+parser must pass all 641 existing tests |
| F# unit tests (199 tests) | Secondary correctness oracle | Additional coverage |

## Installation

No package installation required. These are pure algorithmic tools implemented in LangThree:

```bash
# Compile funlex and funyacc using the existing F# bootstrap
dotnet build   # uses existing fslex/fsyacc to compile the generators

# Once funlex/funyacc are functional, run them on LangThree's own grammar:
funlex Lexer.funl -o Lexer.fun
funyacc Parser.funy -o Parser.fun
```

## Alternatives Considered

| Category | Recommended | Alternative | When to Use Alternative |
|----------|-------------|-------------|------------------------|
| Parser algorithm | LALR(1) | LR(1) (canonical) | If LALR(1) produces conflicts on LangThree's grammar that aren't resolvable with precedence declarations — unlikely but possible. Menhir uses full LR(1) for this reason |
| Parser algorithm | LALR(1) | Earley / GLL / GLR | Only if the grammar is genuinely ambiguous and you need to handle all parses. Overkill for a well-designed ML-family grammar |
| Parser algorithm | LALR(1) | Recursive descent (hand-written) | If the grammar were LL(1) — it is not, due to expression precedence. LALR(1) handles this correctly |
| Lexer algorithm | DFA table-driven | Hand-written recursive descent lexer | Valid only if the token set is tiny and stable. Defeats the purpose of funlex |
| Lookahead computation | DeRemer/Pennello (propagation-based) | Canonical LR(1) merge | Canonical LR(1) has exponentially more states than LALR(1); propagation-based LALR is standard and sufficient |
| DFA minimization | Hopcroft's algorithm | Brzozowski's algorithm | Brzozowski is simpler to code but requires two full DFA determinizations; Hopcroft is faster in practice |
| NFA construction | Thompson's construction | Direct DFA from regex (Glushkov/Berry-Sethi) | Glushkov avoids ε-transitions and can be faster, but Thompson's is simpler to implement and correct first |
| Code generation output | .fun source text | Binary table format | Source text is debuggable, version-controllable, and integrates with LangThree's existing build pipeline |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| PEG / Packrat parsing for funyacc | Out of scope per PROJECT.md; would break .funy compatibility; fundamentally different semantics (no left recursion without memoization tricks) | LALR(1) |
| LL(k) / recursive descent parser generator | LangThree's grammar has left-recursive expressions and operator precedence; LL parsers require grammar rewriting that changes semantic actions | LALR(1) |
| SLR(1) for lookahead computation | SLR uses global FOLLOW sets, not per-state lookaheads; produces more conflicts than LALR(1) on typical grammars; fsyacc uses LALR(1) and the replacement must match | LALR(1) with per-state lookaheads |
| Unicode full 21-bit DFA tables | A 2D array[state][0x10FFFF] is 4M entries per state — impractical | Equivalence classes: group input bytes/chars into ~100 character classes, index by class rather than raw char code |
| Generating C or JS output | This project targets LangThree only; generating other languages adds complexity with zero benefit for the bootstrapping goal | Generate .fun source only |
| External parser generator tools (Menhir, ANTLR, Bison) | Would create a new external dependency and not contribute to the bootstrapping goal | Self-contained implementation in LangThree |
| fsyacc's known %nonassoc bug | fsyacc has a confirmed bug where %nonassoc sometimes behaves like %right (see FsLexYacc issue #39) | Implement %nonassoc correctly: eliminate both shift and reduce for that operator as lookahead |

## Stack Patterns by Variant

**If LangThree grammar has LALR(1) conflicts not resolvable by precedence:**
- Inspect the conflict states using the -v verbose output from funyacc
- Refactor the grammar rule (introduce an intermediate non-terminal) before resorting to LR(1)
- This is the approach OCaml itself uses with its grammar (it has known refactored productions for exactly this reason)

**If DFA state explosion occurs during lexer generation:**
- Use equivalence classes: partition the 256 ASCII or 65536 Unicode chars into groups with identical transitions
- Reduces the transition table from [states × 256] to [states × n_classes] where n_classes is typically 50–100
- fslex uses unicode equivalence classes internally; funlex should do the same

**If the bootstrapping chicken-and-egg problem arises (funlex needs to lex .funl, but funlex is written in .funl/.funy):**
- Phase 0: write a hand-rolled .funl/.funy parser in LangThree (recursive descent is fine for the simple .funl/.funy grammar)
- Phase 1: once funlex/funyacc generate correct output, use them to regenerate their own parsers
- Phase 2: remove the hand-rolled parsers; the tools are now fully self-hosting
- This is the exact pattern used by Alex/Happy (Happy ships a parser-combinator-based bootstrap implementation) and OCaml's ocamlyacc

## Reference Implementations to Study

| Tool | Language | What to Learn | Source |
|------|----------|--------------|--------|
| FsLexYacc | F# | Exact .funl/.funy format, runtime LexBuffer/tables structure, generated code layout | https://github.com/fsprojects/FsLexYacc |
| ocamllex / ocamlyacc | OCaml | DFA generation for lexer, LALR table generation, bootstrapping strategy | OCaml distribution source (`bytecomp/`, `parsing/`) |
| Alex | Haskell | DFA generation with equivalence classes, bootstrap strategy (pre-generated Parser.hs/Scan.hs on Hackage) | https://github.com/haskell/alex |
| Happy | Haskell | LALR(1) table construction in pure FP, bootstrap via parser-combinator fallback | https://github.com/haskell/happy |
| Menhir | OCaml | Full LR(1) construction (useful reference for lookahead computation even if we use LALR subset) | https://github.com/LexiFi/menhir |
| Lemon | C | Clean, readable LALR(1) implementation used by SQLite; lemon.c is ~5000 lines of well-commented C | https://sqlite.org/lemon.html |
| GNU Bison | C | Precedence/associativity conflict resolution, %left/%right/%nonassoc semantics | https://www.gnu.org/software/bison/ |

**Priority order for studying:** FsLexYacc (format compatibility) → Lemon (clean LALR(1)) → Alex (DFA equivalence classes) → Happy (LALR in FP).

## Version Compatibility

| Component | Compatibility Requirement | Notes |
|-----------|--------------------------|-------|
| .funl input | Must match FsLexYacc's documented format | Unicode flag (`--unicode`) must be supported if LangThree's Lexer.funl uses it |
| .funy input | Must match FsLexYacc's format including `%left`, `%right`, `%nonassoc`, `%token <type>`, `%type`, `%start` | Implement %nonassoc correctly (unlike fsyacc's known bug) |
| Generated .fun output | Must use LangThree v1.8 module syntax | Type-check against LangThree's type system; pattern match on token union type |
| LALR table integers | No specific constraint | Use LangThree's native int for state indices and action codes |
| Runtime LexBuffer equivalent | Must implement the same interface as FsLexYacc.Runtime.LexBuffer | The generated lexer functions take a buffer argument and return tokens |

## Sources

- [FsLexYacc GitHub](https://github.com/fsprojects/FsLexYacc) — Format specification for .funl/.funy compatibility (HIGH confidence)
- [FsLex format docs](https://github.com/fsprojects/FsLexYacc/blob/master/docs/content/fslex.md) — Complete .funl regex syntax (HIGH confidence)
- [FsYacc format docs](https://github.com/fsprojects/FsLexYacc/blob/master/docs/content/fsyacc.md) — .funy token/type/rule syntax (MEDIUM confidence — incomplete documentation)
- [FsLexYacc issue #39](https://github.com/fsprojects/FsLexYacc/issues/39) — %nonassoc bug in fsyacc (HIGH confidence — confirmed bug report)
- [LALR parser Wikipedia](https://en.wikipedia.org/wiki/LALR_parser) — Algorithm overview including DeRemer/Pennello (HIGH confidence)
- [LALR parser generator Wikipedia](https://en.wikipedia.org/wiki/LALR_parser_generator) — State construction, table building (HIGH confidence)
- [Alex on Stackage](https://www.stackage.org/nightly-2026-03-22/package/alex-3.5.4.1) — Current Alex version (3.5.4.1) and bootstrap approach (HIGH confidence)
- [Happy GitHub](https://github.com/haskell/happy) — Bootstrap parser-combinator strategy (MEDIUM confidence — inferred from search results)
- [OCamllex manual](https://ocaml.org/manual/5.4/lexyacc.html) — Reference for lexer generator design (HIGH confidence)
- [Menhir project](https://gallium.inria.fr/~fpottier/menhir/) — LR(1) reference for lookahead computation (HIGH confidence)
- [Compact lexer table representation](https://def.lakaban.net/2020-05-02-compact-lexer-table-representation/) — DFA table compression via overlapping vectors (HIGH confidence)
- [DeRemer & Pennello 1982](https://www.semanticscholar.org/paper/Efficient-Computation-of-LALR(1)-Look-Ahead-Sets-DeRemer-Pennello/4337e7504e0d43a0c21802d5301fdbb1c3950d0f) — Original LALR(1) lookahead propagation algorithm (HIGH confidence)
- [Parsing with Haskell Alex Part 1](https://serokell.io/blog/lexing-with-alex) — Practical lexer generator usage (MEDIUM confidence)
- [Self-hosted parser design — Drew DeVault](https://drewdevault.com/2021/04/22/Our-self-hosted-parser-design.html) — Bootstrapping strategy lessons (MEDIUM confidence)
- [Derw bootstrapping post](https://derw.substack.com/p/writing-a-bootstrapping-compiler) — Functional language bootstrapping example (MEDIUM confidence)

---
*Stack research for: funlex/funyacc — lexer/parser generator bootstrapping for LangThree*
*Researched: 2026-03-23*
