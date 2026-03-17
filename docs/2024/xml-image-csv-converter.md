# xml-image-csv-converter 상세 정리

> 레이아웃 자동 생성 모델 연구용 **학습 데이터 생성기** — XML을 파싱해 Image(썸네일·스킨·요소)와 CSV(템플릿 메타데이터)를 생성하는 모듈.  
> **출처**: Confluence [xml-image-csv-converter 1.0](https://miridih.atlassian.net/wiki/spaces/AIDMLOps/pages/584451340), [개인 스페이스 xml-image-csv-converter 폴더](https://miridih.atlassian.net/wiki/spaces/~712020a23635f8190743df9448a03c47cdb090/folder/713565403)

---

## 1. 개요

| 항목 | 내용 |
|------|------|
| **목적** | 레이아웃 자동 생성 모델 연구에 필요한 학습용 데이터 생성 |
| **입력** | XML (`thumbnail.xml`, `skin.xml`, `elements.xml`) 또는 template idx가 들어 있는 CSV |
| **출력** | Image(thumbnail, skin, element), CSV(템플릿 메타데이터) |
| **참여자** | 문혁준, 문지혜, 장주영 |

---

## 2. 아키텍처·처리 흐름

### 2.1. 입력(Input)

- **template_idx(csv)** 또는 **xml_file_path** 로 구분
  - **template idx인 경우**: API 요청으로 XML string 획득
  - **xml_file_path인 경우**: 해당 파일 경로에서 XML string 읽기
- XML 파싱 결과: `thumbnail.xml`, `skin.xml`, `elements.xml` 로 분리·활용

### 2.2. Process 세 가지

1. **CSV·Image 저장**
   - Input: thumbnail, skin, element XML
   - **CSV**: XML 파싱 후 필요한 데이터만 추출해 CSV로 저장
   - **Image**: Puppeteer로 XML 렌더링 후 이미지 저장

2. **Template image 렌더링(미리보기)**
   - Input: thumbnail XML
   - Puppeteer로 렌더링 후, 파일로 저장하지 않고 브라우저에 바로 표시

3. **Element Image 추출** (별도 프로세스)
   - XML 확보 → 파싱(xmlParser.py) → element별 position 노드로 크기 확보(xmlEditor.py) → element별 XML 생성(getXmlByElement.py) → 이미지 추출
   - **파일명 규칙**: `템플릿ID_페이지번호_요소번호.png` (priority 기준 순서)
   - xml_file_path로 주어진 경우 템플릿 ID 대신 파일명 사용
   - **그룹**: `group = true` → 그룹 단위 캡처, `group = false` → 그룹 없이 모든 요소 캡처

---

## 3. 모듈 사용법 (CLI 명령어)

### 3.1. Save Process

| 명령어 | 설명 | 인자 |
|--------|------|------|
| **`xml-save-process`** | XML이 저장된 폴더 경로를 입력으로 image + csv 저장 | 1) XML 폴더 경로 2) 이미지 저장 폴더 3) CSV 저장 경로·파일명 4) 그룹화 여부(True/False, 기본 True) |
| **`idx-save-process`** | template idx가 들어 있는 CSV를 입력으로 image + csv 저장 | 1) template idx CSV 경로 2) 이미지 저장 폴더 3) CSV 저장 경로·파일명 |

**예시**

```bash
xml-save-process /Users/miridih/Documents/package-test/input_xml ./output_image ./output_csv/241016.csv True
idx-save-process /Users/miridih/Documents/python_typescript/input/원본.csv ./output_image ./output_csv/241016.csv
```

### 3.2. Visualize Process

| 명령어 | 설명 | 인자 |
|--------|------|------|
| **`xml-visualize-process`** | XML 폴더 경로를 받아 브라우저로 렌더링 | 1) XML 폴더 경로 |
| **`idx-visualize-process`** | template idx CSV를 받아 브라우저로 렌더링 | 1) template idx CSV 경로 |

**예시**

```bash
xml-visualize-process /Users/miridih/Documents/python_typescript/input_xml
idx-visualize-process /Users/miridih/Documents/python_typescript/input/원본.csv
```

---

## 4. CSV Format (template metadata)

- xml-image-csv-converter **output CSV**는 템플릿 메타데이터를 담으며, 필요에 따라 스키마가 변경됨.

### 4.1. 주요 컬럼

