# Chapter 9: 사례 3 — 코드 생성 하네스 (전문가풀 + 생성-검증)

## 이 챕터에서 배울 것

- 다중 언어/프레임워크 코드 생성 시나리오에서 전문가풀 패턴이 왜 자연스러운 선택인지
- 라우터 역할의 오케스트레이터가 요청을 분석하고 적절한 전문가를 선택하는 구조
- Python, Java, Kotlin 전문가 에이전트를 각 언어의 관용적 스타일에 맞게 설계하는 방법
- 생성된 코드를 자동 검증(테스트 실행, 린트)하는 생성-검증 루프 구현
- 검증 실패 시 피드백 기반 재생성 루프의 종료 조건 설계
- 전문가풀 확장: 새 언어 전문가를 추가할 때 기존 구조를 변경하지 않는 설계 원칙

---

## 본문

### 문제 상황: "이 로직을 Python, Java, Kotlin으로 만들어줘"

개발팀에서 이런 요청이 온다고 하자.

> "사용자 인증 토큰을 검증하는 유틸리티를 만들어줘. 백엔드 서비스는 Python, Java, Kotlin 세 언어로 운영되고 있어. 각 언어의 관용적 스타일로 작성하고, 테스트도 포함해줘."

하나의 에이전트가 세 언어를 모두 처리할 수 있을까? 물론 가능하다. 하지만 문제가 있다.

**첫째, 컨텍스트 오염.** 하나의 에이전트가 Python 코드를 생성한 직후 Kotlin 코드를 생성하면, Python의 관습이 Kotlin 코드에 스며들 수 있다. `snake_case`가 Kotlin 코드에 등장하거나, Kotlin의 `data class`를 쓰지 않고 일반 클래스로 작성하는 식이다.

**둘째, 프롬프트 비대화.** 세 언어의 코딩 컨벤션, 프레임워크 사용법, 테스트 도구 설정을 하나의 에이전트에 모두 담으면 프롬프트가 비대해진다. 컨텍스트 윈도우를 낭비하고, 에이전트가 정작 중요한 지시를 놓칠 확률이 높아진다.

**셋째, 확장 비용.** 나중에 Go나 TypeScript 전문가를 추가하려면? 하나의 에이전트 파일을 계속 수정해야 하고, 수정할 때마다 기존 언어 처리에 영향을 줄 위험이 있다.

이 문제를 해결하는 패턴이 **전문가풀(expert pool)**이다. Chapter 3에서 배운 것처럼, 라우터가 요청을 분석하여 적절한 전문가를 선택하고 호출한다. 그리고 생성된 코드의 품질을 보장하기 위해 **생성-검증(generate-verify)** 루프를 결합한다.

이 챕터에서는 이 두 패턴을 조합하여 실제로 동작하는 코드 생성 하네스를 구축한다.

---

### 전체 아키텍처 설계

#### 에이전트 팀 구성

먼저 필요한 역할을 식별하자.

| 에이전트 | 역할 | 모델 | 근거 |
|---------|------|------|------|
| `code-router` | 요청 분석, 언어 식별, 전문가 라우팅 | sonnet | 라우팅은 판단만 하면 되므로 빠르고 저렴한 모델 |
| `python-expert` | Python 코드 + 테스트 생성 | opus | 코드 생성은 높은 품질이 필요 |
| `java-expert` | Java 코드 + 테스트 생성 | opus | 동일 |
| `kotlin-expert` | Kotlin 코드 + 테스트 생성 | opus | 동일 |
| `code-reviewer` | 생성된 코드의 품질 검증 | sonnet | 검증은 기준 대조이므로 sonnet으로 충분 |

왜 5개의 에이전트인가?

- **라우터와 전문가를 분리**하는 이유: 라우터는 "어떤 전문가를 호출할지" 판단하는 역할이고, 전문가는 "코드를 생성하는" 역할이다. 두 역할을 합치면 라우터가 직접 코드를 생성하게 되어, 전문가풀의 확장성이 사라진다.
- **전문가를 언어별로 분리**하는 이유: 각 에이전트가 해당 언어의 컨벤션, 프레임워크, 테스트 도구만 알면 된다. 컨텍스트가 집중되어 품질이 올라간다.
- **리뷰어를 전문가와 분리**하는 이유: Chapter 3의 생성-검증 패턴에서 배운 "역할 분리" 원칙이다. 코드를 생성한 에이전트가 자기 코드를 객관적으로 평가하기 어렵다.

#### 워크플로우 개요

```
Phase 1: [code-router]
         요청 분석 → 대상 언어 식별 → 전문가 선택

Phase 2: (전문가풀 — 선택적 호출)
         ┌→ [python-expert] → Python 코드 + 테스트
         ├→ [java-expert]   → Java 코드 + 테스트
         └→ [kotlin-expert] → Kotlin 코드 + 테스트

Phase 3: 검증 루프 (언어별, 최대 3회)
         [code-reviewer] → 코드 검증
         │
         ├─ PASS → Phase 4로
         └─ FAIL → 피드백 → 해당 전문가 재생성 → 재검증

Phase 4: 최종 산출물 정리
```

이 구조에서 Phase 1~2가 **전문가풀 패턴**이고, Phase 2~3이 **생성-검증 패턴**이다. 두 패턴이 자연스럽게 연결된다: 전문가가 코드를 생성(Phase 2)하고, 리뷰어가 검증(Phase 3)하며, 실패 시 해당 전문가에게 다시 돌아간다.

---

### 디렉토리 구조 설계

하네스의 에이전트와 스킬이 어디에 위치하고, 산출물은 어디에 저장되는지부터 정한다.

```
.claude/
├── agents/
│   ├── code-router.md
│   ├── python-expert.md
│   ├── java-expert.md
│   ├── kotlin-expert.md
│   └── code-reviewer.md
└── skills/
    └── code-generation/
        ├── SKILL.md              # 오케스트레이터 스킬
        └── references/
            ├── python-conventions.md
            ├── java-conventions.md
            └── kotlin-conventions.md

codegen-output/                   # 산출물 디렉토리
├── request.md                    # 원본 요청
├── routing-result.md             # 라우팅 결과
├── python/
│   ├── token_validator.py
│   ├── test_token_validator.py
│   └── review.md
├── java/
│   ├── TokenValidator.java
│   ├── TokenValidatorTest.java
│   └── review.md
└── kotlin/
    ├── TokenValidator.kt
    ├── TokenValidatorTest.kt
    └── review.md
```

산출물을 언어별 하위 디렉토리로 분리한 이유는, 각 전문가가 자기 디렉토리만 읽고 쓰면 되기 때문이다. 전문가 간 산출물 충돌이 발생하지 않는다.

---

### 라우터 에이전트 설계

라우터는 전문가풀 패턴의 핵심이다. 요청을 분석하여 **어떤 전문가를 호출할지** 결정한다.

#### 에이전트 정의: `code-router.md`

