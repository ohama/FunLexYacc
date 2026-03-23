# FunLexYacc

## What This Is

LangThree 언어로 작성된 lexer/parser 생성기(funlex, funyacc). 기존 F#의 fslex/fsyacc를 대체하여 LangThree가 자기 자신의 lexer/parser를 생성할 수 있게 하는 bootstrapping 도구다.

## Core Value

LangThree가 F# fslex/fsyacc 의존성 없이 자기 자신의 lexer와 parser를 생성할 수 있어야 한다.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] funlex: .fsl 호환 입력을 받아 .fun lexer 모듈을 생성
- [ ] funyacc: .fsy 호환 입력을 받아 .fun LALR(1) parser 모듈을 생성
- [ ] IndentFilter를 .fun으로 재작성하여 모듈로 제공
- [ ] 생성된 .fun 모듈이 LangThree의 module 시스템으로 다른 코드와 결합 가능
- [ ] 기존 LangThree의 641개 테스트를 모두 통과

### Out of Scope

- 성능 최적화 — 우선 동작 정확성에 집중, 최적화는 이후
- 새로운 문법 형식 정의 — .fsl/.fsy 호환 유지
- PEG/Packrat 등 다른 파싱 알고리즘 — LALR(1) 유지

## Context

- LangThree는 ML 계열 함수형 언어 (v1.8), F#(.NET 10)로 구현됨
- 현재 파이프라인: Source → Lexer(fslex) → IndentFilter → Parser(fsyacc/LALR(1)) → Elaborate → TypeCheck → Eval
- Lexer.fsl과 Parser.fsy가 FsLexYacc에 의존하고 있음
- IndentFilter는 F#로 작성된 토큰 필터 (NEWLINE(col) → INDENT/DEDENT/IN 변환)
- LangThree는 ADT, GADT, 패턴 매칭, 모듈 시스템, indentation syntax 지원
- 총 641개 테스트 (199 F# unit + 442 fslit integration)
- funlex/funyacc 자체를 LangThree(.fun)로 작성

## Constraints

- **입력 호환성**: .fsl/.fsy 형식과 호환되어야 함 — 기존 LangThree의 lexer/parser 정의를 그대로 사용하기 위해
- **출력 형식**: .fun 소스 파일을 생성하며 LangThree 모듈 시스템으로 결합 가능해야 함
- **구현 언어**: funlex/funyacc 자체가 LangThree(.fun)로 작성됨 — bootstrapping 목적
- **파싱 알고리즘**: LALR(1) — 기존 fsyacc와 동일

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| .fsl/.fsy 호환 문법 사용 | 기존 LangThree 정의 파일을 변경 없이 재사용 | — Pending |
| LALR(1) 파서 알고리즘 | 기존 fsyacc와 동일한 동작 보장 | — Pending |
| LangThree로 구현 | bootstrapping 목표 달성을 위해 | — Pending |
| LangThree 모듈 시스템으로 결합 | 별도 빌드 도구 없이 기존 인프라 활용 | — Pending |

---
*Last updated: 2026-03-23 after initialization*
