# 하네스 엔지니어링 실전 가이드 — 책 집필 프로젝트

## 배경

ai-toolkit(구 prompts) 프로젝트에서 하네스 메타 스킬(`/harness`)을 테스트하기 위해 시작된 책 집필 프로젝트.
`/harness`로 Agent 5명 + 오케스트레이터 Skill을 자동 생성하고, 12챕터 집필 + 리뷰 + 인포그래픽 + PDF + 웹뷰어까지 완성.
이후 별도 프로젝트로 분리됨.

### 현재 산출물 상태
- 12개 챕터 집필 완료 (book/chapters/)
- 리뷰 완료 (CRITICAL 1건 수정, WARNING 13건)
- SVG 인포그래픽 7개 (book/assets/)
- PDF 259페이지 (book/output/pdf/)
- 웹 뷰어 (book/output/web/index.html)
- 빌드 스크립트 (book/build.py)

### 에이전트/스킬 파일 위치
현재 루트에 에이전트 파일이 흩어져 있음. 프로젝트 초기화 시 `.claude/` 구조로 배치 필요.

---

## TODO

### Phase 0: 프로젝트 초기화
- [x] git init
- [x] `.claude/agents/` 디렉토리 생성 후 에이전트 파일 이동
  - book-planner.md, chapter-writer.md, book-editor.md, infographic-creator.md, book-publisher.md
- [x] `.claude/skills/book-writer/` 디렉토리 생성 후 오케스트레이터 이동
  - book-writer/SKILL.md → .claude/skills/book-writer/SKILL.md
- [x] `/add-permission` 실행 (Bash 허용 규칙 설정)
- [x] .gitignore 작성

### Phase 0.5: Chapter 0 추가
- [x] ch00-harness-at-a-glance.md 작성 (핵심 개념 서두 배치)
- [x] outline.md에 Ch00 추가

### Phase 1: 리뷰 WARNING 수정
- [x] review-part1-2.md의 WARNING 6건 수정
  - W4-1: Ch04 중첩 코드 블록 zero-width space 트릭 설명 추가
  - W5-1: Ch05 code-reviewer 예시를 Ch04 완성본과 일치
  - W6-1: Ch06 오케스트레이터에 최대 반복 횟수(2회) 추가
  - W-Cross-1: Ch04 "네 가지" → "다섯 가지" 수정
  - W-Cross-2: Ch05 INFO → SUGGESTION 용어 통일
  - W2-1: "스스로"는 올바른 맞춤법이므로 수정 불필요
- [x] review-part3-4.md의 WARNING 7건 수정
  - Ch07-W1: 추측형 "나왔을 것이다" → "나올 수 있다" 수정
  - Ch08-W1: "SubAgent" → "서브 에이전트" 전체 통일
  - Ch09-W1: Phase 2/3 코드 타임라인 설명 추가
  - Ch09-W2: Java/Kotlin `claim.as()` → `claim.asString()` 수정
  - Ch10-W1: Phase 4 소제목에 "(이 예제에서는 별도 스킬 미생성)" 명시
  - Ch11-W1: 디버깅 로그 날짜 하드코딩 → 플레이스홀더 변경
  - Ch12-W1: 마지막 챕터에 "부록 안내" 섹션 추가

### Phase 2: GitHub Pages 배포
- [ ] GitHub 저장소 생성
- [ ] book/output/web/index.html을 GitHub Pages로 배포
- [ ] README.md 작성 (프로젝트 소개 + 온라인 뷰어 링크)

### Phase 3: PDF 재빌드
- [ ] WARNING 수정 반영 후 `python3 book/build.py`로 재빌드
- [ ] PDF + 웹 뷰어 최종 검증
