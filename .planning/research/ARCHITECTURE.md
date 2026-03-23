# Architecture Patterns

**Domain:** Lexer/Parser Generator (funlex/funyacc) — bootstrapping tool for LangThree
**Researched:** 2026-03-23
**Confidence:** HIGH

## Standard Architecture

### System Overview

The full pipeline — from .fsl/.fsy specification files to a running LangThree compiler — has three distinct layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SPECIFICATION LAYER                            │
│  ┌─────────────┐          ┌─────────────┐      ┌───────────────┐   │
│  │  Lexer.fsl  │          │  Parser.fsy │      │  (embedded    │   │
│  │ (lex rules) │          │ (grammar)   │      │   .fun code)  │   │
│  └──────┬──────┘          └──────┬──────┘      └───────────────┘   │
└─────────┼───────────────────────┼─────────────────────────────────┘
          │                       │
          ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    GENERATOR LAYER (written in .fun)                │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐  │
│  │       funlex         │    │            funyacc               │  │
│  │                      │    │                                  │  │
│  │  FslParser (F# DSL   │    │  FsyParser (F# DSL parser)       │  │
│  │   parser for .fsl)   │    │  LALR(1) Table Builder           │  │
│  │  NFA Builder         │    │  Grammar Conflict Resolver       │  │
│  │  DFA Minimizer       │    │  Code Emitter                    │  │
│  │  Code Emitter        │    │                                  │  │
│  └──────────┬───────────┘    └──────────────────┬───────────────┘  │
└─────────────┼───────────────────────────────────┼──────────────────┘
              │                                   │
              ▼                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   GENERATED OUTPUT LAYER (.fun modules)             │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐  │
│  │    Lexer.fun          │    │          Parser.fun              │  │
│  │  (DFA table lookup)  │    │  (LALR table interp. + actions)  │  │
│  └──────────────────────┘    └──────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    IndentFilter.fun                           │  │
│  │           (NEWLINE(col) → INDENT/DEDENT/IN)                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| FslParser | Parse .fsl spec file into an in-memory lex rule AST | Hand-written recursive descent |
| NFA Builder | Convert regex rules from the lex AST into NFA graph | Thompson's construction algorithm |
| DFA Builder | Convert NFA to deterministic DFA | Subset construction |
| DFA Minimizer | Minimize DFA state count | Hopcroft's algorithm (optional for v1) |
| Lexer Code Emitter | Emit Lexer.fun from DFA transition table | Template string generation |
| FsyParser | Parse .fsy spec file into a grammar AST | Hand-written recursive descent |
| Grammar Extractor | Extract terminals, nonterminals, productions, precedence rules | Walks grammar AST |
| LALR(1) Table Builder | Build LR(0) items, compute LALR(1) lookahead sets, build action/goto tables | Classic Dragon Book algorithm |
| Conflict Resolver | Resolve shift/reduce and reduce/reduce conflicts using precedence annotations | Precedence table lookup |
| Parser Code Emitter | Emit Parser.fun from action/goto tables + semantic actions | Template string generation |
| IndentFilter.fun | Stateful filter converting NEWLINE(col) tokens to INDENT/DEDENT/IN | Port of existing IndentFilter.fs |

## Recommended Project Structure

```
src/
├── FslSpec/              # .fsl spec file representation
│   ├── FslAst.fun        # AST: LexSpec, Rule, RegexExpr, Action
│   └── FslParser.fun     # Parse .fsl → FslAst
├── Nfa/                  # NFA construction from lex rules
│   ├── NfaTypes.fun      # NfaState, NfaEdge, NfaGraph
│   └── NfaBuild.fun      # Thompson's construction
├── Dfa/                  # DFA construction from NFA
│   ├── DfaTypes.fun      # DfaState, DfaTable
│   └── DfaBuild.fun      # Subset construction + minimization
├── FunLex/               # funlex top-level
│   └── LexEmit.fun       # Emit Lexer.fun from DFA
├── FsySpec/              # .fsy spec file representation
│   ├── FsyAst.fun        # Grammar, Production, Precedence, Action
│   └── FsyParser.fun     # Parse .fsy → FsyAst
├── Lalr/                 # LALR(1) table construction
│   ├── LrItems.fun       # LR(0) item sets, closure, goto
│   ├── Lookahead.fun     # LALR(1) lookahead propagation
│   └── Tables.fun        # Action/goto table builder + conflict resolution
├── FunYacc/              # funyacc top-level
│   └── ParserEmit.fun    # Emit Parser.fun from LALR tables
├── IndentFilter/         # Standalone indent filter module
│   └── IndentFilter.fun  # Port of IndentFilter.fs
└── Main.fun              # CLI entry point: dispatch funlex/funyacc
```

