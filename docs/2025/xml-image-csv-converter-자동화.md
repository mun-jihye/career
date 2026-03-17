# xml-image-csv-converter 자동화 (Airflow, EC2, ECR, AWS Batch)

> 대량 XML→Image·CSV 변환 작업을 **수동 EC2 운영**에서 **컨테이너·스케줄 기반 자동화**로 전환한 빌드업과 구현 상세.  
> **출처**: Confluence [아키텍쳐 변경 - ec2](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/979599696), [docker image 빌드 및 ecr 업로드 자동화](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/959645183), 2025 하반기 셀프리뷰

---

## 1. 빌드업 요약 (어떤 점이 어떻게 됐는지)

| 단계 | 상태 | 설명 |
|------|------|------|
| **1) 초기** | AS-IS | **EC2 인스턴스 수동 기동·종료**. 대량 추출 시 r7i.xlarge 최대 **6대** 직접 띄워 병렬 처리. 작업 단위 스케줄링·실패 재시도·리소스 회수 없음, 운영 개입·인프라 관리 부담 큼. |
| **2) 컨테이너화** | 전환 | xml-image-csv-converter 모듈을 **Docker 이미지**로 패키징. **ECR**에 이미지 저장. 로컬에서는 Rancher Desktop으로 빌드·푸시 후 서버에서 pull 해서 실행. |
| **3) CI/CD** | 도입 | **GitHub Actions**로 `develop` 브랜치 푸시 시 **Docker 이미지 빌드 → ECR 업로드** 자동화. 수동 빌드·태그·푸시 제거. |
| **4) 오케스트레이션 (1차)** | EKS → EC2 전환 | 원래는 **Airflow + EKS**에서 KubernetesPodOperator로 작업 실행 목표였으나, **EKS 연동·IAM·네트워크 등 구축 일정이 부담**되어 **Airflow + EC2** 방식으로 변경. EC2 상시 기동, 내부에 Docker 환경 구성, **Airflow DAG에서 EC2에 접근해 ECR 이미지 pull 후 컨테이너 실행**. |
| **5) 오케스트레이션 (2차)** | AWS Batch | **EC2 직접 운영 제거**. **AWS Batch** 도입으로 작업을 **컨테이너 Job** 단위 실행. 작업 수 증가 시 Batch가 컴퓨팅 리소스 **자동 확장**, 작업 종료 후 **리소스 자동 해제**. **Airflow DAG**로 배치 실행 시점·재시도·순차/병렬 제어. |
| **6) 결과** | TO-BE | 대량 XML→CSV 변환을 **일배치** 형태로 안정 자동화. 수작업·임시 스크립트 운영 제거. 규모 증가 시 자동 확장으로 대응. |

---

## 2. 내가 한 작업 (역할)

- **아키텍처 변경 설계·문서화**: EKS → EC2 전환 사유(연동 미완료, 일정) 정리 및 “EC2 + Docker + Airflow가 ECR 이미지 pull 후 컨테이너 실행” 방식 문서 작성.
- **Docker 이미지 빌드·ECR 업로드 자동화**: GitHub Actions 워크플로우 설계·작성 (`develop` 푸시 시 빌드·ECR 푸시), OIDC·IAM Role(`de-xml-image-csv-converter-github`) 설정, `linux/arm64` 플랫폼·캐시 옵션 적용.
- **Dockerfile 최적화**: 레이어 순서(ENV→시스템 패키지→Node→requirements.txt→소스), requirements.txt 분리 복사, GitHub Actions `--cache-from/--cache-to type=gha` 적용으로 **빌드 시간 단축** (AS-IS 15분 44초 → TO-BE 13분 45초, 재빌드 시 캐시로 약 80% 단축).
- **Airflow DAG 설계**: XML→image·csv 변환 작업을 DAG로 정의, 실행 시점·실패 재시도·순차/병렬 제어.
- **AWS Batch 도입**: EC2 직접 기동 제거, Batch Job으로 컨테이너 실행, 작업 규모에 따른 자동 확장·종료 반영.

---

## 3. 사용한 툴

| 구분 | 툴 |
|------|-----|
| **빌드·이미지** | Docker, Rancher Desktop(로컬), Dockerfile, buildx |
| **레지스트리** | AWS ECR (ap-northeast-2, `010928203580.dkr.ecr.ap-northeast-2.amazonaws.com/de-xml-image-csv-converter`) |
| **CI/CD** | GitHub Actions (checkout, setup-buildx, configure-aws-credentials, amazon-ecr-login, build-and-push) |
| **인증** | AWS OIDC(GitHub), IAM Role `de-xml-image-csv-converter-github`, `AmazonEC2ContainerRegistryPowerUser` |
| **오케스트레이션** | Apache Airflow (DAG), AWS Batch (Job 실행·자동 확장) |
| **실행 환경** | EC2(과도기), AWS Batch(최종) — 컨테이너 기반 |

---

## 4. 워크플로우

### 4.1. Docker 이미지 빌드·ECR 업로드 (CI/CD)

```
develop 브랜치 push
  → GitHub Actions 트리거
  → Checkout / Set up Docker Buildx
  → Configure AWS credentials (OIDC Role)
  → Login to Amazon ECR
  → docker buildx build --platform linux/arm64
       --cache-from type=gha --cache-to type=gha,mode=max
       --tag ${ECR_URL}:latest --tag ${ECR_URL}:develop-${run_number}
       --push .
  → ECR에 latest / develop-N 태그로 이미지 푸시
```

- **로컬 수동** 시: `aws ecr get-login-password` → `docker build --platform linux/arm64` → `docker tag` → `docker push`.

### 4.2. Airflow + 실행 환경 (EC2 경유 → Batch)

- **EC2 방식(과도기)**: Airflow DAG에서 EC2에 접근 → 해당 EC2에서 `docker pull` (ECR) → 컨테이너 실행 → 작업 완료 후 정리.
- **Batch 방식(최종)**: Airflow DAG에서 **AWS Batch Job** 제출 → Batch가 컴퓨팅 리소스 할당 → ECR 이미지 pull → 컨테이너 실행 → 완료 시 리소스 자동 해제. DAG로 실행 시점·재시도·순차/병렬 제어.

---

## 5. Dockerfile·빌드 최적화 요약

| 개선 항목 | AS-IS | TO-BE |
|-----------|--------|--------|
| 레이어 순서 | COPY . 후 pip install (코드 변경 시 대부분 레이어 무효) | ENV→시스템 패키지→Node→COPY requirements.txt→pip install→COPY 소스 (변경 빈도 낮은 순) |
| requirements | 소스와 함께 복사 | requirements.txt만 먼저 복사 후 `pip install -r` |
| 캐시 | 없음 | `--cache-from type=gha` / `--cache-to type=gha,mode=max` |
| 빌드·푸시 | build → tag → push 3단계 | `--tag ... --tag ... --push` 한 번에 빌드·푸시 |

---

## 6. 참고 Confluence·링크

- [아키텍쳐 변경 - ec2](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/979599696) — EKS→EC2 전환 사유, EC2+Docker+Airflow 구조.
- [docker image 빌드 및 ecr 업로드 자동화](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/959645183) — 로컬 명령어, GitHub Actions 워크플로우, Dockerfile 최적화, 빌드 시간 비교.
- [Rancher Desktop 사용 방법](https://miridih.atlassian.net/wiki/spaces/8jaR9OEPTa9X/pages/457834548) (로컬 빌드 시).

---

*2025 하반기 셀프리뷰·Confluence AIDMLOps 문서 기준.*
