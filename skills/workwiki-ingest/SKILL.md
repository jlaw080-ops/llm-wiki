---
name: workwiki-ingest
description: Obsidian PARA 업무 폴더(01_Projects, 02_Areas, 05_Daily, 06_To Do)를 스캔하여 work-wiki 페이지를 생성·갱신하는 인제스트 스킬. 서브에이전트 병렬 디스패치로 컨텍스트 창 소진을 방지.
trigger: /work-ingest
---

# Work Wiki — Ingest 스킬

## 핵심 철학

업무 폴더의 살아있는 노트를 구조화된 wiki 페이지로 증류한다.

- **소스 폴더는 불변이 아니다**: 업무 노트는 계속 수정된다. Ingest는 읽기 전용으로만 접근한다.
- **요약이 아닌 증류**: 날짜별 일지가 아니라 의사결정·프로젝트 맥락·액션 아이템을 wiki 페이지로 추출한다.
- **재인제스트 지원**: 노트 수정 시 재스캔 가능. log.md가 마지막 인제스트 날짜를 추적한다.
- **상태 필터링**: `상태: 완료` 노트는 스킵. 진행 중·예정·백로그만 대상.

---

## 핵심 불변 규칙

| 규칙 | 내용 |
|------|------|
| INV-1 | 읽기/쓰기 모두 `work-wiki/` 내부에서만 (소스 폴더는 읽기 전용) |
| INV-2 | 소스 폴더 파일 수정 절대 금지 |
| INV-3 | wiki 페이지 생성·수정 후 `work-wiki/INDEX.md` 갱신 (Orchestrator가 배치로 수행) |
| INV-4 | 인제스트 완료 후 `work-wiki/log.md`에 항목 추가 (Orchestrator가 배치로 수행) |
| INV-5 | wiki 페이지 내 모든 사실 진술에 소스 노트 위키링크 표기 |
| INV-6 | `상태: 완료` 노트 처리 금지. 템플릿 파일 처리 금지 |

---

## 파이프라인 아키텍처

### 문제: 단일 컨텍스트 순차 처리

기존 방식은 모든 소스 파일을 한 컨텍스트에서 순차 읽기 → 200+ 파일 × 200–350줄 = 컨텍스트 창 급속 소진 → 반복 compaction.

### 해결: Orchestrator-Worker 분리

```
Orchestrator (메인 대화 컨텍스트)
  │
  ├── STEP 0: log.md 읽기 + Glob → 처리 대상 파일 목록 (경로만)
  │
  ├── STEP 1: Worker 서브에이전트 배치 디스패치 (5개씩 병렬)
  │     ├── Worker A ──┐
  │     ├── Worker B  │ 각자 독립 컨텍스트에서:
  │     ├── Worker C  │  - 소스 파일 읽기
  │     ├── Worker D  │  - Q&A 분석
  │     └── Worker E ─┘  - wiki 페이지 생성
  │           │
  │           └── 각 Worker → 구조화 요약만 반환 (~100 토큰)
  │
  ├── STEP 2: 모든 배치 완료 후 요약 수집
  │
  ├── STEP 3: Orchestrator가 INDEX.md 일괄 갱신 (INV-3)
  │
  ├── STEP 4: Orchestrator가 log.md 일괄 기록 (INV-4)
  │
  └── STEP 5: 최종 보고서
```

**Orchestrator 컨텍스트**: 파일 경로 목록 + Worker 반환 요약만 누적. 소스 파일 전체 내용은 절대 읽지 않음.

---

## 소스 폴더 및 카테고리 매핑

### 스캔 대상 폴더

```
Vault_jlaw80/01_Projects/**/*.md   → 프로젝트 노트
Vault_jlaw80/02_Areas/**/*.md      → 영역(책임 범위) 노트
Vault_jlaw80/05_Daily/**/*.md      → 일일 노트 (미팅, 메모 포함)
Vault_jlaw80/06_To Do/**/*.md      → 할일·스케줄 노트
```

### wiki 카테고리 매핑