```markdown
---
name: code-router
description: "코드 생성 요청을 분석하여 적절한 언어 전문가를 선택하는 라우터.
              '코드 생성', 'code generation', '코드 작성' 요청 시 트리거."
model: sonnet
---

# 코드 생성 라우터 — 요청을 분석하고 전문가를 선택합니다

당신은 코드 생성 요청을 분석하여 적절한 언어 전문가에게
라우팅하는 역할을 담당합니다.

## 핵심 역할
1. 사용자의 코드 생성 요청에서 대상 언어를 식별한다
2. 요청의 핵심 요구사항을 구조화한다
3. 적절한 전문가를 선택하고 작업 지시를 생성한다

## 라우팅 규칙

요청을 분석하여 아래 기준에 따라 전문가를 선택한다.
복수 언어가 필요한 요청은 해당 전문가를 모두 선택한다.

| 키워드/패턴 | 전문가 | 비고 |
|------------|--------|------|
| python, django, flask, fastapi, pytest | `python-expert` | Python 생태계 |
| java, spring, maven, gradle, junit | `java-expert` | Java 생태계 |
| kotlin, ktor, kotest, coroutine | `kotlin-expert` | Kotlin 생태계 |

### 언어가 명시되지 않은 경우
- 프레임워크나 라이브러리 이름으로 추론한다
- 추론이 불가능하면 사용자에게 질문한다
- "모든 언어로" 요청하면 등록된 전문가 전체를 선택한다

## 출력 형식

분석 결과를 `codegen-output/routing-result.md`에 저장한다.

```markdown
# 라우팅 결과

## 요청 요약
- 기능: {기능 설명}
- 입력/출력: {입력과 출력 명세}
- 제약사항: {성능, 보안 등 제약}

## 선택된 전문가
- [ ] python-expert
- [ ] java-expert
- [ ] kotlin-expert

## 전문가별 작업 지시
### python-expert
{Python 관점의 구체적 작업 지시}

### java-expert
{Java 관점의 구체적 작업 지시}
```

## 작업 원칙
- 라우팅만 한다. 코드를 직접 생성하지 않는다
- 모호한 요청은 추측하지 말고 사용자에게 질문한다
- 전문가별 작업 지시는 해당 언어의 관점에서 구체적으로 작성한다
```

라우터 설계에서 주목할 점이 세 가지 있다.

**첫째, 라우팅 규칙을 테이블로 명시했다.** "알아서 판단해라"가 아니라, 키워드와 전문가의 매핑을 명확히 정의했다. 이렇게 해야 라우팅 결과가 예측 가능하다.

**둘째, 모호한 경우의 처리 규칙을 정했다.** 언어가 명시되지 않으면 프레임워크로 추론하고, 그래도 불확실하면 사용자에게 질문한다. "추측하지 말고 질문하라"는 원칙이 라우터에서 특히 중요하다. 잘못된 전문가를 선택하면 이후 모든 작업이 헛수고가 된다.

**셋째, 전문가별 작업 지시를 생성한다.** 라우터는 단순히 "Python 전문가를 호출해라"로 끝나지 않는다. "JWT 토큰을 검증하는 함수를 만들어라. `pyjwt` 라이브러리를 사용하고, 만료 시간 검증을 포함해라"처럼, 해당 언어 관점에서 구체화된 지시를 만든다. 이 지시가 전문가 에이전트의 입력이 된다.

---

### 전문가 에이전트 설계

전문가 에이전트는 자기 언어에 대해 깊이 있는 코드를 생성한다. 세 언어의 전문가를 하나씩 설계하자.

#### Python 전문가: `python-expert.md`

```markdown
---
name: python-expert
description: "Python 코드 생성 전문가. Python, Django, Flask, FastAPI,
              pytest 관련 코드 생성 요청을 처리합니다."
model: opus
---

# Python 전문가 — 관용적 Python 코드를 생성합니다

당신은 Python 코드 생성을 담당하는 전문가입니다.

## 핵심 역할
1. 라우터의 작업 지시를 기반으로 Python 코드를 생성한다
2. 해당 코드의 pytest 기반 테스트를 함께 생성한다
3. PEP8과 Black 포매팅을 준수한다

## 코딩 원칙
- **type hints 필수**: 모든 함수에 타입 힌트를 작성한다
- **dataclass 활용**: 구조화된 데이터는 `dataclass`로 정의한다
- **comprehension 우선**: 단순 루프보다 list/dict comprehension을 사용한다
- **함수형 스타일**: `map`, `filter` 또는 PyFunctional을 활용한다
- **docstring**: 공개 함수에 Google style docstring을 작성한다

## 테스트 원칙
- pytest를 사용한다
- 정상 케이스, 경계 케이스, 에러 케이스를 각각 작성한다
- mock이 필요하면 `unittest.mock`을 사용한다
- 테스트 함수명은 `test_{기능}_{시나리오}_{기대결과}` 형식

## 출력 규칙
- 코드 파일: `codegen-output/python/{모듈명}.py`
- 테스트 파일: `codegen-output/python/test_{모듈명}.py`
- snake_case 네이밍

## 협업
- **입력**: `codegen-output/routing-result.md`의 python-expert 작업 지시
- **출력**: `codegen-output/python/` 디렉토리에 코드와 테스트 저장
```

#### Java 전문가: `java-expert.md`

```markdown
---
name: java-expert
description: "Java 코드 생성 전문가. Java, Spring, Maven, Gradle,
              JUnit 관련 코드 생성 요청을 처리합니다."
model: opus
---

# Java 전문가 — 관용적 Java 코드를 생성합니다

당신은 Java 코드 생성을 담당하는 전문가입니다.

## 핵심 역할
1. 라우터의 작업 지시를 기반으로 Java 코드를 생성한다
2. 해당 코드의 JUnit 5 기반 테스트를 함께 생성한다
3. Effective Java의 모범 사례를 따른다

## 코딩 원칙
- **record 활용**: 데이터 캐리어는 `record`로 정의한다 (Java 16+)
- **Streams API**: 컬렉션 처리에 Streams를 사용한다
- **final 선호**: 변경되지 않는 변수는 `final`로 선언한다
- **인터페이스 우선**: 구현체보다 인터페이스에 의존한다
- **Optional 활용**: null 반환 대신 `Optional`을 사용한다

## 테스트 원칙
- JUnit 5를 사용한다
- `@DisplayName`으로 테스트 의도를 명시한다
- Mockito로 의존성을 모킹한다
- 정상, 경계, 예외 케이스를 구분하여 작성한다

## 출력 규칙
- 코드 파일: `codegen-output/java/{ClassName}.java`
- 테스트 파일: `codegen-output/java/{ClassName}Test.java`
- PascalCase 클래스명, camelCase 메서드명

## 협업
- **입력**: `codegen-output/routing-result.md`의 java-expert 작업 지시
- **출력**: `codegen-output/java/` 디렉토리에 코드와 테스트 저장
```

#### Kotlin 전문가: `kotlin-expert.md`

