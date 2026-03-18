# 하네스 엔지니어링 실전 가이드

> Claude Code의 Agent + Skill 구조를 활용한 자동화 체계 구축법

## 온라인 뷰어

https://moongtook.github.io/harness-book/

## 이 책은

Claude Code에서 **에이전트 팀 + 오케스트레이터 스킬**로 구성된 하네스(harness)를 설계하고 구축하는 방법을 다룹니다.

- 13개 챕터 (Ch00~Ch12), 4개 파트
- 7개 SVG 인포그래픽
- 이 책 자체가 하네스(book-writer)로 집필되었습니다

## 목차

| Part | 챕터 | 내용 |
|------|------|------|
| - | Ch00 | 하네스 한눈에 보기 |
| Part 1: 기초 | Ch01~03 | Agent/Skill 개념, 하네스 구성 요소, 4가지 아키텍처 패턴 |
| Part 2: 실전 | Ch04~06 | 에이전트/스킬/오케스트레이터 작성법 |
| Part 3: 심화 | Ch07~09 | 책 집필, 리서치 팀, 코드 생성 하네스 사례 |
| Part 4: 운영 | Ch10~12 | 메타 스킬, 디버깅, 나만의 하네스 설계 |

## 프로젝트 구조

```
.claude/
├── agents/          # 에이전트 정의 (5명)
└── skills/          # 오케스트레이터 스킬
book/
├── chapters/        # 마크다운 챕터 (ch00~ch12)
├── assets/          # SVG 인포그래픽
├── output/web/      # 웹 뷰어 (GitHub Pages)
└── build.py         # 빌드 스크립트
```

## 빌드

```bash
# 웹 뷰어 + PDF 빌드 (pandoc, weasyprint 필요)
python3 book/build.py
```