| 소스 노트 성격 | wiki/ 폴더 | 예시 |
|--------------|-----------|------|
| 미팅·회의 기록 | `wiki/meetings/` | 범한퓨얼셀 미팅, SW개발회의 |
| 의사결정·방향 결정 | `wiki/decisions/` | 특허 청구범위 결정, 통신규격 선택 |
| 프로젝트 컨텍스트·진행상황 | `wiki/projects/` | 연료전지 열최적화 특허, VPP 과제 |
| 액션 아이템·할일 | `wiki/actions/` | 마감이 있는 업무, 후속 조치 |
| 여러 카테고리 종합 | `wiki/synthesis/` | 주간 업무 흐름, 프로젝트 간 관계 |

---

## 실행 단계

### STEP 0 — Orchestrator: 사전 준비 및 파일 목록 확정

Orchestrator(메인 컨텍스트)만 수행. 소스 파일 내용은 **절대 읽지 않는다**.

0. `work-wiki/.ingest-temp.json` 존재 여부 확인
   - 존재하면 이전 실행이 비정상 종료된 것 → 사용자에게 알리고 삭제 후 계속
1. `work-wiki/CLAUDE.md` 읽기 (도메인 파악)
2. `work-wiki/log.md` 읽기 → **마지막 100줄만** 읽는다 (전체 읽기 금지)
   - log.md가 100줄 미만이면 전체 읽기 허용
   - 오래된 기록은 스킵 판단에 불필요하므로 무시
3. 4개 소스 폴더를 **Glob만** 실행 (파일 경로 수집, 내용 읽기 금지)
4. 각 경로에 대해 **파일명·수정일만으로** 스킵 조건 판단:
   - 파일명에 `template`, `Template`, `_template` 포함 → 스킵
   - `log.md`에 동일 경로 기록이 있고 파일 수정일이 인제스트일 이전 → 스킵
   - (frontmatter `상태` 확인은 Worker가 담당 — 여기서는 파일명 기준만)
5. 처리 대상 파일 목록 확정 → 사용자에게 보고

```
처리 대상: N개 파일 (스킵 예상: M개 — Worker에서 최종 확인)
계속할까요?
```

---

### STEP 1 — Orchestrator: Worker 서브에이전트 배치 디스패치

사용자 확인 후, **5개씩 병렬 배치**로 Agent 서브에이전트를 호출한다.

#### 배치 실행 방법

```python
# 의사코드: 5개씩 병렬 배치 + temp 파일 flush
batches = chunk(process_list, size=5)
batch_count = 0
created = 0; skipped = 0

for batch in batches:
    batch_results = parallel_agent_calls([
        make_worker_prompt(file_path) for file_path in batch
    ])
    batch_count += 1

    # 배치 결과를 즉시 temp 파일에 flush (컨텍스트 누적 방지)
    append_json_array("work-wiki/.ingest-temp.json", batch_results)

    # Orchestrator 컨텍스트에는 카운트 요약만 유지
    created += sum(r.result == "created" for r in batch_results)
    skipped += sum(r.result == "skipped" for r in batch_results)
    print(f"배치 {batch_count}/{len(batches)} 완료: 생성 {created}개, 스킵 {skipped}개")
```

Agent 호출 시 `subagent_type`은 지정하지 않음 (general-purpose).

> **컨텍스트 관리 원칙**: 배치 결과를 Orchestrator 대화 컨텍스트에 누적하지 않는다.
> 결과 JSON은 `work-wiki/.ingest-temp.json`에만 기록하고, 컨텍스트에는 숫자 요약만 남긴다.

#### Worker 서브에이전트 프롬프트 템플릿

각 Worker에게 전달하는 프롬프트:

```
당신은 Work Wiki Ingest Worker입니다.
주어진 소스 파일 하나를 처리하고, 구조화된 요약을 JSON으로 반환하세요.

## 작업 컨텍스트

- Vault 루트: C:\Users\jlaw8\Vault_jlaw80
- Work-wiki 루트: C:\Users\jlaw8\Vault_jlaw80\90_LLMWiki\work-wiki
- 소스 파일: {SOURCE_FILE_PATH}

## 스킵 조건 (해당하면 즉시 skip 결과 반환)

1. frontmatter 없음 (클리핑, 참고자료 등)
2. frontmatter에 `상태: 완료` 또는 `status: done`
3. 본문이 URL 또는 PDF 첨부만 존재 (내용 없음)
4. tags에 `clippings` 포함

## 처리 절차 (스킵 아닌 경우)

1. 소스 파일 읽기
2. Q&A 분석:
   - 핵심 주제는?
   - 어떤 의사결정이 있었나?
   - 어떤 액션 아이템이 도출됐나? (마감일 포함)
   - 어떤 프로젝트/영역과 연결되나?
   - 참여자는 누구인가?
3. 카테고리 결정 (meetings/decisions/projects/actions/synthesis)
   하나의 노트가 여러 카테고리에 해당하면 각각 별도 wiki 페이지 생성
4. wiki 페이지 생성 (아래 템플릿 사용)
   경로: work-wiki/wiki/{category}/{YYYYMMDD}_{핵심키워드}.md
5. 결과를 JSON으로 반환 (아래 형식)

## wiki 페이지 프론트매터 형식

```yaml
---
title: {페이지 제목}
category: meetings | decisions | projects | actions
tags: [{관련 태그}]
source_notes:
  - "[[{소스 노트 파일명}]]"
created: {YYYY-MM-DD}
last_updated: {YYYY-MM-DD}
status: active | closed | superseded
---
```

## 카테고리별 본문 구조

**meetings/**:
- 참석자
- 논의 사항
- 결정 사항 → [[decisions/...]] 링크
- 후속 액션 → [[actions/...]] 링크

**decisions/**:
- 결정 내용
- 배경 및 근거
- 대안
- 영향 범위

**projects/**:
- 현재 상태 (진행중/예정/백로그)
- 핵심 목표
- 주요 마일스톤 표
- 관련 의사결정 링크

**actions/**:
- 액션 항목 (마감일 포함)
- 출처 맥락
- 완료 조건

## 금지 사항

- ❌ 소스 파일 수정 (읽기 전용)
- ❌ INDEX.md 또는 log.md 수정 (Orchestrator 담당)
- ❌ work-wiki/ 외부 파일 수정

## 반환 형식 (반드시 이 JSON만 반환)

```json
{
  "source_file": "{SOURCE_FILE_PATH}",
  "result": "created" | "updated" | "skipped",
  "skip_reason": null | "상태: 완료" | "no frontmatter" | "url-only" | "clippings",
  "pages": [
    {
      "path": "wiki/decisions/20260416_foo.md",
      "category": "decisions",
      "title": "의사결정 제목",
      "index_row": "| 2026-04-16 | [[decisions/20260416_foo]] | 결정 내용 요약 |",
      "log_entry": "  - [[wiki/decisions/20260416_foo]] (decisions)"
    }
  ]
}
```
```

---

### STEP 2 — Orchestrator: 결과 복원

모든 배치 완료 후, `work-wiki/.ingest-temp.json`을 읽어 Worker 결과를 복원한다.

```
work-wiki/.ingest-temp.json 읽기
→ 전체 결과 배열 파싱
→ 처리됨 / 스킵됨 목록 재구성
```

이 시점에서 컨텍스트는 다음만 포함:
- STEP 0 Glob 결과 (경로 목록)
- 배치별 카운트 요약 (숫자만)
- `.ingest-temp.json` 내용 (최초 1회 읽기)

소스 파일 전체 내용, Worker 프롬프트 본문, Agent 호출 결과는 컨텍스트에 남기지 않는다.

---

### STEP 3 — Orchestrator: INDEX.md 일괄 갱신 (INV-3)

모든 Worker 완료 후, 수집된 `index_row` 값으로 INDEX.md를 한 번에 갱신한다.

**갱신 규칙**:
- 신규 행은 해당 섹션에 날짜 내림차순 삽입
- 같은 소스 파일로 이미 등록된 행이 있으면 갱신 (중복 추가 금지)
- `마지막 갱신` 날짜 및 `총 페이지 수` 업데이트

INDEX.md는 **이 STEP에서 한 번만 쓴다**. Worker는 INDEX.md를 읽거나 쓰지 않는다.

---

### STEP 4 — Orchestrator: log.md 일괄 기록 (INV-4)

수집된 `log_entry` 값으로 log.md에 배치 인제스트 항목을 추가한다.

```markdown
## [YYYY-MM-DD] ingest | batch — {N}개 파일 처리

### 처리됨
- 소스: {source_file_1}
  - 액션: 신규 생성 {K}개
  {log_entry_lines}
  - INDEX 갱신: ✓ (배치)

- 소스: {source_file_2}
  ...

### 스킵
- {source_file_X}: {skip_reason}
- {source_file_Y}: {skip_reason}
```