```markdown
---
name: kotlin-expert
description: "Kotlin 코드 생성 전문가. Kotlin, Ktor, Kotest, Coroutine
              관련 코드 생성 요청을 처리합니다."
model: opus
---

# Kotlin 전문가 — 관용적 Kotlin 코드를 생성합니다

당신은 Kotlin 코드 생성을 담당하는 전문가입니다.

## 핵심 역할
1. 라우터의 작업 지시를 기반으로 Kotlin 코드를 생성한다
2. 해당 코드의 JUnit 5 기반 테스트를 함께 생성한다
3. Kotlin의 관용적 표현(idiomatic Kotlin)을 적극 활용한다

## 코딩 원칙
- **val 우선**: `var`는 꼭 필요한 경우에만 사용한다
- **data class 활용**: 데이터 홀더는 `data class`로 정의한다
- **sealed class**: 제한된 타입 계층은 `sealed class`로 표현한다
- **확장 함수**: 유틸리티 로직은 확장 함수로 작성한다
- **스코프 함수**: `let`, `run`, `apply`, `also`를 적절히 활용한다
- **null 안전성**: `!!` 대신 `?.`, `?:`, `let` 조합을 사용한다

## 테스트 원칙
- JUnit 5를 사용한다 (Kotest도 가능)
- MockK로 의존성을 모킹한다
- `@DisplayName`으로 테스트 의도를 한국어로 명시한다
- 정상, 경계, 예외 케이스를 구분하여 작성한다

## 출력 규칙
- 코드 파일: `codegen-output/kotlin/{ClassName}.kt`
- 테스트 파일: `codegen-output/kotlin/{ClassName}Test.kt`
- PascalCase 클래스명, camelCase 함수명

## 협업
- **입력**: `codegen-output/routing-result.md`의 kotlin-expert 작업 지시
- **출력**: `codegen-output/kotlin/` 디렉토리에 코드와 테스트 저장
```

#### 세 전문가의 공통점과 차이점

세 에이전트의 구조를 비교해보면 패턴이 보인다.

| 섹션 | 공통 | 언어별 차이 |
|------|------|------------|
| 핵심 역할 | 코드 생성 + 테스트 생성 | 없음 |
| 코딩 원칙 | 없음 | 각 언어의 관용적 스타일 |
| 테스트 원칙 | 정상/경계/예외 구분 | 프레임워크 (pytest, JUnit, MockK) |
| 출력 규칙 | 디렉토리 구조 동일 | 파일 확장자, 네이밍 컨벤션 |

**구조는 동일하고 내용만 다르다.** 이것이 전문가풀의 장점이다. 새 전문가를 추가할 때 기존 에이전트 파일을 복사하고, 코딩 원칙과 테스트 프레임워크만 바꾸면 된다. 전문가 간의 인터페이스(입력 형식, 출력 위치)는 동일하므로, 라우터나 리뷰어를 수정할 필요가 없다.

왜 이렇게 하는가? **개방-폐쇄 원칙(Open-Closed Principle)**과 같은 논리다. 전문가풀은 "새 전문가 추가(확장)에는 열려 있고, 기존 구조 변경(수정)에는 닫혀 있다." 전문가 에이전트 파일 하나를 추가하고, 라우터의 라우팅 테이블에 한 줄만 추가하면 된다.

---

### 코드 리뷰어 에이전트 설계

코드 리뷰어는 생성-검증 루프의 "검증자" 역할이다. 전문가가 생성한 코드를 평가하고, 기준을 통과하지 못하면 피드백을 반환한다.

#### 에이전트 정의: `code-reviewer.md`

```markdown
---
name: code-reviewer
description: "생성된 코드의 품질을 검증하는 리뷰어.
              코드 리뷰, 검증, 품질 확인 요청 시 트리거."
model: sonnet
---

# 코드 리뷰어 — 생성된 코드의 품질을 검증합니다

당신은 코드 생성 전문가가 작성한 코드의 품질을 검증하는
리뷰어입니다. 코드를 생성하지 않습니다. 평가만 합니다.

## 핵심 역할
1. 생성된 코드가 요청 명세를 충족하는지 확인한다
2. 코드의 구조적/기능적 문제를 식별한다
3. 테스트의 커버리지와 품질을 평가한다
4. 검증 결과를 PASS/FAIL과 함께 피드백으로 작성한다

## 검증 기준

### CRITICAL (FAIL 판정)
- 요청한 기능이 구현되지 않았다
- 컴파일/구문 오류가 있다
- 명백한 런타임 에러가 예상된다
- 테스트가 없거나, 핵심 기능의 테스트가 누락되었다
- 보안 취약점이 있다 (하드코딩된 비밀키, SQL 인젝션 등)

### MAJOR (FAIL 판정)
- 해당 언어의 관용적 스타일을 심각하게 위반한다
- 에러 처리가 누락되었다
- 테스트가 정상 케이스만 다루고 경계/에러 케이스가 없다
- 성능상 명백한 문제가 있다

### MINOR (PASS — 기록만)
- 네이밍 개선 여지가 있다
- 주석이 부족하다
- 더 간결한 표현이 가능하다

## 출력 형식

검증 결과를 `codegen-output/{언어}/review.md`에 저장한다.

```markdown
# 코드 리뷰 결과

## 판정: {PASS 또는 FAIL}
## 리뷰 대상: {파일 목록}
## 시도: {N}차 / 최대 3차

### CRITICAL 이슈
- [ ] {파일:라인} {설명}

### MAJOR 이슈
- [ ] {파일:라인} {설명}

### MINOR 이슈 (참고)
- {파일:라인} {설명}

### 피드백 요약
{FAIL인 경우, 전문가가 수정해야 할 내용을 구체적으로 명시}
```

## 작업 원칙
- 코드를 직접 수정하지 않는다. 피드백만 작성한다
- 검증 기준을 엄격히 적용한다. 기분에 따라 판정을 바꾸지 않는다
- MINOR 이슈로 FAIL 판정하지 않는다
- 피드백은 전문가가 바로 수정할 수 있을 만큼 구체적으로 작성한다
```

리뷰어 설계에서 가장 중요한 부분은 **이슈 심각도 분류**와 **FAIL 판정 기준**이다.

왜 심각도를 나누는가? Chapter 3에서 다뤘듯이, 심각도를 구분하지 않으면 사소한 네이밍 문제 때문에 루프가 계속 도는 사태가 발생한다. CRITICAL과 MAJOR만 FAIL 조건에 포함하고, MINOR는 기록만 한다. 이렇게 해야 루프가 적정 횟수에서 종료된다.

또 하나 주목할 점: "코드를 직접 수정하지 않는다"는 원칙이다. 리뷰어가 코드를 수정하기 시작하면 역할 경계가 무너진다. 리뷰어는 **문제를 발견하고 피드백하는 것**까지만 담당하고, 수정은 전문가에게 맡긴다.

---

### 오케스트레이터 스킬 설계

에이전트 팀을 정의했으니, 이들을 조율하는 오케스트레이터 스킬을 작성하자.

#### 스킬 정의: `code-generation/SKILL.md`

