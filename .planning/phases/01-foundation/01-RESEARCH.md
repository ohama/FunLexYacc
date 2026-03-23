# Phase 1: Foundation - Research

**Researched:** 2026-03-23
**Domain:** Test harness infrastructure, project scaffolding, fslit integration, LangThree CLI
**Confidence:** HIGH

## Summary

Phase 1 establishes the safety net before any generator code is written. The core work is a side-by-side comparison harness that runs generated .fun pipeline components against the F# reference and reports token-level or AST-level differences. Three concrete deliverables: (1) the `src/` directory skeleton, (2) the comparison harness as a shell script or .fun driver, and (3) a per-component isolation runner with a rollback target.

The LangThree binary already provides `--emit-tokens` and `--emit-ast` diagnostic modes that output deterministic text. This is the comparison oracle. The harness needs to invoke the reference binary, invoke a generated-component path, and diff the outputs. Because the fslit test suite is already structured with `// --- Command: <binary> %input` and `// --- Output: <expected>`, the harness can reuse the 442 .flt test files as input corpus directly, rather than inventing a new test format.

An important baseline finding: the current LangThree repo has 74 pre-existing fslit test failures (368/442 passing), mostly in `file/offside/`, `file/option/`, and some `file/adt/` tests. The side-by-side harness must compare generated output against the F# reference — not against expected output in .flt files — because the .flt expected outputs were written assuming the F# pipeline. The rollback target must restore the F# Lexer.fs/Parser.fs and confirm it returns to 368/442 baseline.

**Primary recommendation:** Write the side-by-side harness as a shell script that takes a binary path and an input file, invokes both the reference binary and a candidate binary with identical arguments, and diffs stdout. This approach is zero-dependency, debuggable, and composable with `dotnet test`, `fslit`, or direct invocation. The per-component isolation runner is a separate script that stubs missing components with passthroughs so each component can be tested alone.

## Standard Stack

There is no external library stack for Phase 1. All tooling is already present.

### Core

| Tool | Version | Purpose | Why Standard |
|------|---------|---------|--------------|
| LangThree binary | Release (net10.0) | Reference oracle for token/AST output | Already built; `--emit-tokens` and `--emit-ast` produce stable text output |
| fslit | installed at `~/.local/bin/fslit` | Run 442 integration tests | Already used by LangThree; .flt format understood |
| dotnet test | .NET 10 SDK | Run 199 F# unit tests | Already used by LangThree |
| bash/zsh | system | Harness scripts | Zero new dependencies; composable |

### Supporting

| Tool | Version | Purpose | When to Use |
|------|---------|---------|-------------|
| LangThree interpreter | current | Execute .fun source files | When any harness logic is written in .fun rather than bash |
| git | system | Rollback target via `git checkout` | For restoring Lexer.fs/Parser.fs during rollback |
| diff | system | Byte-level output comparison | Side-by-side diff of reference vs generated output |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Shell script harness | .fun harness program | .fun harness has no file I/O or process execution yet — shell is correct for Phase 1 |
| Shell script harness | F# test project | Adds F# dependency, slower iteration; shell is simpler and directly composable with CI |
| diff for comparison | Custom diff tool | Overkill; token-level diff via `diff` on `--emit-tokens` output is sufficient |

**Installation:** Nothing to install. All tools present.

## Architecture Patterns

### Recommended Project Structure

```
FunLexYacc/
├── src/
│   ├── IndentFilter/
│   │   └── IndentFilter.fun      # Phase 2 — empty skeleton in Phase 1
│   ├── FslSpec/
│   │   └── .gitkeep              # Phase 3 placeholder
│   ├── FsySpec/
│   │   └── .gitkeep              # Phase 4 placeholder
│   ├── Nfa/
│   │   └── .gitkeep              # Phase 3 placeholder
│   ├── Dfa/
│   │   └── .gitkeep              # Phase 3 placeholder
│   ├── Lalr/
│   │   └── .gitkeep              # Phase 5 placeholder
│   └── Main.fun                  # CLI entry point skeleton
├── tests/
│   ├── harness/
│   │   ├── compare.sh            # Side-by-side comparison: reference vs candidate
│   │   ├── run-isolated.sh       # Per-component runner with stubs
│   │   └── rollback.sh           # Restore F# pipeline and verify baseline
│   └── fixtures/
│       └── (symlink or copy of LangThree flt test inputs)
└── .planning/
    └── phases/01-foundation/
```