### Structure Rationale

- **FslSpec/ and FsySpec/ are separate:** The input formats are distinct domain languages. Keeping their ASTs and parsers isolated prevents coupling and allows either to evolve independently.
- **Nfa/ and Dfa/ are separate:** The NFA→DFA pipeline is a two-stage transformation. Keeping them separate makes each stage testable in isolation.
- **Lalr/ is isolated from FsySpec/:** Grammar analysis (LALR tables) should not depend on the surface syntax of .fsy files. FsyParser produces a normalized grammar representation that Lalr/ operates on.
- **IndentFilter/ is standalone:** It must be usable independently of funlex/funyacc output. It is a pure runtime filter, not a generator component.
- **Emitters are separate from builders:** LexEmit and ParserEmit concern string generation. Keeping them separate from the automata logic means algorithm correctness is testable without checking string output.

## Architectural Patterns

### Pattern 1: Two-Phase Spec Parsing

**What:** Each specification format (.fsl, .fsy) is parsed by a dedicated hand-written recursive descent parser that produces a typed AST. The AST is then separately processed by the automaton or grammar builder.

**When to use:** Always — separating parsing from analysis is the foundation of composability.

**Trade-offs:** Requires writing two parsers (one for .fsl, one for .fsy), but gives clean boundaries between "what the user wrote" and "what the generator does with it."

**Example (.fun pseudocode):**
```
// FslParser produces this AST
type LexSpec =
  | LexSpec of header: string * rules: LexRule list * footer: string

type LexRule =
  | LexRule of name: string * cases: (RegexExpr * string) list

// NfaBuild takes LexSpec, not raw text
let buildNfa (spec: LexSpec) : NfaGraph = ...
```

### Pattern 2: DFA-as-Data, Emitter-as-Template

**What:** The DFA is represented as an immutable data structure (transition table as a list/map). The emitter serializes this data into LangThree source text. The generated lexer module is a self-contained table-driven interpreter.

**When to use:** Always for generator output.

**Trade-offs:** Generated code is verbose but straightforward to debug. The table is data, not code, so the generated module has no dependency on funlex at runtime.

**Generated Lexer.fun structure:**
```
// Emitted by LexEmit
module Lexer

// DFA transition table encoded as a 2D list
let table : (int * int) list list = [
  [(0, 1); (1, 2); ...];  // state 0 transitions
  ...
]

// Accepting states: state index → token constructor name
let accepting : (int * (string -> Token)) list = [
  (3, fun s -> IDENT s);
  (7, fun _ -> LET);
  ...
]

// Entry point: tokenize a character stream
let tokenize (input: CharStream) : Token = ...
```

### Pattern 3: LALR Table as Encoded Arrays

**What:** The LALR(1) action and goto tables are encoded as integer arrays (matching the fsyacc approach seen in Parser.fs). Semantic actions are an array of closures indexed by production number. The generated parser is a table interpreter with a value stack.

**When to use:** Always — this is the standard approach and matches what fsyacc generates, ensuring behavioral compatibility.

**Trade-offs:** The tables are opaque to inspection but are compact and fast. Generating them as LangThree list literals (rather than binary blobs) keeps the output readable.