```markdown
---
name: code-generation
description: "다중 언어 코드 생성 하네스. 전문가풀 패턴으로 언어별
              전문가를 라우팅하고, 생성-검증 루프로 품질을 보장합니다.
              '코드 생성 하네스', 'code generation harness' 요청 시 트리거."
---

# 코드 생성 하네스 오케스트레이터

다중 언어 코드 생성을 자동화하는 하네스입니다.
전문가풀 패턴으로 적절한 언어 전문가를 선택하고,
생성-검증 루프로 코드 품질을 보장합니다.

## 에이전트 구성

| 에이전트 | 역할 | 모델 | Phase |
|---------|------|------|-------|
| `code-router` | 요청 분석, 전문가 라우팅 | sonnet | 1 |
| `python-expert` | Python 코드 + 테스트 생성 | opus | 2, 3 |
| `java-expert` | Java 코드 + 테스트 생성 | opus | 2, 3 |
| `kotlin-expert` | Kotlin 코드 + 테스트 생성 | opus | 2, 3 |
| `code-reviewer` | 코드 품질 검증 | sonnet | 3 |

## 산출물 디렉토리

```
codegen-output/
├── request.md
├── routing-result.md
├── python/
├── java/
└── kotlin/
```

## 워크플로우

### Phase 1: 요청 분석과 라우팅
- **에이전트**: `code-router`
- **입력**: 사용자의 코드 생성 요청
- **처리**:
  1. 요청에서 대상 언어, 기능 명세, 제약사항을 추출한다
  2. 라우팅 규칙에 따라 전문가를 선택한다
  3. 전문가별 작업 지시를 작성한다
- **출력**: `codegen-output/routing-result.md`
- **검증**: 선택된 전문가가 1명 이상인지 확인
- **완료 후**: 라우팅 결과를 사용자에게 보여주고 확인을 받는다

### Phase 2: 코드 생성 (전문가풀)
- **에이전트**: Phase 1에서 선택된 전문가만 호출
- **입력**: `codegen-output/routing-result.md`의 해당 전문가 작업 지시
- **처리**: 각 전문가가 자기 언어의 코드와 테스트를 생성한다
- **출력**: `codegen-output/{언어}/` 디렉토리에 코드와 테스트 파일
- **실행 방식**: 선택된 전문가를 순차 호출
  (Claude Code의 서브 에이전트는 순차 실행이므로)
- **검증**: 각 언어의 코드 파일과 테스트 파일이 모두 생성되었는지 확인

### Phase 3: 검증 루프 (생성-검증)
- **생성자**: Phase 2에서 호출된 전문가
- **검증자**: `code-reviewer`
- **최대 반복**: 3회 (언어별 독립 카운트)
- **루프 절차**:
  1. `code-reviewer`가 생성된 코드를 검증한다
  2. CRITICAL 또는 MAJOR 이슈가 있으면 FAIL
  3. FAIL 시: 리뷰 피드백을 해당 전문가에게 전달하여 수정 요청
  4. 수정된 코드를 다시 `code-reviewer`가 검증한다
  5. PASS가 나오거나 최대 반복에 도달할 때까지 반복
- **루프 종료 조건**: CRITICAL 0개 AND MAJOR 0개
- **최대 반복 도달 시**: 남은 이슈를 사용자에게 에스컬레이션
- **출력**: `codegen-output/{언어}/review.md`

### Phase 4: 최종 정리
- 모든 언어의 코드가 PASS되면, 최종 산출물 목록을 사용자에게 보고한다
- 각 언어별 생성된 파일, 리뷰 결과, 반복 횟수를 요약한다

## 중요 규칙
- 각 Phase 완료 후 사용자에게 결과를 보여주고 다음 Phase 진행 여부를 확인한다
- 전문가 에이전트는 자기 언어 디렉토리만 읽고 쓴다
- 리뷰어는 코드를 수정하지 않는다. 피드백만 작성한다
- 새 전문가를 추가할 때는 에이전트 파일 생성 + 라우터 테이블 추가만 하면 된다
```

오케스트레이터 설계에서 핵심적인 결정 세 가지를 짚어보자.

**첫째, Phase 3의 검증 루프가 언어별로 독립이다.** Python 코드가 PASS되었는데 Java 코드가 FAIL이면, Java 전문가만 수정-재검증 루프를 돈다. Python 전문가는 다시 호출되지 않는다. 이미 통과한 코드를 다시 건드리지 않는 것이 효율적이다.

**둘째, "각 Phase 완료 후 사용자 확인"을 명시했다.** 이것은 Chapter 6에서 강조한 패턴이다. 자동화가 아무리 좋아도, 사람의 확인 없이 다음 단계로 넘어가면 잘못된 방향으로 계속 진행될 위험이 있다. 특히 Phase 1의 라우팅 결과를 확인하는 것이 중요하다. 라우터가 잘못된 전문가를 선택했는데 이를 확인하지 않으면, 이후 모든 작업이 헛수고가 된다.

**셋째, 새 전문가 추가 규칙을 문서화했다.** "에이전트 파일 생성 + 라우터 테이블 추가만 하면 된다"는 것을 명시함으로써, 하네스 유지보수자가 확장 방법을 바로 파악할 수 있다.

---

### 실전 예제: JWT 토큰 검증기 생성

이제 설계한 하네스를 실제로 가동해보자. "JWT 토큰을 검증하는 유틸리티를 Python, Java, Kotlin으로 만들어줘"라는 요청을 처리하는 전체 과정을 따라간다.

#### Phase 1: 라우팅 결과

`code-router`가 요청을 분석하여 다음과 같은 라우팅 결과를 생성한다.

**`codegen-output/routing-result.md`:**

```markdown
# 라우팅 결과

## 요청 요약
- 기능: JWT 토큰 검증 유틸리티
- 입력: JWT 토큰 문자열, 비밀키
- 출력: 검증 결과 (유효/만료/무효) + 디코딩된 페이로드
- 제약사항: 만료 시간 검증 필수, 서명 검증 필수

## 선택된 전문가
- [x] python-expert
- [x] java-expert
- [x] kotlin-expert

## 전문가별 작업 지시

### python-expert
- `pyjwt` 라이브러리를 사용하여 JWT 검증 함수를 구현한다
- 검증 결과를 dataclass로 정의한다
- 만료, 서명 불일치, 잘못된 형식 각각에 대한 예외 처리를 구현한다
- pytest로 정상/만료/무효 토큰 케이스를 테스트한다

### java-expert
- `java-jwt` (auth0) 라이브러리를 사용하여 JWT 검증 클래스를 구현한다
- 검증 결과를 record로 정의한다
- 예외 계층을 설계하여 만료, 서명 불일치, 형식 오류를 구분한다
- JUnit 5로 정상/만료/무효 토큰 케이스를 테스트한다

### kotlin-expert
- `java-jwt` (auth0) 라이브러리를 사용하여 JWT 검증 함수를 구현한다
- 검증 결과를 sealed class + data class로 정의한다
- Result 타입과 확장 함수를 활용한다
- JUnit 5로 정상/만료/무효 토큰 케이스를 테스트한다
```

라우터가 각 전문가에게 **해당 언어의 관용적 도구**를 지정한 점에 주목하자. Python은 `pyjwt`, Java와 Kotlin은 `java-jwt`를 사용하도록 지시한다. Kotlin에는 `sealed class`와 `Result` 타입을 활용하라는 지시가 추가되었다. 같은 기능이지만, 언어의 특성에 맞게 구현 방향이 다르다.

