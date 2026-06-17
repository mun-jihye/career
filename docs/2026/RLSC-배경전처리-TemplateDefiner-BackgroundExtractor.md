# RLSC 배경 전처리 파이프라인 (Template Definer · Background Extractor)

> **기간**: 2026 상반기 (진행 중)
> **역할**: 신규 전처리 모듈 2종 설계·구현, LLM 프롬프트 평가·자동 최적화 체계 구축
> **출처**: Confluence [Template Definer 설계](https://miridih.atlassian.net/wiki/spaces/~712020a23635f8190743df9448a03c47cdb090/pages/2503803838), [Background Extractor 설계](https://miridih.atlassian.net/wiki/spaces/~712020a23635f8190743df9448a03c47cdb090/pages/2514715440), [모듈 스키마 정의](https://miridih.atlassian.net/wiki/spaces/TM/pages/2598404266), [BE 프롬프트 성능 평가·개선](https://miridih.atlassian.net/wiki/spaces/~712020dcab33d69e3c4e2c903eddb893220e83/pages/2589294835), repo blueprint

---

## 1. 프로젝트 소개

미리캔버스 템플릿을 AI가 구조화(**RLSC**)하기 전에, **페이지 역할을 분류**하고 **배경 요소와 본문 요소를 분리**하는 LLM 기반 전처리 파이프라인을 신규로 설계·구축한 프로젝트.

- **Template Definer**: 한 템플릿의 페이지 묶음을 받아 페이지별 역할(`Opening / Agenda / SectionDivider / Content / Ending`)을 추론.
- **Background Extractor**: 템플릿 공통 배경(로고·헤더·푸터·반복 도형 등)과 페이지 고유 본문을 분리(`backgroundALSC` / `foregroundALSC`).

두 모듈은 기존 `PageJson → ALSC → RLSC` 파이프라인 앞단에 붙어, RLSC가 **배경이 제거된 본문 중심 입력**으로 동작하게 만든다.

---

## 2. 문제

- 기존 RLSC Structuriser는 한 페이지의 **모든 요소(배경·꾸밈·본문)를 한 번에** 해석한다. 그 결과 모델이 *"무엇이 의미 있는 컴포넌트인가"* 를 판단할 때 **배경·반복 요소가 노이즈로 작용**해 본문(컴포넌트) 추출 정확도를 떨어뜨린다.
- 페이지 역할(`Role.Page`) 판정이 RLSC 추론 내부에 묶여 있어, 표지·목차 같은 **비-content 페이지까지 동일하게 무겁게 처리**된다.

## 3. 가설 설정

- **가설 1 (정확도)**: 반복되는 배경 요소를 **RLSC 실행 전에 먼저 분리·제거**하면, 본문에 집중한 입력이 되어 **컴포넌트 추출 정확도가 올라간다.**
- **가설 2 (효율)**: 페이지 역할을 **앞단에서 먼저 결정**하면, Content 페이지만 후속 처리로 보내 **불필요한 연산을 줄일 수 있다.**
- **검증 전제**: 배경을 잘못 분류해 **본문을 배경으로 빼면 복구 지점이 없으므로**, "배경 제거"가 실제로 안전하고 효과가 있는지는 **PoC + 사람 라벨러 샘플 검수**로 먼저 확인해야 한다. → 이 전제가 뒤의 평가 체계(§4-4)로 이어진다.

---

## 4. 해결 과정

### 4-1. 파이프라인 3모듈 분리

기존 단일 RLSC Structuriser를 **Template Definer(템플릿 단위) → Background Extractor(페이지 단위) → RLSC Structuriser(기존)** 3단계로 분리. 페이지 역할 판정을 Structuriser 내부에서 **Template Definer로 이관**하고, 신 route의 Structuriser는 역할을 다시 추론하지 않도록 책임을 정리했다.

### 4-2. Template Definer 설계 (페이지 역할 분류)

- **입력 최소화**: 페이지 역할은 시각적 레이아웃 패턴으로 충분히 분류 가능하다고 보고 **썸네일 이미지만** 입력(ALSC 미사용). 이미지 detail을 `low`(1장당 고정 85토큰)로 두어 **페이지 수가 늘어도 토큰이 선형·예측 가능**하게 설계.
- **순서 보장 문제 해결**: OpenAI 이미지 입력에는 식별 필드가 없어 멀티 이미지의 순서가 보장되지 않는다. *"순서를 LLM에 부탁하지 않고 `pageIdx` 정수 키로 다시 정렬한다"* 는 원칙으로 **4단계 방어선**(DB 정렬 → 텍스트 라벨·이미지 interleave → Structured Outputs strict 스키마 → 애플리케이션 재조립)을 설계해 누락·중복·unknown을 모두 처리.
- **형식 강제**: OpenAI **Structured Outputs(json_schema strict)** 로 `pageIdx: integer`, `pageRole: enum`을 디코더 단에서 강제해 형식 위반·환각 키를 차단.

### 4-3. Background Extractor 설계 (배경/전경 분리)

- **핵심 설계 — ID 분류 + 애플리케이션 분할**: LLM이 ALSC를 통째로 재생성하면 구조 손실·환각 위험이 크다. 그래서 **LLM은 top-level 노드의 `id`만 background/foreground로 분류**하고, **원본 ALSC를 코드가 결정론적으로 분할**하도록 설계해 **구조적 무결성을 보장**(두 버킷의 합집합 = 입력 ALSC와 1:1 일치).
- **반복성 휴리스틱**: *"같은 템플릿의 여러 페이지에 반복되는 요소 → 배경(slidemaster)"* 를 핵심 신호로, target 페이지 ALSC + target 썸네일 + 템플릿 전체 썸네일을 함께 주입해 모델이 "반복"의 의미를 보게 함.
- **비대칭 위험 반영**: 본문을 배경으로 잘못 빼면 복구가 불가능하므로 *"모호하면 foreground"* 라는 보수적 안전장치를 프롬프트에 명시.
- **운영 안정성**: 템플릿의 모든 페이지를 **DB 1회 조회로 메모리에 적재**(template-batch)해 부하를 줄이고, **한 페이지 실패가 다른 페이지를 막지 않도록** 실패 페이지는 `NULL` ALSC + 에러 메타로 격리 저장. 모든 LLM 호출은 **Langfuse로 추적**.

### 4-4. 평가(Evaluation) 체계 구축

확률적인 LLM 출력을 신뢰하려면 정량 평가가 필수라, 별도 평가 파이프라인을 만들었다.

- **정답(GT) 라벨링**: production 정렬·frozen 데이터셋(**element 12,359개 / 48 template**)에 대해 사람이 element 단위로 layer(배경/전경)·role을 라벨링. 사람이 백지에서 라벨링하던 것을 **계층형 multi-agent judge가 초안을 생성하고 사람이 검수·수정**하는 방식으로 바꿔 속도·일관성을 높임.
- **지표 정의**: 단일 종합 점수(aggregate)와 별개로, **비대칭 위험을 가드레일로 분리**.
  - `layer 관대정확도`(최우선 KPI): 배경/전경 분리 정확도. 사람도 모호한(ambiguous) 요소는 어느 쪽이든 정답 처리.
  - `과제거율(critical)`(가드레일, 낮을수록 좋음): **진짜 배경을 본문으로 오판해 삭제**하는 가장 치명적인 오류. 다른 지표가 좋아도 이 값이 높으면 운영 부적합.
  - 그 외 과소제거율, pagetitle split, per-role precision/recall/F1 등으로 약점을 진단.

### 4-5. 프롬프트 진화: 수동 → 자동 최적화

운영 프롬프트를 단계적으로 개선하며 동일 데이터셋에서 측정.

| 단계 | 종합 | layer(관대) | 과제거(critical) |
|------|------|------------|-----------------|
| v4.0 운영 baseline | 0.874 | 0.913 | **1.04%** |
| v5.0 수동 개선 | 0.868 | 0.894 | 2.86% |
| v6.0 auto optimizer | 0.910 | **0.970** | 1.88% |
| v6.0 @ high·marks·montage | **0.912** | 0.968 | **1.38%** |

- **수동 개선의 함정**: v5에서 pagetitle 조각화(35.8%→2.9%)는 잡았지만 "더 적극적으로 잡는" 방향이라 **과제거가 2.7배 늘고 종합이 하락**. 사람이 한 축만 보고 미는 한계를 확인.
- **자동 최적화로 교정**: 과제거(critical)를 **하드 제약(예산)으로 둔 lexicographic 목적함수**로 프롬프트를 자동 탐색(GEPA·ProTeGi·few-shot 기반 4개 직교 전략 병렬). critical을 지키면서 **layer 0.894→0.970, 과소제거 24.2%→5.1%, 종합 0.868→0.910**으로 끌어올림. 모든 제안은 **사람 검수(HITL) 후 채택**.
- **이미지 옵션**: 같은 프롬프트에 고해상도+set-of-marks+montage를 적용해 **과제거 1.88%→1.38%**로 추가 개선.

### 4-6. ML 대조군 투입 + 비용 고려 성능 고도화

- **ML 대조군(운영 투입 X)**: "프롬프트가 **데이터가 허용하는 상한** 대비 어디인지"를 보기 위해, **logistic regression 기반 분류기**(layer 이진 + role One-vs-Rest, 44개 피처)를 대조군으로 투입. **GroupKFold OOF**(template 단위 분할)로 과적합 없이 평가 → **layer 정확도 93.9% / macro-F1 93.5%**. 단 role(특히 희소 role)·과제거(critical 2.8%)는 약해, **개선 프롬프트(v6)가 종합·과제거 모두 상회**함을 정량적으로 입증.
- **비용 고려 고도화(진행 중)**: LLM은 정확도가 높지만 호출 비용이 든다(Langfuse 실측 **≈ $0.15/page**). ML은 사실상 무료지만 정확도가 낮다. 둘을 합치는 **캐스케이드(ML 신뢰도 임계값 기반으로 ML/LLM 분기)** 를 검토. 단 Background Extractor가 **페이지 단위 호출**이라, element 캐스케이드는 비용 절감이 미미하고 **page 단위 캐스케이드만 실제 비용을 줄일 수 있음**을 분석해 방향을 잡는 중.

| 방식 | 종합 | layer | 과제거 | 비용(LLM 페이지) |
|------|------|-------|--------|------------------|
| 현재 프롬프트(v4) | 0.874 | 0.913 | 1.0% | 100% |
| **개선 프롬프트(v6@high)** | **0.912** | 0.968 | 1.4% | 100% |
| ML 분류기 | 0.859 | 0.942 | 2.8% | **0%** |
| 캐스케이드(개선+ML) | 0.912 | 0.967 | 1.3% | 100% |

---

## 5. 사용 기술

| 구분 | 기술 |
|------|------|
| **LLM** | OpenAI GPT-5.x, Structured Outputs(json_schema strict), 멀티모달(이미지 interleave, detail low/high, set-of-marks·montage) |
| **평가·최적화** | 사람 GT 라벨링, 계층형 multi-agent judge, 자동 프롬프트 최적화(GEPA·ProTeGi·few-shot), lexicographic 목적함수 |
| **ML 대조군** | logistic regression(layer 이진 / role OvR), GroupKFold OOF, 44개 피처 |
| **구현·운영** | TypeScript, Next.js(API route·dev tool), Supabase(결과 테이블·RPC), Langfuse(추적) |

---

## 6. 현재 상태 / 향후

- 진행 중(2026 상반기). 두 모듈 PoC·평가 체계·자동 최적화까지 구축 완료, **비용 대비 성능 고도화(캐스케이드)·GT 확대·BE 에러 해결성 트랙**이 후속 과제.

---

*Confluence MORDOR/TM·RLSC 배경 전처리 관련 문서(문지혜 작성) 및 repo blueprint 기준.*
