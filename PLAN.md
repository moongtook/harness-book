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
- [ ] review-part1-2.md의 WARNING 6건 수정
  - Ch04/Ch05 에이전트 예시 불일치
  - Ch06 최대 반복 횟수 누락
  - 용어 불일치 (SUGGESTION vs INFO, "스스로")
- [ ] review-part3-4.md의 WARNING 7건 수정
  - Ch09 Java JWT API 불일치
  - Ch08 "SubAgent" 용어 통일

### Phase 2: GitHub Pages 배포
- [ ] GitHub 저장소 생성
- [ ] book/output/web/index.html을 GitHub Pages로 배포
- [ ] README.md 작성 (프로젝트 소개 + 온라인 뷰어 링크)

### Phase 3: PDF 재빌드
- [ ] WARNING 수정 반영 후 `python3 book/build.py`로 재빌드
- [ ] PDF + 웹 뷰어 최종 검증