**Generated Parser.fun structure:**
```
// Emitted by ParserEmit
module Parser

// action table: state × terminal → Shift(s) | Reduce(p) | Accept | Error
let actionTable : int list list = [ ... ]

// goto table: state × nonterminal → state
let gotoTable : int list list = [ ... ]

// semantic actions: production index → (valueStack → AST node)
let reductions : (Value list -> Value) list = [
  fun args -> box (Ast.Number (unbox args.[0]));
  ...
]

// LALR interpreter loop
let parse (lexer: unit -> Token) : Ast.Module = ...
```

### Pattern 4: IndentFilter as Pure Stream Transformer

**What:** IndentFilter.fun takes a `Token list` (or `Token seq`) and returns a `Token list`, inserting INDENT, DEDENT, and IN tokens. It is a pure stateful fold over the token stream, not a coroutine or callback.

**When to use:** The existing IndentFilter.fs follows this pattern exactly. The .fun port should preserve it.

**Trade-offs:** Requires materializing the full token list before parsing (the current implementation does this: `let rawTokens = collect()`). This is fine for source files of practical size.

**Port strategy — existing F# state maps directly to LangThree ADT:**
```
// F# type:
type FilterState = {
  IndentStack: int list
  Context: SyntaxContext list
  PrevToken: Parser.token option
  ...
}

// LangThree equivalent:
type FilterState = {
  indentStack: int list
  context: SyntaxContext list
  prevToken: Token option
  ...
}
```

### Pattern 5: Module Boundary = One .fun File

**What:** Each component (Lexer, Parser, IndentFilter) is one LangThree module, one .fun file. The generated files are designed to be `open`-ed in the compiler's pipeline module.

**When to use:** Matches LangThree's existing module system (Module, NamedModule, open declarations).

**Trade-offs:** No sub-module namespacing within a file, but LangThree's module system does not yet support nested sub-modules within a file either.

## Data Flow

### funlex Data Flow

```
.fsl file (string)
    │
    ▼ FslParser
FslAst (LexSpec)
    │
    ▼ NfaBuild (Thompson's construction per rule)
NfaGraph (states × edges × accepting predicates)
    │
    ▼ DfaBuild (subset construction)
DfaTable (states × transition map × accepting map)
    │
    ▼ LexEmit (template generation)
Lexer.fun (string written to disk)
```

### funyacc Data Flow

```
.fsy file (string)
    │
    ▼ FsyParser
FsyAst (Grammar: terminals, nonterminals, productions, precedence, header/footer)
    │
    ▼ LrItems (LR(0) item set construction + closure)
LR(0) item sets (states × kernel items × closure items)
    │
    ▼ Lookahead (LALR(1) lookahead propagation)
LALR(1) item sets (each item annotated with lookahead terminal set)
    │
    ▼ Tables (action/goto table construction + conflict resolution)
ActionTable × GotoTable (int arrays)
    │
    ▼ ParserEmit (table serialization + semantic action embedding)
Parser.fun (string written to disk)
```

### Runtime Pipeline (After Code Generation)

This is how generated modules plug into the LangThree compiler pipeline:

```
Source file (string)
    │
    ▼ Lexer.fun (generated DFA table interpreter)
Token list (includes NEWLINE(col) tokens, raw)
    │
    ▼ IndentFilter.fun (stateful fold)
Token list (INDENT/DEDENT/IN inserted, NEWLINE consumed)
    │
    ▼ Parser.fun (generated LALR table interpreter)
Ast.Module (same AST type as today)
    │
    ▼ Elaborate → TypeCheck → Eval (unchanged pipeline stages)
Value
```

### Key Data Flows

1. **Spec-to-Table:** Both funlex and funyacc read a spec file (text), produce an internal AST, run an algorithm on it (NFA→DFA or LALR table construction), and emit a .fun source file. The algorithm runs at generator time; the table is frozen into the generated file.

2. **Token-stream threading:** The lexer emits a `Token list`. IndentFilter transforms that list in a single pass. The parser consumes the filtered list via a tokenizer function wrapping the list. This matches the current `parseModuleFromString` pattern exactly.