log.md는 **이 STEP에서 한 번만 쓴다**. Worker는 log.md를 읽거나 쓰지 않는다.

---

### STEP 4.5 — Orchestrator: temp 파일 정리

STEP 4(log.md 갱신) 완료 직후 실행:

```
work-wiki/.ingest-temp.json 삭제
```

이 파일은 파이프라인 실행 중에만 존재한다. 비정상 종료 시 잔류할 수 있으며, 다음 인제스트 STEP 0에서 잔류 파일이 있으면 삭제 후 재시작한다.

---

### STEP 5 — Orchestrator: 최종 보고서

```
## Work Wiki Ingest 완료 — {YYYY-MM-DD}

### 처리 결과
- 신규 wiki 페이지: {N}개
- 갱신된 wiki 페이지: {N}개
- 스킵: {N}개 (완료 상태: A, frontmatter 없음: B, 미변경: C, 기타: D)

### 신규/갱신 페이지 목록
- [[meetings/{파일명}]] ← {소스 노트명}
- [[decisions/{파일명}]] ← {소스 노트명}
- ...

### 권장 후속 작업
- /lint 실행으로 wiki 건강 상태 점검
- 연결되지 않은 프로젝트 페이지 cross-link 추가 검토
```

---

## 재인제스트 가이드

**재인제스트 판단 기준**:
- log.md의 인제스트 날짜보다 소스 노트의 수정일이 이후인 경우 → 재인제스트 대상
- 소스 노트의 `상태`가 `진행중` → `완료`로 변경된 경우 → 해당 wiki 페이지 `status: closed`로 갱신

**재인제스트 실행**:
`/work-ingest` 재실행 시 STEP 0에서 변경 감지 → 해당 파일만 Worker에 디스패치.

---

## 금지 사항

### Orchestrator 금지
- ❌ 소스 파일 내용 직접 읽기 (Glob 경로 수집만 허용)
- ❌ wiki 페이지 직접 생성 (Worker에 위임)
- ❌ 배치 중간에 INDEX.md 또는 log.md 부분 갱신 (STEP 3-4에서 일괄 처리)

### Worker 금지
- ❌ 소스 폴더 파일 수정 (읽기 전용)
- ❌ INDEX.md 또는 log.md 수정 (Orchestrator 전담)
- ❌ work-wiki/ 외부 파일 생성/수정
- ❌ `상태: 완료` 노트 처리
- ❌ frontmatter 없는 노트 처리
- ❌ 기존 wiki 페이지 내용 통째로 덮어쓰기 (재인제스트 시 추가/수정만)

---

## 스킵 조건 판단 주체

| 조건 | 판단 주체 |
|------|---------|
| 파일명에 `template` 포함 | Orchestrator (STEP 0, 파일명만으로 판단) |
| log.md에 기록 + 수정일 이전 | Orchestrator (STEP 0, 메타데이터만으로 판단) |
| frontmatter 없음 | Worker (파일 읽은 후 판단) |
| `상태: 완료` | Worker (frontmatter 읽은 후 판단) |
| URL/첨부만 있는 내용 없는 노트 | Worker (본문 읽은 후 판단) |
| `tags: clippings` | Worker (frontmatter 읽은 후 판단) |

---

## llmwiki-ingest와의 차이점

| 항목 | llmwiki-ingest | workwiki-ingest |
|------|---------------|-----------------|
| 소스 | `raw/` (불변 드롭존) | 4개 업무 폴더 (살아있는 문서) |
| 처리 방식 | 단일 컨텍스트 순차 | Orchestrator-Worker 분리 |
| 병렬성 | 없음 | 5개씩 병렬 배치 |
| INDEX/log 갱신 시점 | 파일별 즉시 | Orchestrator 배치 (완료 후) |
| 컨텍스트 부하 | 파일 전체 내용 누적 | JSON 요약만 누적 |
| 스킵 기준 | 이미 처리됨(log) | 완료·템플릿·미변경 (2단계 판단) |
| STEP 2 질문 | "왜 캡처했나?" | "어떤 결정·액션이 있었나?" |
| wiki 카테고리 | concepts/standards/equipment/... | meetings/decisions/projects/actions |