| 컬럼 | 설명 |
|------|------|
| template_idx | 템플릿 인덱스 |
| template_page_idx | 템플릿 내 페이지 인덱스 |
| sheet_key | sheet.xml 경로 식별자 (예: `s3://bucket/.../sheet_key/sheet.xml`) |
| thumb_key | thumb.webp 경로 식별자 |
| page_num | 페이지 번호 |
| background | 배경 여부 |
| image_file_path | 이미지 저장 경로 (예: `/data/shared/template_page/{sheet_key}/1.png`) |
| img_width_resized, img_height_resized | 리사이즈된 이미지 크기 |
| tag | 요소 태그 |
| is_text | 텍스트 여부 |
| text_content | 텍스트 내용(속성별/그룹별로 리스트·다중 리스트 가능) |
| tbpeID | 요소 고유값 |
| resourceKey | 비텍스트 요소 공통 식별자 |
| left, top | 요소 위치 |
| img_width, img_height | 원본 크기 |
| priority, rotation, opacity | 우선순위, 회전, 투명도 |
| vertical_align, text_align | 텍스트 세로/가로 정렬 |
| font_size, font_type, bold, italic, color | 폰트·스타일 |
| group_id, group_elements_info, group_status | 그룹 시 group / in_group / out_group 등 |

### 4.2. 그룹·복합 요소

- **그룹**: 해당 컬럼은 리스트(또는 다중 리스트)로 나열.
- **GRID, FrameItem, SvgImageFrame**: 한 요소가 내부에 다른 요소(photo, simple_text 등)를 포함하는 경우, 그룹과 유사하게 리스트로 여러 값 나열.  
  - img_width_resized, img_height_resized는 **외부 요소** 기준.  
  - tag, text_content, left, top 등은 리스트로 저장.

---

## 5. &lt;SIMPLE_TEXT&gt; 태그 속성값

- **t**: 텍스트 타입 (`b`: 블록)
- **c**: 콘텐츠(여러 스타일 속성 포함)
- **st**: 스타일 타입 (`p`: 문단)
- **priority**, **opacity**: SIMPLE_TEXT 태그에서 추출

### rp 내부 속성

| 속성 | 의미 |
|------|------|
| fill | font color |
| size | font size |
| bkgr | background color |
| fmly | font family |
| wght | font weight (bold) |
| styl | italic (0/1) |
| ltSp | 글자 간격 |
| deco | 밑줄·줄간격 (udln 0/1 등) |
| strk | 외곽선 |
| eff | 효과(활성화, 각도, 거리, 블러) |

### bp 내부 속성

| 속성 | 의미 |
|------|------|
| vtln | 세로 정렬 (0 상단, 1 가운데, 2 하단) |
| txal | 가로 정렬 (0 왼쪽, 1 가운데, 2 오른쪽, 3 양쪽) |

---

## 6. 배포·버전 (Deploy)

- **0.0.65**: is_group 디폴트값 수정, 리드미(natsort 의존성) 추가  
- **0.0.66**: visualize 타입 오류 수정, 요소(csv) image_file_name 형식 수정  
- **0.0.67**: rotate 적용 요소 깨짐 해결, tqdm 추가  
- **0.0.69** ~ **0.0.70**: node 배치·예외 처리  
- **0.0.74**: 썸네일 저장 배치 구현  

*(상세는 Confluence [Deploy] xml-image-csv-converter 배포 상황 페이지 참고)*

---

## 7. 개발 이력 (Branch·기능)

- **main**: 빌드·배포 브랜치  
- **feature/develop-xxx**: 기능 단위 개발 (일부 예시)
  - 002: node로 XML 넘길 때 json 파일 생성
  - 004: 요소 리사이즈 로직
  - 007: text일 때 scale 값을 노드로 전달
  - 008: thumbnail만 추출
  - 009: priority 중복 시 파일명 구분
  - 010~013: 그룹·다중 그룹 이미지/CSV 추출, 파일명·CSV 일치
  - 014~024: 중복 파일명, priority 중복(FrameItem/Grid), 회전·그룹 회전, SvgImageFrame 내부 photo 등
  - 025~035: CSV 텍스트 유니코드, rotate 여백 제거, bound box/align/폰트 추출, FrameItem 내부 video 등
  - 036~040: template idx 기반 로직, S3 경로 접근·다운·업로드, saveThumbnailByXml / saveProcessByXml / saveProcessByUrl 자동화
  - 041~048: Python–Node Docker, EC2, 이미지/요소 속성 CSV, Docker 패키징, template_page_idx, p-limit, ECR 자동 배포 등  

*(Jira 이슈 기반 브랜치 네이밍 예: feature/AI-000)*

---

## 8. 사용 기술·키워드

- **입·출력**: XML 파싱, CSV 입·출력, Image 생성·저장  
- **렌더링**: Puppeteer  
- **구조**: Python + Node 연동, 모듈화 CLI  
- **인프라**: Docker, EC2, S3, ECR  
- **데이터**: 템플릿 메타데이터, 그룹/다중 그룹, SIMPLE_TEXT·GRID·FrameItem·SvgImageFrame 등 XML 구조 반영  

---

*Confluence AIDMLOps 스페이스·개인 스페이스 문서 기준 정리.*
