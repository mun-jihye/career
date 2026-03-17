# Agentic RLSC 추출 모듈 개발

> Agentic RLSC 추출 시 **LLM에 피드백을 제공**해 보다 **정확한 RLSC**를 추출하는 모듈 설계·구현 및 **전체 템플릿 대량 추출 파이프라인**까지 담당한 내역.  
> **출처**: 2025 하반기 셀프리뷰, Confluence [Agentic RLSC Structuriser 전체 파이프라인 구현](https://miridih.atlassian.net/wiki/spaces/~712020a23635f8190743df9448a03c47cdb090/pages/1901461616), Jira MOR-595·MOR-622·MOR-577

---

## 1. 목표·배경

- **목표**: Agentic RLSC를 추출할 때 **LLM에게 피드백을 주어** 이전 추론 결과를 반영해 **더 정확한 RLSC**를 추출.
- **결과**: Page 단위 80%, Component 단위 95% 목표를 **모든 테스트 구간에서 초과** 달성 후, 약 **143,790 pages** 전체 추출 진행 결정·실행.

---

## 2. 내가 한 작업 (역할)

- **RLSC Generator 구현**: Agentic RLSC 추출 모듈의 **가장 앞단**인 **RLSC 생성** 로직 구현. LLM 호출·입출력 형식 설계.
- **AI 프롬프트·피드백 설계**: **RLSC 생성용 프롬프트**와 **Evaluator 평가 결과를 반영하는 피드백 프롬프트** 설계·수정. LLM에 피드백을 주고 재추론하는 흐름 반영.
- **Evaluator 연동**: **Evaluator**의 평가 결과를 기반으로 **피드백 프롬프트**를 구성해 LLM에 전달하는 로직 구현.
- **Langfuse 연동**: LLM 호출에 **Langfuse** 로그 기록 및 **비용 반영**.
- **전체 템플릿 RLSC 추출 파이프라인**: 3단계(json→Agentic RLSC 배치)·4단계(Design Object 추출 및 썸네일) 파이프라인 구현·문서화.

---

## 3. 사용한 툴·기술

| 구분 | 툴·기술 |
|------|----------|
| **LLM·에이전트** | Agentic RLSC Structuriser, LLM API 호출, 피드백 루프 |
| **관측·비용** | Langfuse (트레이스·로그·비용) |
| **백엔드·DB** | Node.js/TypeScript, Supabase (PostgreSQL), pg-boss |
| **테이블** | `layouts`, `rlsc_results`, `job_history`, `design_objects` |
| **API·프론트** | `/api/agentic-total-batch-conversion/agentic-rlsc/start`, `AgenticTotalBatchConversionPage`, `AgenticRLDialog` |

---

## 4. 워크플로우 (3단계: json → Agentic RLSC 배치)

### 4.1. 실행 대상·성공/실패 정의

| 구분 | 내용 |
|------|------|
| **실행 대상** | `rlsc_results`에 `type = 'agentic'`인 레코드가 한 건이라도 있는 `layout_id`는 제외. 즉, **아직 Agentic RLSC가 없는** 레이아웃. (실제로는 “JSON은 있는데 Agentic RLSC는 아직 없는” layout id 목록 사용) |
| **성공** | Agentic RLSC 결과가 레이아웃에 저장된 건수. `layouts.structured_content_agentic_jsonb`, `total_status = 'success_agentic_rlsc'`. |
| **실패** | `layouts.total_status = 'failed_design_object'` 또는 `failed_agentic_rlsc`. |

### 4.2. 프론트엔드 → API → 큐

1. **프론트**: “Agentic RLSC 배치 실행” 버튼 → `AgenticRLSCDialog`에서 `modelType`, `iterationCount`, `reasoningEffort`, `jobCount` 등 입력 → `startAgenticRlscBatch()` → `POST /api/agentic-total-batch-conversion/agentic-rlsc/start`.
2. **백엔드 (agentic-rlsc/start/route.ts)**:
   - Supabase `layouts`에서 조건 조회:
     - `json_sheet_data IS NOT NULL` (2단계 JSON 변환 완료)
     - `structured_content_agentic_jsonb IS NULL` (Agentic RLSC 미적용)
     - 정렬: `id` 오름차순, limit(요청 body, 없으면 최대 10,000)씩 반복.
   - `layoutIds`에 대해 `batchId`(UUID) 생성, 각 `layoutId`마다 `jobCountPerLayout`만큼 payload 생성: `{ layoutId, batchId, modelType, iterationCount, reasoningEffort }`.
   - **pg-boss** 큐에 `send()`로 비동기 job 등록.
   - 생성된 `jobId`들을 **job_history**에 `status='pending'`으로 upsert.

### 4.3. 워커에서 실제 RLSC 실행

- **워커**: `src/workers/rlsc-worker.ts`, `pg-boss.work(QUEUE_NAME, ...)`로 큐 구독.
- **job 1건 처리 흐름**:
  1. **layouts 로드**: `getLayoutById(layoutId)` → `json_sheet_data`, `template_idx` 등.
  2. **job_history**: 해당 job `status='processing'`, `started_at` 갱신.
  3. **Agentic RLSC 실행**: `AgenticRLSCStructuriserManager(layout, { maxIterationCount, modelType, reasoningEffort, ... }).process()` → `relativeContent`, `iterations`, `history`, `traceId` 등 생성. (여기서 Generator·Evaluator·피드백 프롬프트가 사용됨.)
  4. **성공 시**:
     - `job_history`: `status='completed'`, `finished_at`, `duration_sec`, `logs=result.rlscHistory` upsert.
     - `layouts`: `structured_content_agentic_jsonb`, `structured_content_agentic_status='ready'`, `total_status='success_agentic_rlsc'` 갱신.
     - `rlsc_results`: `layout_id`, `iterations`, `type='agentic'`, `model_name`, `max_iteration`, `status='pending'`, `trace_id`, `batch_id` insert.
  5. **실패 시**: `job_history` → `status='failed'`, `error`, `finished_at` 등. `layouts.total_status='failed_agentic_rlsc'`. 예외 재throw로 pg-boss 재시도 유도.
  6. **타임아웃**: 별도 워커(reapTimeOutJobs)가 `job_history.status='processing'`이면서 `started_at`이 오래된 job을 `failed`, `error='timeout...'`으로 정리.

### 4.4. 3단계 결과가 적재되는 곳

| 저장소 | 용도 |
|--------|------|
| **job_history** | job_id, queue, batch_id, layout_id, status(pending/processing/completed/failed), started_at, finished_at, duration_sec, error, logs(JSON). |
| **layouts** | structured_content_agentic_jsonb, structured_content_agentic_status, total_status. |
| **rlsc_results** | layout_id, iterations, type='agentic', model_name, max_iteration, status, trace_id, batch_id. |

---

## 5. 4단계: Design Object 추출 및 썸네일

- **실행 대상**: `rlsc_results`에 `type='agentic'`인 레코드가 한 건이라도 있는 `layout_id`.
- **성공/실패**: `layouts.total_status = 'success_design_object'` / `'failed_design_object'`.
- **프론트**: “Design Object 배치 실행” → `startAgenticDesignObjectsBatch({ limit, batchSize })` → `POST /api/agentic-total-batch-conversion/design-objects/start`.
- **백엔드**: `rlsc_results`에서 `type='agentic'`, layout_id별 `created_at` 최신 1건으로 후보 `layout_id` 선정 → `design_objects` 생성 로직에서 `layouts`(json_sheet_data, rlsc_result.iterations 등) 사용해 Design Object 추출·저장.

---

## 6. 정량 결과 (품질 목표 대비)

| 구간 | Page 단위 정답률 | Component 단위 정답률 |
|------|------------------|------------------------|
| 1차 (98개 샘플) | **89.80%** | **97.75%** |
| 2차 (추가 20개) | **90.00%** | **97.89%** |
| 3차 (랜덤 100개) | **90%** | **97%** |
| **목표** | 80% | 95% |

→ 모든 구간에서 목표 초과. 내부 검수 약 1,000건에서도 유사 수준 정합도 확인 후, **약 143,790 pages** 전체 추출 진행 결정.

---

## 7. 참고 링크

- [Agentic RLSC Structuriser 전체 파이프라인 구현](https://miridih.atlassian.net/wiki/spaces/~712020a23635f8190743df9448a03c47cdb090/pages/1901461616)
- [MOR-595](https://miridih.atlassian.net/browse/MOR-595), [MOR-622](https://miridih.atlassian.net/browse/MOR-622), [MOR-577](https://miridih.atlassian.net/browse/MOR-577)

---

*2025 하반기 셀프리뷰·Confluence 개인 스페이스 문서 기준.*