3. **AST type reuse:** The generated Parser.fun must produce `Ast.Module` values (or whatever the target AST type is). The semantic actions embedded in Parser.fun reference types from the target's Ast module. This means Parser.fun has a compile-time dependency on the target's Ast module — an expected coupling.

## Suggested Build Order

The dependency graph between components drives the build order:

```
Phase 1: IndentFilter.fun
    — No dependencies on funlex/funyacc. Testable immediately.
    — Port of IndentFilter.fs, pure list transformation.

Phase 2: FslParser + FslAst (funlex input parsing)
    — Depends only on LangThree itself (string processing, ADTs).
    — Can be tested by parsing Lexer.fsl and printing the AST.

Phase 3: NFA Builder + DFA Builder (funlex automaton)
    — Depends on FslAst. No external dependencies.
    — Test by verifying DFA tables for small regex sets.

Phase 4: LexEmit (funlex output)
    — Depends on DfaTable. Produces Lexer.fun.
    — Integration test: run generated Lexer.fun on LangThree source.

Phase 5: FsyParser + FsyAst (funyacc input parsing)
    — Depends only on LangThree. Can parse Parser.fsy, print grammar.

Phase 6: LrItems + Lookahead (funyacc LALR construction)
    — Depends on FsyAst. Core algorithm, testable with toy grammars.

Phase 7: Tables + Conflict Resolution (funyacc table generation)
    — Depends on Lookahead. Test by comparing tables against fsyacc output.

Phase 8: ParserEmit (funyacc output)
    — Depends on Tables. Produces Parser.fun.
    — Integration test: run generated Parser.fun on LangThree source.

Phase 9: End-to-end bootstrap validation
    — Replace Lexer.fs/Parser.fs with Lexer.fun/Parser.fun in LangThree pipeline.
    — Run all 641 tests to confirm behavioral equivalence.
```

## Anti-Patterns

### Anti-Pattern 1: Conflating Generator and Runtime

**What people do:** Build the DFA or LALR interpreter into the generator, and have the generated code call back into generator functions at runtime.

**Why it's wrong:** The generated .fun module must be self-contained — it runs inside LangThree, not inside funlex/funyacc. Any runtime behavior must be emitted as .fun code, not as a reference to a generator library.

**Do this instead:** The generated code is a complete, standalone .fun module. The generator emits all logic as source text. The only coupling between generator and generated code is the table data format.

### Anti-Pattern 2: Generating Unreadable Output

**What people do:** Encode tables as single-line integer blobs with no structure, producing generated files that are impossible to debug.

**Why it's wrong:** During development, the generated Lexer.fun and Parser.fun will need manual inspection to diagnose bugs. Unreadable output makes this impossible. fsyacc's `Parser.fs` is already 3,389 lines of densely packed arrays — tolerable in a binary artifact, but not in a tool we need to debug.

**Do this instead:** Structure generated output with one state per line, named constants for special sentinel values, and comments that reference the original .fsl/.fsy line numbers.

### Anti-Pattern 3: Encoding IndentFilter Logic in the Generated Lexer

**What people do:** Try to handle indentation sensitivity inside the DFA, adding INDENT/DEDENT states to the lexer automaton.

**Why it's wrong:** The existing IndentFilter has complex context tracking (match arms, try-with, module bodies, let-binding offside rule). Embedding this in DFA states is impractical and would break the clean separation between lexical scanning and syntactic indentation processing.

**Do this instead:** Keep IndentFilter as a separate pipeline stage, exactly as it is today. The lexer emits NEWLINE(col); the filter transforms it. This is the current architecture and it works.

### Anti-Pattern 4: Reusing fsyacc's Binary Table Format

**What people do:** Try to produce the exact same `_fsyacc_actionTableElements` byte arrays that fsyacc generates, to avoid writing a new table interpreter.

**Why it's wrong:** That format depends on FsLexYacc's `FSharp.Text.Parsing.Tables` runtime which is F#/.NET specific and cannot run inside LangThree's interpreter. The whole point is to replace that dependency.

**Do this instead:** Emit the tables as LangThree list literals and write a small LALR interpreter inside Parser.fun that operates on those lists. The interpreter is ~50-100 lines and only needs to be written once.