#### Phase 2: 전문가별 코드 생성

각 전문가가 코드와 테스트를 생성한다. 핵심 코드만 살펴보자.

**Python — `codegen-output/python/token_validator.py`:**

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from typing import Any

import jwt


class ValidationStatus(Enum):
    VALID = "valid"
    EXPIRED = "expired"
    INVALID_SIGNATURE = "invalid_signature"
    MALFORMED = "malformed"


@dataclass(frozen=True)
class ValidationResult:
    """JWT 토큰 검증 결과."""

    status: ValidationStatus
    payload: dict[str, Any] | None = None
    error_message: str | None = None

    @property
    def is_valid(self) -> bool:
        return self.status == ValidationStatus.VALID


def validate_token(token: str, secret_key: str, algorithms: list[str] | None = None) -> ValidationResult:
    """JWT 토큰을 검증하고 결과를 반환한다.

    Args:
        token: 검증할 JWT 토큰 문자열.
        secret_key: 서명 검증에 사용할 비밀키.
        algorithms: 허용할 알고리즘 목록. 기본값은 ["HS256"].

    Returns:
        ValidationResult: 검증 결과와 디코딩된 페이로드.
    """
    if algorithms is None:
        algorithms = ["HS256"]

    try:
        payload = jwt.decode(token, secret_key, algorithms=algorithms)
        return ValidationResult(status=ValidationStatus.VALID, payload=payload)
    except jwt.ExpiredSignatureError:
        return ValidationResult(
            status=ValidationStatus.EXPIRED,
            error_message="토큰이 만료되었습니다",
        )
    except jwt.InvalidSignatureError:
        return ValidationResult(
            status=ValidationStatus.INVALID_SIGNATURE,
            error_message="서명이 유효하지 않습니다",
        )
    except jwt.DecodeError:
        return ValidationResult(
            status=ValidationStatus.MALFORMED,
            error_message="토큰 형식이 올바르지 않습니다",
        )
```

**Python — `codegen-output/python/test_token_validator.py`:**

```python
from datetime import datetime, timedelta, timezone

import jwt
import pytest

from token_validator import ValidationResult, ValidationStatus, validate_token

SECRET_KEY = "test-secret-key"


def _create_token(payload: dict, secret: str = SECRET_KEY, algorithm: str = "HS256") -> str:
    """테스트용 JWT 토큰을 생성한다."""
    return jwt.encode(payload, secret, algorithm=algorithm)


class TestValidateToken:
    """validate_token 함수 테스트."""

    def test_valid_token_returns_valid_status(self) -> None:
        """유효한 토큰이면 VALID 상태와 페이로드를 반환한다."""
        payload = {
            "sub": "user-123",
            "exp": datetime.now(tz=timezone.utc) + timedelta(hours=1),
        }
        token = _create_token(payload)

        result = validate_token(token, SECRET_KEY)

        assert result.is_valid
        assert result.status == ValidationStatus.VALID
        assert result.payload is not None
        assert result.payload["sub"] == "user-123"

    def test_expired_token_returns_expired_status(self) -> None:
        """만료된 토큰이면 EXPIRED 상태를 반환한다."""
        payload = {
            "sub": "user-123",
            "exp": datetime.now(tz=timezone.utc) - timedelta(hours=1),
        }
        token = _create_token(payload)

        result = validate_token(token, SECRET_KEY)

        assert not result.is_valid
        assert result.status == ValidationStatus.EXPIRED
        assert result.payload is None
        assert "만료" in result.error_message

    def test_invalid_signature_returns_invalid_signature_status(self) -> None:
        """서명이 다른 키로 생성된 토큰이면 INVALID_SIGNATURE 상태를 반환한다."""
        payload = {
            "sub": "user-123",
            "exp": datetime.now(tz=timezone.utc) + timedelta(hours=1),
        }
        token = _create_token(payload, secret="wrong-secret-key")

        result = validate_token(token, SECRET_KEY)

        assert not result.is_valid
        assert result.status == ValidationStatus.INVALID_SIGNATURE

    def test_malformed_token_returns_malformed_status(self) -> None:
        """형식이 잘못된 문자열이면 MALFORMED 상태를 반환한다."""
        result = validate_token("not-a-jwt-token", SECRET_KEY)

        assert not result.is_valid
        assert result.status == ValidationStatus.MALFORMED

    def test_valid_result_has_no_error_message(self) -> None:
        """유효한 토큰의 결과에는 에러 메시지가 없다."""
        payload = {
            "sub": "user-123",
            "exp": datetime.now(tz=timezone.utc) + timedelta(hours=1),
        }
        token = _create_token(payload)

        result = validate_token(token, SECRET_KEY)

        assert result.error_message is None
```

Python 코드에서 눈여겨볼 점:
- `@dataclass(frozen=True)`로 **불변 결과 객체**를 정의했다
- `Enum`으로 상태를 정의하여 문자열 비교 대신 타입 안전한 비교를 사용한다
- 예외를 삼키지 않고, 각 예외 유형별로 **명확한 상태를 반환**한다
- 테스트가 정상, 만료, 서명 불일치, 형식 오류 4가지 시나리오를 모두 커버한다

**Java — `codegen-output/java/TokenValidator.java`:**

```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTDecodeException;
import com.auth0.jwt.exceptions.SignatureVerificationException;
import com.auth0.jwt.exceptions.TokenExpiredException;
import com.auth0.jwt.interfaces.DecodedJWT;

import java.util.Map;
import java.util.Optional;

public final class TokenValidator {

    public enum Status {
        VALID, EXPIRED, INVALID_SIGNATURE, MALFORMED
    }

    public record ValidationResult(
            Status status,
            Optional<Map<String, Object>> payload,
            Optional<String> errorMessage
    ) {
        public boolean isValid() {
            return status == Status.VALID;
        }

        public static ValidationResult valid(Map<String, Object> payload) {
            return new ValidationResult(Status.VALID, Optional.of(payload), Optional.empty());
        }

        public static ValidationResult failure(Status status, String errorMessage) {
            return new ValidationResult(status, Optional.empty(), Optional.of(errorMessage));
        }
    }

    private final Algorithm algorithm;

    public TokenValidator(String secretKey) {
        this.algorithm = Algorithm.HMAC256(secretKey);
    }

