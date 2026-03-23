# Requirements: FunLexYacc

**Defined:** 2026-03-23
**Core Value:** LangThree가 F# fslex/fsyacc 의존성 없이 자기 자신의 lexer와 parser를 생성할 수 있어야 한다.

## v1 Requirements

### Test Infrastructure

- [ ] **TEST-01**: Side-by-side 비교 하네스 — 생성된 .fun lexer/parser와 기존 F# 버전의 출력을 토큰/AST 단위로 비교
- [ ] **TEST-02**: 기존 641개 테스트를 생성된 코드에 대해 실행하여 패리티 확인
- [ ] **TEST-03**: 단계별 회귀 테스트 — lexer, IndentFilter, parser 각각 독립적으로 테스트

### funlex (Lexer Generator)

- [ ] **FLEX-01**: .fsl 포맷 파싱 — FsLexYacc 호환 입력 형식의 재귀 하강 파서
- [ ] **FLEX-02**: 정규식 → NFA 변환 (Thompson's construction)
- [ ] **FLEX-03**: NFA → DFA 변환 (subset construction)
- [ ] **FLEX-04**: Hopcroft DFA 최소화
- [ ] **FLEX-05**: 파라미터화된 규칙 지원 (tokenize, block_comment depth, read_indent col, read_string buf)
- [ ] **FLEX-06**: .fun lexer 모듈 코드 생성 — 전이 테이블 + longest-match 드라이버

### funyacc (Parser Generator)

- [ ] **FYAC-01**: .fsy 포맷 파싱 — FsLexYacc 호환 입력 형식의 재귀 하강 파서
- [ ] **FYAC-02**: LALR(1) 테이블 생성 — LR(0) 오토마톤 구축 + DeRemer/Pennello lookahead 전파
- [ ] **FYAC-03**: %left/%right/%nonassoc 우선순위/결합성 처리 — INFIXOP0-4 포함
- [ ] **FYAC-04**: 다중 시작 심볼 지원 (start: Ast.Expr, parseModule: Ast.Module)
- [ ] **FYAC-05**: .fun parser 모듈 코드 생성 — LALR 테이블 + 인라인 인터프리터 + 시맨틱 액션

### IndentFilter

- [ ] **IFLT-01**: 7개 SyntaxContext 전체 포팅 (TopLevel, InMatch, InTry, InLetDecl, InExprBlock, InModule, InFunctionApp)
- [ ] **IFLT-02**: NEWLINE(col) → INDENT/DEDENT/IN 토큰 변환 로직
- [ ] **IFLT-03**: JustSawMatch 단일행 match 케이스 특수 처리

### Code Generation

- [ ] **CGEN-01**: .fun 모듈 출력 — LangThree 모듈 시스템으로 다른 코드와 결합 가능
- [ ] **CGEN-02**: 시맨틱 액션 임베딩 — .fsl/.fsy의 { } 액션 블록을 생성 코드에 포함 (opaque 처리)
- [ ] **CGEN-03**: LALR 인터프리터 인라인 — Parser.fun에 파싱 런타임(스택 기반 shift/reduce 엔진) 포함

### Bootstrap

- [ ] **BOOT-01**: funlex/funyacc로 LangThree의 Lexer.fsl/Parser.fsy를 처리하여 Lexer.fun/Parser.fun 생성
- [ ] **BOOT-02**: 생성된 Lexer.fun + IndentFilter.fun + Parser.fun이 641개 테스트 전체 통과 (F# 버전과 동일 결과)
- [ ] **BOOT-03**: F# fslex/fsyacc를 생성된 .fun으로 완전 대체 (cutover)

## v2 Requirements

### Self-hosting

- **SELF-01**: funlex/funyacc가 자기 자신의 .fsl/.fsy 파서를 생성 (meta-bootstrapping)
- **SELF-02**: 에러 메시지 개선 — 파싱 에러 위치 및 기대 토큰 표시

### Optimization

- **OPT-01**: 등가 클래스(equivalence class) 기반 DFA 테이블 압축
- **OPT-02**: LALR 테이블 직렬화 (시작 시간 최적화)

### Tooling

- **TOOL-01**: 충돌 보고서 — shift/reduce, reduce/reduce 충돌 상세 보고
- **TOOL-02**: 그래프 시각화 — DFA 상태 다이어그램 출력

## Out of Scope

| Feature | Reason |
|---------|--------|
| GLR 파싱 | LangThree 문법은 LALR(1)로 충분; 불필요한 복잡성 |
| 에러 복구 (error token) | v1에서는 정확성에 집중 |
| Auto-AST 생성 | LangThree는 수동 Ast.fs를 사용; 자동 생성은 불필요 |
| Unicode 등가 클래스 | LangThree lexer가 ASCII 중심; v2에서 필요시 추가 |
| PEG/Packrat 파싱 | LALR(1) 호환 유지 |
| 새로운 문법 형식 | .fsl/.fsy 호환 유지 |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| TEST-01 | Phase 1 | Pending |
| TEST-02 | Phase 7 | Pending |
| TEST-03 | Phase 1 | Pending |
| FLEX-01 | Phase 3 | Pending |
| FLEX-02 | Phase 3 | Pending |
| FLEX-03 | Phase 3 | Pending |
| FLEX-04 | Phase 3 | Pending |
| FLEX-05 | Phase 3 | Pending |
| FLEX-06 | Phase 3 | Pending |
| FYAC-01 | Phase 4 | Pending |
| FYAC-02 | Phase 5 | Pending |
| FYAC-03 | Phase 5 | Pending |
| FYAC-04 | Phase 5 | Pending |
| FYAC-05 | Phase 6 | Pending |
| IFLT-01 | Phase 2 | Pending |
| IFLT-02 | Phase 2 | Pending |
| IFLT-03 | Phase 2 | Pending |
| CGEN-01 | Phase 3 | Pending |
| CGEN-02 | Phase 6 | Pending |
| CGEN-03 | Phase 6 | Pending |
| BOOT-01 | Phase 7 | Pending |
| BOOT-02 | Phase 7 | Pending |
| BOOT-03 | Phase 7 | Pending |

**Coverage:**
- v1 requirements: 23 total
- Mapped to phases: 23
- Unmapped: 0

---
*Requirements defined: 2026-03-23*
*Last updated: 2026-03-23 after roadmap creation*