### Pattern 1: Side-by-Side Shell Harness

**What:** A script that takes a reference binary path and a candidate binary path, runs both on the same input using the same CLI flags, and diffs stdout. Exits 0 if identical, 1 with a diff if different.

**When to use:** For every comparison test in Phase 1. The script is the single reusable primitive the per-component runner and CI both call.

**Example:**
```bash
#!/usr/bin/env bash
# compare.sh: compare reference vs candidate on a single input
# Usage: compare.sh <ref-binary> <candidate-binary> <flags...> <input-file>
# Returns 0 if identical, 1 if different

REF=$1; CAND=$2; shift 2; FLAGS=("${@:1:$#-1}"); INPUT="${@: -1}"

REF_OUT=$("$REF" "${FLAGS[@]}" "$INPUT" 2>/dev/null)
CAND_OUT=$("$CAND" "${FLAGS[@]}" "$INPUT" 2>/dev/null)

if [ "$REF_OUT" = "$CAND_OUT" ]; then
    echo "PASS"
    exit 0
else
    echo "FAIL"
    diff <(echo "$REF_OUT") <(echo "$CAND_OUT")
    exit 1
fi
```

### Pattern 2: Per-Component Isolation via Stub Binary

**What:** To test a component before its dependencies exist, create a stub binary that wraps the real LangThree binary but intercepts only the stage being tested. For example, to test IndentFilter.fun in isolation, the stub lexes with the real Lexer (F# reference) and then applies IndentFilter.fun's logic, bypassing the real parser.

**When to use:** Phase 1 sets up the infrastructure so later phases can run `run-isolated.sh lexer <input>`, `run-isolated.sh indent-filter <input>`, `run-isolated.sh parser <input>` independently.

**Implementation:** The isolation runner is a bash script that:
1. For `--lexer`: runs `ref-binary --emit-tokens <input>` as baseline; future: runs `fun-lexer --emit-tokens <input>`
2. For `--indent-filter`: runs reference pipeline tokens through the IndentFilter stub
3. For `--parser`: assumes tokens from real lexer, runs through generated parser only

For Phase 1, each component's isolated test simply calls the reference binary as a passthrough. The scaffold is wired; the replacement is done in Phases 2 and 3.

### Pattern 3: Rollback Target

**What:** A script that confirms the F# reference pipeline is intact and can regenerate its own output. Run this after any pipeline substitution to verify the 368/442 baseline is recoverable.

**Example:**
```bash
#!/usr/bin/env bash
# rollback.sh: restore F# pipeline and verify baseline
cd /Users/ohama/vibe-coding/LangThree
git checkout src/LangThree/Lexer.fs src/LangThree/Parser.fs 2>/dev/null || true
dotnet build src/LangThree/ -c Release --nologo -q
dotnet test tests/LangThree.Tests/ --no-build --nologo 2>&1 | tail -3
fslit tests/flt/ 2>&1 | tail -3
```

### Pattern 4: fslit Test Format

**What:** The .flt format used throughout LangThree's test suite. Phase 1 must understand this format because the comparison harness reads .flt corpus files as input sources.

**Format:**
```
// Test: <description>
// --- Command: /path/to/binary <flags> %input
// --- Input:
<source code here>
// --- Output:
<expected stdout>
// --- ExitCode: N          (optional)
// --- Stderr:              (optional)
<expected stderr>
```

Variables:
- `%input` — path to a temp file containing the Input section
- `%output` — path to a temp file containing the Output section
- `%S` — directory of the test file (useful for multi-file tests)

**When to use:** The harness in Phase 1 does not need to run fslit itself — it uses the `.flt` files only to extract `Input` sections as test fixtures. The comparison is run between reference binary and candidate binary on the same inputs, not between candidate output and `.flt` expected output.

### Anti-Patterns to Avoid

- **Comparing candidate output against .flt expected output:** The .flt files describe what the F# reference produces. Some of those test cases currently fail. The correct comparison is always reference-binary vs candidate-binary, not candidate vs hardcoded expected.
- **Running the generated pipeline before it passes 100% of side-by-side comparisons:** The Pitfalls research documents this as the highest-risk failure mode. The harness must enforce this invariant.
- **Putting harness scripts in LangThree's repository:** FunLexYacc is a separate project. The harness lives in `FunLexYacc/tests/harness/`. It calls the LangThree binary by absolute path.
- **Building generated .fun components into LangThree's fsproj:** Generated .fun files are executed by the LangThree interpreter, not compiled into LangThree.fsproj. There is no build wiring into LangThree for Phase 1.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Parsing .flt test files | custom .flt parser in bash | Read Input section with sed/awk between `Input:` and `Output:` markers | The format is simple enough for basic shell text processing; no parser needed |
| Token-level diff | custom token differ | `diff` on `--emit-tokens` output | formatToken in Format.fs already produces stable text like `NUMBER(42) IDENT(x) EQUALS` |
| AST-level diff | custom AST differ | `diff` on `--emit-ast` output | formatModule/formatAst already produces deterministic text representation |
| Isolation test runner | complex subprocess manager | simple bash wrapper around LangThree binary with different flags | LangThree's own `--emit-tokens`, `--emit-ast` flags already expose each pipeline stage separately |

**Key insight:** LangThree's existing `--emit-tokens` and `--emit-ast` modes are the side-by-side diff interface. The harness is trivially thin because the oracle already exists.

## Common Pitfalls

### Pitfall 1: Assuming All 442 Tests Currently Pass

**What goes wrong:** The harness is designed to compare candidate output against "passing tests" from fslit, not against the reference binary. When the baseline has 74 failures, this produces false negatives.

**Why it happens:** The ROADMAP.md says "641 passing tests" but the current reality is 368/442 fslit passing. The 74 failures are pre-existing LangThree bugs, not failures introduced by FunLexYacc work.

**How to avoid:** The harness always compares against the F# reference binary's actual output, not against .flt expected output. "Parity" means generated-output == reference-output, regardless of whether reference-output matches .flt expectations.

**Warning signs:** Harness showing "74 new failures" even before any code is written.

### Pitfall 2: Using Debug Build as Reference

**What goes wrong:** The .flt test files hardcode the Release binary path: `/Users/ohama/vibe-coding/LangThree/src/LangThree/bin/Release/net10.0/LangThree`. If the harness uses the Debug binary, comparisons may differ (though unlikely for LangThree's pure interpreter).

**How to avoid:** Always use the Release build as the reference oracle. Add a check at the start of `compare.sh` that the reference binary exists.

**Warning signs:** `compare.sh: reference binary not found` errors.

### Pitfall 3: Module Skeleton That Doesn't Type-Check

**What goes wrong:** Phase 1 creates `src/IndentFilter/IndentFilter.fun` as a skeleton. If this skeleton file is malformed LangThree syntax, it will crash the LangThree interpreter when later phases try to run it.

**Why it happens:** .fun files are LangThree source — they must parse and type-check, even as skeletons. An empty file, a `// comment only` file, or a file with wrong module syntax will fail.

**How to avoid:** The skeleton must be valid LangThree. The minimum valid skeleton for each module is:
```
module IndentFilter

let placeholder = 0
```
Or it must contain the actual type declarations for ADTs that later phases will fill in. Phase 1's Plan 01-01 must define what the skeleton contains.

**Warning signs:** `Error: Module not found` or parse error when any later phase tries to `open IndentFilter`.

### Pitfall 4: The FunLexYacc Project Has No Build System Yet

**What goes wrong:** Phase 1 creates `src/` structure but there is no `.fsproj`, no `Makefile`, no `dotnet.json`, and no `build` target — because FunLexYacc is a collection of .fun files interpreted by LangThree, not a compiled .NET project.

**Why it happens:** The instinct from F# projects is to create a .fsproj. But FunLexYacc is different: it is run with `langthree src/Main.fun`, not `dotnet run`.

**How to avoid:** Phase 1 does NOT create a .fsproj. The build target for FunLexYacc is a shell invocation: `langthree src/Main.fun <args>`. The rollback target is a shell script. Document this explicitly in a top-level README.md or build.sh.

**Warning signs:** Creating LangThree.fsproj in FunLexYacc or trying to `dotnet build` the .fun files.

### Pitfall 5: Harness Scripts Hardcoding Absolute Paths

**What goes wrong:** `compare.sh` hardcodes `/Users/ohama/vibe-coding/LangThree/src/LangThree/bin/Release/net10.0/LangThree` — the same path that the .flt files hardcode. If the LangThree repo moves or a CI machine has a different path, all scripts break.

**How to avoid:** Accept the reference binary path as a parameter, with a sensible default. The default can be the known local path but it should be overridable:
```bash
REF_BINARY="${LANGTHREE_BIN:-/Users/ohama/vibe-coding/LangThree/src/LangThree/bin/Release/net10.0/LangThree}"
```

**Warning signs:** Hardcoded paths in harness scripts with no override mechanism.

## Code Examples

Verified patterns from LangThree source:

### Extracting token output (reference oracle invocation)
```bash
# Reference: emit tokens for a source file
/Users/ohama/vibe-coding/LangThree/src/LangThree/bin/Release/net10.0/LangThree \
    --emit-tokens some-file.fun
# Output format (from Format.fs formatToken):
# NUMBER(42) PLUS NUMBER(1) EOF

# Reference: emit AST for a source file
/Users/ohama/vibe-coding/LangThree/src/LangThree/bin/Release/net10.0/LangThree \
    --emit-ast some-file.fun
# Output format:
# LetDecl ("x", Number 42)
```

### Valid minimum .fun module skeleton
```
module IndentFilter

let placeholder = 0
```
Source: LangThree Prelude/List.fun — modules contain flat let bindings, no wrapper braces.

### Extracting Input section from a .flt file for use as test fixture
```bash
# Extract input section: text between "--- Input:" and next "---" marker
awk '/^\/\/ --- Input:/{found=1; next} /^\/\/ ---/{found=0} found && !/^\/\//{print}' "$FLT_FILE"
```

### Minimal harness invocation for side-by-side comparison
```bash
#!/usr/bin/env bash
# compare-single.sh <ref-binary> <cand-binary> <input-file>
REF="$1"; CAND="$2"; INPUT="$3"

REF_TOKENS=$("$REF" --emit-tokens "$INPUT" 2>/dev/null)
CAND_TOKENS=$("$CAND" --emit-tokens "$INPUT" 2>/dev/null)

if [ "$REF_TOKENS" = "$CAND_TOKENS" ]; then
    echo "PASS (tokens)"
else
    echo "FAIL (tokens)"
    diff <(echo "$REF_TOKENS") <(echo "$CAND_TOKENS") | head -20
fi

REF_AST=$("$REF" --emit-ast "$INPUT" 2>/dev/null)
CAND_AST=$("$CAND" --emit-ast "$INPUT" 2>/dev/null)

if [ "$REF_AST" = "$CAND_AST" ]; then
    echo "PASS (ast)"
else
    echo "FAIL (ast)"
    diff <(echo "$REF_AST") <(echo "$CAND_AST") | head -20
fi
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| fslit tests hardcode binary paths | fslit uses `%input` variable | Built-in to fslit tool | Comparison harness can use `%input` pattern for test input extraction |
| Manual test comparison | `diff` on CLI output | LangThree ARCHITECTURE.md established | Harness is shell-composable |
| LangThree .fun files needed .fsproj | .fun files run via `langthree` interpreter | LangThree v1.8 current design | FunLexYacc needs no .fsproj |

**Deprecated/outdated:**
- Nothing relevant to Phase 1 is deprecated.

## Open Questions

1. **Exact test input corpus for the harness**
   - What we know: 442 .flt files exist in `LangThree/tests/flt/`, each with an Input section
   - What's unclear: Should the harness use ALL 442 .flt inputs, or a curated subset? Using all 442 means the harness runs on inputs that currently fail in the reference — this is fine if the comparison is ref vs candidate (not candidate vs expected).
   - Recommendation: Use all 442 .flt input sections. The harness's job is parity with the reference, not passing the .flt expected outputs.

2. **Harness invocation for `--expr` mode tests**
   - What we know: 42 of the 442 .flt tests use `--expr "..."` (expression mode), not `%input` (file mode). The expression string is embedded in the `--- Command:` line, not in `--- Input:`.
   - What's unclear: Should Phase 1's harness handle both modes, or focus on file mode only?
   - Recommendation: Phase 1 should handle both. The `--emit-tokens --expr "..."` and `--emit-ast --expr "..."` modes work identically via CLI flags; the harness just needs to parse the Command line to extract flags and substitution.

3. **FunLexYacc repo structure — does it need a dotnet project at all?**
   - What we know: FunLexYacc's runtime is the LangThree interpreter. No .NET compilation of .fun files is needed.
   - What's unclear: The FunLexYacc git repo currently has only `.claude/` and `.planning/` — no src/, no tests/ yet.
   - Recommendation: Phase 1 (Plan 01-01) creates `src/` and `tests/harness/` as plain directories. No .fsproj. A top-level `run.sh` invokes `langthree src/Main.fun`.

## Sources

### Primary (HIGH confidence)
- `/Users/ohama/vibe-coding/LangThree/src/LangThree/Program.fs` — CLI interface: `--emit-tokens`, `--emit-ast`, `--expr`, file modes; exact binary invocation patterns
- `/Users/ohama/vibe-coding/LangThree/src/LangThree/Format.fs` — `formatToken`, `formatAst`, `formatModule` output formats (the diff oracle)
- `/Users/ohama/vibe-coding/LangThree/src/LangThree/Cli.fs` — Argu CLI argument definitions; all flag names
- `/Users/ohama/vibe-coding/LangThree/tests/README.md` — authoritative test structure, category counts, run commands
- `/Users/ohama/vibe-coding/LangThree/ARCHITECTURE.md` — full pipeline: Lexer → IndentFilter → Parser, component responsibilities
- Direct fslit execution: `fslit /Users/ohama/vibe-coding/LangThree/tests/flt/` — 368/442 pass, 74 failing

### Secondary (MEDIUM confidence)
- `/Users/ohama/vibe-coding/FunLexYacc/.planning/research/PITFALLS.md` — bootstrapping pitfalls, rollback strategy (from prior project-level research)
- `/Users/ohama/vibe-coding/FunLexYacc/.planning/research/ARCHITECTURE.md` — src/ module layout, component boundaries
- `/Users/ohama/vibe-coding/LangThree/Prelude/List.fun` — minimal valid .fun module format (no wrapper, flat let bindings)

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — tools verified to exist and work; binary tested live
- Architecture: HIGH — LangThree source directly inspected; harness patterns derived from actual CLI interface
- Pitfalls: HIGH — pitfalls derived from direct inspection of current test state (74 failures), CLI behavior, and .fun module format requirements

**Research date:** 2026-03-23
**Valid until:** 2026-04-22 (stable; only changes if LangThree CLI changes or fslit format changes)