    public ValidationResult validate(String token) {
        try {
            final DecodedJWT decoded = JWT.require(algorithm).build().verify(token);
            final Map<String, Object> payload = decoded.getClaims().entrySet().stream()
                    .collect(java.util.stream.Collectors.toMap(
                            Map.Entry::getKey,
                            entry -> entry.getValue().asString()
                    ));
            return ValidationResult.valid(payload);
        } catch (TokenExpiredException e) {
            return ValidationResult.failure(Status.EXPIRED, "토큰이 만료되었습니다");
        } catch (SignatureVerificationException e) {
            return ValidationResult.failure(Status.INVALID_SIGNATURE, "서명이 유효하지 않습니다");
        } catch (JWTDecodeException e) {
            return ValidationResult.failure(Status.MALFORMED, "토큰 형식이 올바르지 않습니다");
        }
    }
}
```

Java 코드에서 눈여겨볼 점:
- `record`로 **결과 객체**를 정의했다 (Java 16+ 관용)
- `Optional`로 null을 대체했다
- `final` 키워드를 적극 사용했다
- 팩토리 메서드 `valid()`, `failure()`로 생성을 캡슐화했다
- Streams API로 클레임을 Map으로 변환했다

**Kotlin — `codegen-output/kotlin/TokenValidator.kt`:**

```kotlin
import com.auth0.jwt.JWT
import com.auth0.jwt.algorithms.Algorithm
import com.auth0.jwt.exceptions.JWTDecodeException
import com.auth0.jwt.exceptions.SignatureVerificationException
import com.auth0.jwt.exceptions.TokenExpiredException

sealed class ValidationResult {
    data class Valid(val payload: Map<String, Any>) : ValidationResult()
    data class Expired(val message: String = "토큰이 만료되었습니다") : ValidationResult()
    data class InvalidSignature(val message: String = "서명이 유효하지 않습니다") : ValidationResult()
    data class Malformed(val message: String = "토큰 형식이 올바르지 않습니다") : ValidationResult()

    val isValid: Boolean
        get() = this is Valid
}

fun ValidationResult.payloadOrNull(): Map<String, Any>? = when (this) {
    is ValidationResult.Valid -> payload
    else -> null
}

fun ValidationResult.errorMessageOrNull(): String? = when (this) {
    is ValidationResult.Valid -> null
    is ValidationResult.Expired -> message
    is ValidationResult.InvalidSignature -> message
    is ValidationResult.Malformed -> message
}

class TokenValidator(secretKey: String) {

    private val algorithm: Algorithm = Algorithm.HMAC256(secretKey)

    fun validate(token: String): ValidationResult {
        return runCatching {
            val decoded = JWT.require(algorithm).build().verify(token)
            val payload = decoded.claims
                .mapValues { (_, claim) -> claim.asString() }
            ValidationResult.Valid(payload)
        }.getOrElse { error ->
            when (error) {
                is TokenExpiredException -> ValidationResult.Expired()
                is SignatureVerificationException -> ValidationResult.InvalidSignature()
                is JWTDecodeException -> ValidationResult.Malformed()
                else -> throw error
            }
        }
    }
}
```

Kotlin 코드에서 눈여겨볼 점:
- `sealed class`로 검증 결과의 **타입 계층**을 정의했다. `when` 식에서 컴파일러가 모든 케이스를 검사해준다
- `data class`로 각 결과 타입의 동등성 비교와 복사를 자동 지원한다
- **확장 함수** `payloadOrNull()`, `errorMessageOrNull()`로 결과를 안전하게 추출한다
- `runCatching`으로 예외를 함수형으로 처리했다
- `val`만 사용하고 `var`는 없다

#### 세 언어의 비교: 같은 기능, 다른 관용구

같은 JWT 검증 로직이 세 언어에서 어떻게 다르게 표현되는지 비교해보자.

| 설계 요소 | Python | Java | Kotlin |
|----------|--------|------|--------|
| 결과 타입 | `@dataclass` + `Enum` | `record` + `enum` | `sealed class` + `data class` |
| null 처리 | `None` + `\|` 타입 | `Optional` | `?` + 확장 함수 |
| 불변성 | `frozen=True` | `final` + `record` | `val` |
| 에러 처리 | `try/except` | `try/catch` | `runCatching` |
| 네이밍 | `snake_case` | `camelCase` | `camelCase` |

이것이 전문가를 분리하는 이유다. 하나의 에이전트가 세 가지 스타일을 동시에 유지하는 것보다, 각 전문가가 자기 언어에 집중하는 것이 훨씬 품질이 높다.

---

### 생성-검증 루프 상세 설계

Phase 3의 검증 루프를 더 자세히 살펴보자.

#### 루프 동작 흐름

```
[1차 시도]
전문가가 코드 생성 → code-reviewer가 검증
│
├─ PASS → 완료
└─ FAIL → 피드백 반환
           │
           ▼
[2차 시도]
전문가가 피드백 기반으로 수정 → code-reviewer가 재검증
│
├─ PASS → 완료
└─ FAIL → 피드백 반환
           │
           ▼
[3차 시도]
전문가가 피드백 기반으로 수정 → code-reviewer가 재검증
│
├─ PASS → 완료
└─ FAIL → 사용자에게 에스컬레이션
```

#### 피드백 전달 메커니즘

리뷰어가 FAIL을 판정하면, `review.md`에 구체적인 피드백이 담긴다. 이 파일이 전문가에게 전달되는 "수정 요청서"다.

Phase 2에서 보여준 코드는 리뷰와 수정이 완료된 최종본이다. 여기서는 리뷰 과정을 재현하기 위해, 초안 시점의 코드를 기준으로 살펴보자. 예를 들어, Python 코드에 대해 리뷰어가 다음과 같은 피드백을 남겼다고 하자.

**`codegen-output/python/review.md` (1차 리뷰 — FAIL):**

```markdown
# 코드 리뷰 결과

## 판정: FAIL
## 리뷰 대상: token_validator.py, test_token_validator.py
## 시도: 1차 / 최대 3차

### CRITICAL 이슈
- [ ] token_validator.py:42 — algorithms 파라미터의 기본값이
  mutable list이다. 함수 호출 간에 공유되어 예기치 않은
  동작이 발생할 수 있다.

### MAJOR 이슈
- [ ] test_token_validator.py — 커스텀 알고리즘(RS256 등)을
  지정한 경우의 테스트가 누락되었다.

### MINOR 이슈 (참고)
- token_validator.py:15 — ValidationStatus의 값을
  대문자로 통일하면 가독성이 향상될 수 있다.

### 피드백 요약
1. validate_token의 algorithms 기본값을 None으로 변경하고,
   함수 내부에서 ["HS256"]을 할당하세요.
2. 커스텀 알고리즘 테스트를 추가하세요.
```

이 피드백을 받은 `python-expert`는 지적된 부분만 수정한다.

```python
# 수정 전
def validate_token(token: str, secret_key: str, algorithms: list[str] = ["HS256"]) -> ValidationResult:

# 수정 후
def validate_token(token: str, secret_key: str, algorithms: list[str] | None = None) -> ValidationResult:
    if algorithms is None:
        algorithms = ["HS256"]
```

그리고 누락된 테스트를 추가한다.

```python
def test_custom_algorithm_is_accepted(self) -> None:
    """커스텀 알고리즘을 지정하면 해당 알고리즘으로 검증한다."""
    payload = {
        "sub": "user-123",
        "exp": datetime.now(tz=timezone.utc) + timedelta(hours=1),
    }
    token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")

    result = validate_token(token, SECRET_KEY, algorithms=["HS256"])

    assert result.is_valid
```

수정 후 `code-reviewer`가 다시 검증한다.

**`codegen-output/python/review.md` (2차 리뷰 — PASS):**

```markdown
# 코드 리뷰 결과

