# Wiki Operation Log

> append-only. 기존 항목 수정·삭제 금지 (INV-4).

## [2026-04-25] ingest | 위키 초기화
- 신규: 폴더 구조 생성 (raw/, wiki/ 하위 9개 카테고리)
- 신규: INDEX.md 초기화
- 신규: log.md 초기화
- INDEX 갱신: ✓

## [2026-04-25] ingest | 2026-04-25T183802+0900 llm-wiki.md
- 출처: Andrej Karpathy GitHub Gist (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f#llm-wiki)
- 신규 (sources/): [[karpathy_llm_wiki_gist]] (캡처 부분적 → needs-review)
- 신규 (concepts/): [[LLM_Wiki_패턴]]
- 모순: 없음
- 주의: 웹 클리퍼가 원문 일부만 캡처. 원본 재수집 시 보완 필요.
- INDEX 갱신: ✓ (페이지 수 5 → 7)

## [2026-04-25] ingest | 2026-04-25T192525+0900 상호운용성 모델.md
- 출처: KSGA 스마트그리드 표준화 프레임워크 3.0 (https://www.ksga.org/sgstandard/framework/01/01/01.do)
- 신규 (sources/): [[KSGA_스마트그리드표준화프레임워크_3.0]]
- 신규 (concepts/): [[스마트그리드]], [[상호운용성]]
- 신규 (standards/): [[SGAM]], [[GWAC_상호운용성모델]]
- 모순: 없음
- INDEX 갱신: ✓ (페이지 수 0 → 5)

## [2026-04-25] project | paper-collector → LLM Wiki raw/papers/ 경로 연동 (Option A)
- 변경: papercollector-worker-agent.md B-1-0 — PDF 다운로드 경로를 `90_LLMWiki/raw/papers/{arxiv_id}.pdf`로 변경
- 변경: papercollector-worker-agent.md B-2 — 노트 프론트매터 `pdf_path` 필드 업데이트
- 변경: paper-collector/CLAUDE.md — Obsidian 폴더 구조 다이어그램 및 B-1-0 경로 문서 갱신
- 설계: PDF 원본 → `raw/papers/` (wiki용), `.md` 노트 → `03_Resources/01_Papers/Topics/` (유지)
- 데이터 공백: 없음
- INDEX 갱신: 해당 없음 (wiki 페이지 생성 없음)

## [2026-04-25] ingest | 2026-04-25T194029+0900 llm-wiki.md (2차 캡처 + 개념 페이지 보강)
- 출처: Andrej Karpathy GitHub Gist (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f#llm-wiki) — 2차 캡처, 동일하게 도입부만 수집
- 업데이트 (sources/): [[karpathy_llm_wiki_gist]] — 2차 캡처 파일 추가, 캡처 실패 2회 명시
- 업데이트 (concepts/): [[LLM_Wiki_패턴]] — 대폭 보강: INV 6대 규칙, Ingest 9단계 워크플로, Query 검색 우선순위, Lint 9항목, 인용 체인, 모순 처리 패턴, RAG 비교 확장 (CLAUDE.md 파생 출처 활용)
- 신규 (people/): [[Andrej_Karpathy]]
- 주의: 2차 캡처도 원문 부분적 수집. 세부 패턴은 CLAUDE.md (Karpathy 패턴 기반 명시)를 파생 출처로 활용하여 보완
- 모순: 없음
- INDEX 갱신: ✓ (페이지 수 7 → 8)
