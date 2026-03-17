# AI 학습 데이터 적재 파이프라인

> **신규 미캔 데이터 전처리 자동화(SQS)** 및 **Airflow 기반 학습 데이터 파이프라인 중앙 집중화**, 그리고 **저퀄리티 분류 모델**용 데이터셋·파이프라인(AS-IS/TO-BE) 정리.  
> **출처**: Jira AI-697·AI-796, Confluence [딱 맞는 디자인 찾기 - 파이프 라인](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/882737394), [저퀄리티 분류 모델 - 파이프라인](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/882509913), [저퀄리티 분류 모델 데이터셋 - 디자인 허브](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/869470273)

---

## 1. 빌드업 요약

| 단계 | 내용 |
|------|------|
| **AS-IS** | 리서처가 MA 스쿼드에 **온콜로 데이터 요청** → MA가 **데이터브릭스**로 데이터 수집 후 링크 전달 → 리서처가 **CSV 추출·GPU 서버 업로드** → **csv 파싱·이미지 적재 스크립트** 수동 실행 (파이프라인마다 프로세스 익혀야 함, 히스토리·스크립트 버전 관리 부재, 데이터 품질 보장 어려움). |
| **중간** | **SQS 활용** 신규 미캔 데이터 전처리 자동화 — **매주 월요일 10시** 실행, 추출된 이미지 **GPU 자동 적재** (AI-697). |
| **TO-BE** | **Airflow**가 “데이터 적재 요청 → 이미지 처리 → GPU 연산”까지 **자동화된 워크플로우**로 관리. **학습 데이터 파이프라인 중앙 집중화** (AI-796). (라벨링 데이터 제외 시, 사람 개입 없이 파이프라인 자동화 목표.) |

---

## 2. 내가 한 작업 (역할)

- **신규 미캔 데이터 전처리 자동화 (SQS)**: 매주 월요일 10시 실행 트리거, 추출 이미지 GPU 자동 적재까지 흐름 설계·구현. [AI-697](https://miridih.atlassian.net/browse/AI-697)
- **Airflow 기반 학습 데이터 파이프라인 중앙 집중화**: 기존 SQS 버전을 유지·보완하지 않고 **Airflow로 학습 데이터 파이프라인을 한곳에서 관리**하도록 전환. [AI-796](https://miridih.atlassian.net/browse/AI-796)
- **저퀄리티 분류 모델 데이터셋·파이프라인**: 디자인허브·검탐 측면 데이터셋 정의(승인/거부, 유저픽·노출 대비 채택률), 데이터브릭스 쿼리·경로 정리. 저퀄리티 파이프라인 AS-IS(리서처→MA→데이터브릭스→리서처 csv·GPU→스크립트) 문서화, TO-BE 방향 고려.

---

## 3. 사용한 툴

| 구분 | 툴 |
|------|-----|
| **스케줄·오케스트레이션** | SQS(트리거·메시지), Apache Airflow (DAG, 중앙 집중화) |
| **데이터·쿼리** | 데이터브릭스(Notebook, SQL), CSV, 이미지 경로 |
| **저장·연산** | GPU 서버, file.miricanvas.com(파일 서버) |
| **협업·문서** | Confluence AIDMLOps, Jira AI-697·AI-796 |

---

## 4. 워크플로우

### 4.1. 신규 미캔 데이터 전처리 (SQS 기반)

- **트리거**: 매주 월요일 10시 (스케줄 또는 SQS 메시지).
- **흐름**: (전처리 조건 만족 시) 데이터 추출 → 이미지 목록/메타 생성 → **GPU 서버 자동 적재** (csv 파싱·이미지 요청·적재 스크립트가 자동 실행되도록 구성).

### 4.2. 저퀄리티 분류 모델 파이프라인

**AS-IS**

1. 리서처가 MA 스쿼드에 **온콜로 데이터 요청**.
2. MA가 **데이터브릭스**로 요청 데이터 수집 후 **데이터브릭스 링크** 전달.
3. 리서처가 **CSV 추출** 후 **GPU 서버에 업로드**.
4. **csv 파싱·이미지 적재 스크립트** 실행:
   - CSV에서 이미지 경로 추출 → **file.miricanvas.com**으로 이미지 요청 및 적재.

**TO-BE (목표)**

- 사람 개입 없이 **Airflow가 데이터 적재 요청부터 이미지 처리·GPU 연산까지** 자동화된 워크플로우로 관리. (Confluence “딱 맞는 디자인 찾기” TO-BE와 동일한 방향.)

### 4.3. 딱 맞는 디자인 찾기 파이프라인 (TO-BE)

- **라벨링 데이터 제외** 시: 파이프라인이 **사람 개입 없이** 자동화되고, **Airflow**가 데이터 적재 요청·이미지 처리·GPU 연산까지 **자동화된 워크플로우**로 관리.

---

## 5. 저퀄리티 분류 모델 데이터셋 정의

- **목적**: 디자인허브에 제출된 요소 중 **저품질 요소가 심사 과정에서 필터링되지 않고 등록**되는 문제 완화.
- **필요 컬럼**: action_item(요소 id), action_item_type, user_pick_cnt, 이미지 url.

### 5.1. 디자인허브 측면

- **승인**: 유지.
- **거부**: 유지 + 승인 후 신고로 거부된 케이스. (신고로 거부된 케이스 CSV는 Slack 등으로 공유.)
- **요소 거부 라벨링 기준**: Confluence [722829784](https://miridih.atlassian.net/wiki/spaces/R4XkdUZCxgHI/pages/722829784).
- **데이터브릭스**: 디자인허브 승인/거부 데이터셋 쿼리·Notebook 경로 문서화 (bronze.miridih_designhub.element_item, content_review_item, element_thumbnail_item 등).

### 5.2. 검탐색 측면 (진행 여부는 팀 정책 따름)

- **고품질**: 최근 6개월, **유저픽 높은 요소 70만 건**.
- **저품질**: **노출 대비 채택률이 낮은 요소 50만 건**.
- **조건**: ‘검색’ 한정 (뷰, 비슷한템플릿찾기, 더보기, 기타 제외).
- **데이터브릭스**: 검탐 요소 품질 분류용 쿼리·Notebook 경로 문서화 (bronze.google_analytics_miricanvas.events, view_element, action_element 등).

---

## 6. 참고 링크

- [딱 맞는 디자인 찾기 - 파이프 라인](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/882737394) — AS-IS/TO-BE, Airflow 자동화 방향.
- [저퀄리티 분류 모델 - 파이프라인](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/882509913) — AS-IS 4단계, TO-BE.
- [저퀄리티 분류 모델 데이터셋 - 디자인 허브](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/869470273) — 문제 정의, 디자인허브/검탐 데이터셋, 데이터브릭스 쿼리.
- [AI-697](https://miridih.atlassian.net/browse/AI-697), [AI-796](https://miridih.atlassian.net/browse/AI-796).

---

*Confluence AIDMLOps·2025 성과아카이브 기준.*