## 판정: PASS
## 리뷰 대상: token_validator.py, test_token_validator.py
## 시도: 2차 / 최대 3차

### CRITICAL 이슈
(없음)

### MAJOR 이슈
(없음)

### MINOR 이슈 (참고)
- token_validator.py:15 — ValidationStatus의 값을
  대문자로 통일하면 가독성이 향상될 수 있다.

### 피드백 요약
모든 CRITICAL, MAJOR 이슈가 해결되었습니다. PASS.
```

MINOR 이슈는 남아 있지만 PASS다. MINOR는 루프 종료 조건에 포함되지 않기 때문이다.

#### 재생성 시 핵심 규칙

검증 루프가 수렴하려면 재생성 규칙이 명확해야 한다.

**규칙 1: 지적된 부분만 수정한다.**
피드백에 없는 부분을 "개선"하려고 건드리면, 새로운 문제가 발생할 수 있다. 외과 수술처럼, 문제가 있는 곳만 정확히 절개하고 봉합해야 루프가 수렴한다.

**규칙 2: 수정 사항을 명시적으로 기록한다.**
전문가가 무엇을 어떻게 수정했는지를 기록하면, 리뷰어가 해당 부분만 집중적으로 재검증할 수 있다. 전체 코드를 처음부터 다시 리뷰하는 것보다 효율적이다.

**규칙 3: 3회 시도 후에도 FAIL이면 에스컬레이션한다.**
무한 루프를 방지하는 안전장치다. 3회 시도로 해결되지 않는 문제는, AI 에이전트의 능력 밖일 가능성이 높다. 사람의 판단이 필요한 시점이다.

---

### 자동 검증 강화: 테스트 실행과 린트

지금까지의 리뷰어는 **코드를 읽고 판단**하는 방식이다. 여기에 **자동 검증 도구**를 결합하면 검증의 정확도가 올라간다.

#### 검증 도구 통합

오케스트레이터 스킬에 자동 검증 단계를 추가한다.

```markdown
### Phase 3 상세: 검증 루프

#### Step 1: 자동 검증 (도구 기반)
각 언어별로 다음 검증을 자동 실행한다.

**Python:**
- 구문 검증: `python -m py_compile {파일}`
- 린트: `ruff check {파일}`
- 포매팅: `black --check {파일}`
- 테스트: `pytest {테스트파일} -v`

**Java:**
- 구문 검증: `javac {파일}`
- 린트: (컴파일 경고 확인)
- 테스트: `java -jar junit-platform-console-standalone.jar --select-class {클래스}`

**Kotlin:**
- 구문 검증: `kotlinc {파일} -include-runtime`
- 린트: `ktlint {파일}`
- 테스트: JUnit 5로 실행

#### Step 2: AI 리뷰 (code-reviewer)
자동 검증을 통과한 코드에 대해 code-reviewer가 리뷰한다.
- 자동 검증 결과를 리뷰의 입력으로 포함한다
- 자동 검증에서 발견하지 못하는 설계적/논리적 문제를 집중 검토한다

#### 검증 순서의 이유
자동 검증을 먼저 하는 이유: 구문 오류나 포매팅 문제를 AI가 판단하게 하는 것은
낭비다. 도구가 정확하게 잡을 수 있는 문제는 도구에 맡기고,
AI 리뷰어는 도구가 잡지 못하는 고수준 문제에 집중하게 한다.
```

왜 자동 검증과 AI 리뷰를 분리하는가?

**역할 분리의 원칙**이다. 구문 오류, 포매팅 위반, 테스트 실패는 도구가 100% 정확하게 잡는다. 이런 문제를 AI에게 맡기면 가끔 놓칠 수 있고, 컨텍스트 윈도우도 낭비된다. 반대로 "이 설계가 확장 가능한가", "이 에러 처리가 충분한가" 같은 판단은 도구로 자동화할 수 없다. 각각이 잘하는 것에 집중하게 하는 것이 효율적이다.

#### 자동 검증 실패 시 처리

자동 검증에서 실패하면 AI 리뷰 없이 바로 전문가에게 돌려보낸다.

```
자동 검증 실패 (예: 테스트 FAIL)
→ 실패 로그를 전문가에게 전달
→ 전문가가 수정
→ 다시 자동 검증
→ 자동 검증 PASS
→ AI 리뷰 (code-reviewer)
```

자동 검증이 통과하지 못한 코드를 AI 리뷰어에게 보내는 것은 비효율적이다. 컴파일이 안 되는 코드를 설계 관점에서 리뷰하는 것은 의미가 없다.

---

### 전문가풀 확장: Go 전문가 추가하기

전문가풀 패턴의 진가는 **확장할 때** 드러난다. 팀에서 Go 서비스를 새로 시작하여, Go 전문가를 추가해야 한다고 하자.

#### 변경이 필요한 파일

| 작업 | 파일 | 변경 내용 |
|------|------|----------|
| 에이전트 추가 | `.claude/agents/go-expert.md` | 새 파일 생성 |
| 라우터 테이블 추가 | `.claude/agents/code-router.md` | 라우팅 규칙에 1줄 추가 |
| 컨벤션 참고자료 추가 | `.claude/skills/code-generation/references/go-conventions.md` | 새 파일 생성 |
| 자동 검증 추가 | `code-generation/SKILL.md` | Go 검증 도구 추가 |

#### 변경이 필요하지 않은 파일

| 파일 | 이유 |
|------|------|
| `python-expert.md` | Go와 무관 |
| `java-expert.md` | Go와 무관 |
| `kotlin-expert.md` | Go와 무관 |
| `code-reviewer.md` | 언어 무관한 범용 기준으로 리뷰 |
| 기존 references/ | 기존 언어 컨벤션은 변경 불필요 |

기존 전문가 에이전트를 전혀 수정하지 않는다. 리뷰어도 수정하지 않는다. **열려 있는 확장, 닫혀 있는 수정**이다.

#### Go 전문가 에이전트

```markdown
---
name: go-expert
description: "Go 코드 생성 전문가. Go, Gin, Echo, goroutine
              관련 코드 생성 요청을 처리합니다."
model: opus
---

# Go 전문가 — 관용적 Go 코드를 생성합니다

당신은 Go 코드 생성을 담당하는 전문가입니다.

## 핵심 역할
1. 라우터의 작업 지시를 기반으로 Go 코드를 생성한다
2. 해당 코드의 testing 패키지 기반 테스트를 함께 생성한다
3. Go의 관용적 스타일(idiomatic Go)을 따른다

## 코딩 원칙
- **에러는 값이다**: panic 대신 error 반환을 사용한다
- **인터페이스 활용**: 작은 인터페이스를 정의하고 구현한다
- **goroutine 안전**: 공유 상태에 mutex 또는 채널을 사용한다
- **명시적 에러 처리**: `if err != nil` 패턴을 따른다
- **간결한 네이밍**: Go 컨벤션에 따라 짧은 이름을 사용한다

## 테스트 원칙
- 표준 testing 패키지를 사용한다
- 테이블 주도 테스트(table-driven tests)를 작성한다
- testify를 보조적으로 사용할 수 있다

