# FunLexYacc

LangThree 언어로 작성된 lexer/parser 생성기. 기존 F#의 fslex/fsyacc를 대체하여 LangThree가 자기 자신의 lexer/parser를 생성할 수 있게 하는 bootstrapping 도구.

## 구성

- **funlex** — `.fsl` 파일을 입력받아 `.fun` lexer 모듈을 생성 (Thompson's NFA → subset DFA → Hopcroft 최소화)
- **funyacc** — `.fsy` 파일을 입력받아 `.fun` LALR(1) parser 모듈을 생성 (DeRemer/Pennello)
- **IndentFilter** — NEWLINE(col) → INDENT/DEDENT/IN 토큰 변환 (.fun으로 재작성)

## 실행

FunLexYacc는 .NET 프로젝트가 아닙니다. 모든 `.fun` 파일은 LangThree 인터프리터로 실행됩니다.

```bash
./run.sh [args...]
```

LangThree 바이너리 경로는 `LANGTHREE_BIN` 환경변수로 변경 가능합니다.

## 프로젝트 구조

```
FunLexYacc/
├── src/
│   ├── IndentFilter/    # IndentFilter.fun — 토큰 필터
│   ├── FslSpec/         # .fsl 포맷 파서
│   ├── FsySpec/         # .fsy 포맷 파서
│   ├── Nfa/             # NFA 구성 (Thompson's construction)
│   ├── Dfa/             # DFA 변환 + Hopcroft 최소화
│   ├── Lalr/            # LALR(1) 테이블 생성
│   └── Main.fun         # CLI 진입점
├── tests/
│   ├── harness/         # 테스트 스크립트
│   │   ├── compare.sh       # side-by-side 비교 (토큰/AST)
│   │   ├── run-isolated.sh  # 컴포넌트별 독립 테스트
│   │   └── rollback.sh      # F# 파이프라인 복원 + 베이스라인 검증
│   └── fixtures/        # LangThree flt 테스트 (symlink)
└── run.sh               # LangThree 인터프리터 호출
```

## 테스트

```bash
# side-by-side 비교 (참조 바이너리 vs 후보 바이너리)
tests/harness/compare.sh <ref-binary> <cand-binary> <input-file>

# 컴포넌트별 독립 테스트
tests/harness/run-isolated.sh --lexer <input-file>
tests/harness/run-isolated.sh --indent-filter <input-file>
tests/harness/run-isolated.sh --parser <input-file>

# F# 파이프라인 복원 + 베이스라인 검증
tests/harness/rollback.sh
```

## 목표

LangThree의 컴파일 파이프라인에서 F# fslex/fsyacc 의존성을 완전히 제거:

```
현재:  Source → Lexer.fsl(fslex) → IndentFilter.fs → Parser.fsy(fsyacc) → ...
목표:  Source → Lexer.fun(funlex) → IndentFilter.fun → Parser.fun(funyacc) → ...
```

검증 기준: 기존 641개 테스트(199 unit + 442 integration) 전체 통과.