### Anti-Pattern 5: Skipping Grammar Normalization

**What people do:** Feed the raw .fsy grammar directly into LR item construction without normalizing it first (augmenting with a start production, resolving %token/%left/%right declarations into a precedence table).

**Why it's wrong:** LALR construction requires a specific normalized form. Without augmentation with a fresh start production, the accept condition is undefined. Without a precedence table, all shift/reduce conflicts produce errors.

**Do this instead:** FsyParser produces a raw FsyAst. A separate normalization step (inside the Tables phase) augments the grammar and builds the precedence map before LALR construction begins.

## Integration Points

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| FslParser → NfaBuild | FslAst.LexSpec (ADT) | Clean; NfaBuild knows nothing about .fsl syntax |
| NfaBuild → DfaBuild | NfaTypes.NfaGraph (ADT) | Immutable; DfaBuild reads it, does not modify |
| DfaBuild → LexEmit | DfaTypes.DfaTable (ADT) | LexEmit serializes this to text |
| FsyParser → Tables | FsyAst.Grammar (ADT) | Grammar is normalized before table construction |
| Tables → ParserEmit | LALR action/goto as int list list | ParserEmit serializes to LangThree list literals |
| Lexer.fun → IndentFilter.fun | Token list | IndentFilter reads and produces Token list |
| IndentFilter.fun → Parser.fun | Token list (as closure) | Parser wraps list in a tokenizer function |

### External Boundaries

| Interface | Direction | Notes |
|-----------|-----------|-------|
| .fsl file (text) | Input to funlex | Must handle full Lexer.fsl syntax |
| .fsy file (text) | Input to funyacc | Must handle full Parser.fsy syntax |
| Lexer.fun (text) | Output of funlex | Written to disk, then compiled by LangThree |
| Parser.fun (text) | Output of funyacc | Written to disk, then compiled by LangThree |
| IndentFilter.fun (text) | Shipped with FunLexYacc | Compiled into LangThree at bootstrap |
| Ast module (types) | Compile-time dep of Parser.fun | Parser.fun references Ast.Module, Ast.Expr etc. |

## Scaling Considerations

This is a bootstrapping tool, not a user-facing service. Scaling concerns are irrelevant. The relevant sizing question is:

| Input Size | Concern | Approach |
|------------|---------|----------|
| LangThree's own Lexer.fsl (~170 lines) | NFA/DFA construction time | Straightforward; even naive subset construction finishes instantly |
| LangThree's own Parser.fsy (~640 lines, ~150 productions) | LALR table construction time | The LALR algorithm for grammars of this size completes in under a second even in interpreted LangThree |
| Generated Parser.fun | File size | fsyacc's Parser.fs is 3,389 lines; our output will be comparable |

DFA minimization (Hopcroft's algorithm) is optional for v1 — the unminimized DFA for a grammar-sized lexer will have at most a few hundred states.

## Sources

- LangThree source code: `/Users/ohama/vibe-coding/LangThree/src/LangThree/`
  - `Lexer.fsl` — fslex input format, DFA rule structure
  - `Parser.fsy` — fsyacc grammar format, precedence declarations, semantic actions
  - `Parser.fs` — generated parser structure: token ADT, tag functions, action/goto tables, `_fsyacc_reductions` array, `tables` record, `engine`/`parseModule`/`start` entry points
  - `IndentFilter.fs` — FilterState ADT, context stack, processNewline, offside rule logic
  - `Program.fs` — pipeline wiring: lex → filter → parse → elaborate → typecheck → eval
  - `Ast.fs` — Expr, Pattern, Module, Decl ADTs with span tracking
- Dragon Book (Aho, Lam, Sethi, Ullman) — authoritative source for LALR(1) construction, Thompson's construction, subset construction
- FsLexYacc source: https://github.com/fsprojects/FsLexYacc — reference for fslex/fsyacc input format semantics

---
*Architecture research for: FunLexYacc (funlex/funyacc bootstrapping tool for LangThree)*
*Researched: 2026-03-23*