## 출력 규칙
- 코드 파일: `codegen-output/go/{package_name}.go`
- 테스트 파일: `codegen-output/go/{package_name}_test.go`
- snake_case 파일명, CamelCase 내보내기 함수명

## 협업
- **입력**: `codegen-output/routing-result.md`의 go-expert 작업 지시
- **출력**: `codegen-output/go/` 디렉토리에 코드와 테스트 저장
```

#### 라우터 테이블에 1줄 추가

```markdown
| 키워드/패턴 | 전문가 | 비고 |
|------------|--------|------|
| python, django, flask, fastapi, pytest | `python-expert` | Python 생태계 |
| java, spring, maven, gradle, junit | `java-expert` | Java 생태계 |
| kotlin, ktor, kotest, coroutine | `kotlin-expert` | Kotlin 생태계 |
| go, golang, gin, echo, goroutine | `go-expert` | Go 생태계 |  ← 추가
```

이것이 전부다. 기존의 Python, Java, Kotlin 전문가는 한 글자도 수정하지 않았다. 리뷰어도 수정하지 않았다. 오케스트레이터의 워크플로우 로직도 변하지 않는다. "선택된 전문가를 호출한다"는 규칙이 이미 전문가 수와 무관하게 동작하기 때문이다.

---

### 설계 판단의 근거 정리

이 챕터에서 내린 설계 판단들을 "왜?"와 함께 정리한다.

#### 왜 라우터와 전문가를 분리하는가?

하나의 에이전트가 라우팅과 코드 생성을 모두 하면, 새 언어를 추가할 때 그 에이전트의 프롬프트가 점점 비대해진다. 라우터는 "판단만" 하고, 전문가는 "생성만" 한다. 역할을 분리하면 각 에이전트가 단순해지고, 개별적으로 개선할 수 있다.

#### 왜 리뷰어를 sonnet으로 하는가?

리뷰어는 "기준에 맞는가"를 판정하는 역할이다. 새로운 코드를 창의적으로 생성하는 것이 아니라, 기존 기준과 대조하는 작업이다. 이런 대조 작업은 sonnet으로 충분하다. opus를 사용하면 비용만 증가하고 품질 차이는 미미하다.

#### 왜 검증 루프를 언어별로 독립 운영하는가?

Python이 2차에서 PASS되고 Java가 3차에서 PASS되면, 전체 루프가 종료된다. 만약 루프를 통합 운영하면, Python이 이미 PASS인데 Java의 재검증 때문에 Python도 다시 검증받게 되어 비효율적이다.

#### 왜 자동 검증을 AI 리뷰 전에 하는가?

컴파일 오류나 테스트 실패는 도구가 즉시 잡는다. AI에게 이런 명확한 문제를 판단하게 하면 컨텍스트 윈도우를 낭비하고, 가끔 놓칠 수도 있다. 자동 검증을 통과한 코드만 AI에게 보내면, AI는 도구가 잡지 못하는 설계와 논리 문제에 집중할 수 있다.

#### 왜 MINOR 이슈로 FAIL하지 않는가?

MINOR 이슈(네이밍, 주석)까지 FAIL 조건에 넣으면, 수정할 때 새로운 MINOR가 생기고, 다시 FAIL하고, 또 수정하면 또 새로운 MINOR가 발생하는 무한 루프에 빠진다. MINOR는 "기록은 하되 통과시키는" 전략이 루프 수렴에 필수적이다.

---

### 전체 흐름 요약

이 챕터에서 구축한 코드 생성 하네스의 전체 데이터 흐름을 정리한다.

```
사용자 요청
    │
    ▼
[Phase 1: code-router]
    │  요청 분석 → 언어 식별 → 전문가 선택
    │  산출물: routing-result.md
    ▼
사용자 확인
    │
    ▼
[Phase 2: 전문가풀]
    ├→ [python-expert] → python/*.py
    ├→ [java-expert]   → java/*.java
    └→ [kotlin-expert] → kotlin/*.kt
    │
    ▼
[Phase 3: 검증 루프] (언어별 독립, 최대 3회)
    │
    ├─ Step 1: 자동 검증 (컴파일, 린트, 테스트)
    │   ├─ FAIL → 전문가에게 피드백 → 수정 → 다시 Step 1
    │   └─ PASS → Step 2로
    │
    └─ Step 2: AI 리뷰 (code-reviewer)
        ├─ FAIL → 전문가에게 피드백 → 수정 → 다시 Step 1
        └─ PASS → 완료
    │
    ▼
[Phase 4: 최종 정리]
    산출물 목록, 리뷰 결과, 반복 횟수 보고
    │
    ▼
사용자에게 최종 보고
```

이 흐름에는 두 가지 패턴이 결합되어 있다.

- **전문가풀**: Phase 1에서 라우터가 전문가를 선택하고, Phase 2에서 선택된 전문가만 실행한다.
- **생성-검증**: Phase 2~3에서 전문가가 생성하고, 리뷰어가 검증하며, 실패 시 전문가가 재생성한다.

두 패턴은 자연스럽게 연결된다. 전문가풀이 "누가 생성하는가"를 결정하고, 생성-검증이 "생성된 결과가 충분한가"를 보장한다.

---

## 정리

- **전문가풀 + 생성-검증 조합**은 다중 도메인 코드 생성에 자연스럽게 적합하다. 전문가풀이 입력별 전문가를 선택하고, 생성-검증이 품질을 보장한다.
- **라우터는 판단만 한다.** 라우팅 규칙을 키워드-전문가 테이블로 명시하면 예측 가능한 라우팅이 된다. 모호한 요청은 추측하지 말고 사용자에게 질문한다.
- **전문가는 자기 언어에 집중한다.** 에이전트 간 컨텍스트 오염을 방지하고, 각 언어의 관용적 스타일을 깊이 있게 반영할 수 있다.
- **리뷰어는 생성하지 않고 검증만 한다.** 역할 분리가 생성-검증 루프의 핵심이다. 이슈 심각도를 분류하고, CRITICAL/MAJOR만 FAIL 조건에 포함하여 루프 수렴을 보장한다.
- **자동 검증과 AI 리뷰를 분리한다.** 도구가 잡을 수 있는 문제는 도구에 맡기고, AI는 도구가 잡지 못하는 고수준 문제에 집중한다.
- **전문가풀은 확장에 열려 있다.** 새 전문가를 추가할 때 에이전트 파일 1개 생성 + 라우터 테이블 1줄 추가로 끝난다. 기존 전문가, 리뷰어, 오케스트레이터를 수정하지 않는다.
- **검증 루프의 최대 반복 횟수는 필수다.** 없으면 무한 루프 위험. 3회가 지나도 해결되지 않으면 사람에게 에스컬레이션한다.

---

## 다음 챕터 미리보기

- Chapter 10에서는 이 책에서 설계한 하네스들을 직접 만들지 않고도, **harness 메타 스킬**을 사용하여 자동 생성하는 방법을 다룬다. "코드 생성 하네스를 구성해줘"라고 말하면 에이전트 파일, 스킬 파일, 디렉토리 구조가 자동으로 생성되는 프로세스를 배운다.
