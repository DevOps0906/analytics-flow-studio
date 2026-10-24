# 02. 시스템 구조도 (Architecture)

> 원칙: **AI는 계획만 세우고, 서버가 검증한 연산만 실행한다.** 원본은 바꾸지 않고 버전으로 쌓는다. 진행 상황은 실시간으로 화면에 보여준다.

## 1. 전체 구조

```mermaid
flowchart TB
  subgraph Client["클라이언트 (Next.js App Router · shadcn/ui · Tailwind)"]
    UI1["대시보드 / 프로젝트 / 업로드"]
    UI2["데이터분석단계 (7단계 진행 화면)"]
    UI3["전처리·정제 (제안 승인)"]
    UI4["프로젝트 상세 · 질문 창 · 리포트 미리보기"]
  end

  subgraph Edge["Next.js 서버 (BFF)"]
    API["Route Handlers / Server Actions<br/>인증 확인 · 업로드 URL 발급 · 요청 전달"]
  end

  subgraph Supabase["Supabase"]
    AUTH["Auth"]
    DB[("Postgres<br/>프로젝트·실행·제안·리포트 메타")]
    STORE[("Storage<br/>원본 / 정제본(Parquet) / 리포트 파일")]
    RT["Realtime<br/>단계 진행 상태 푸시"]
  end

  subgraph Analysis["분석 서비스 (FastAPI · Pydantic)"]
    ORCH["Run Orchestrator<br/>7단계 실행·재시도·체크포인트"]
    S1["1 목표 정의"]
    S2["2 수집·프로파일링"]
    S3["3 전처리·정제"]
    S4["4 EDA·그래프 추천"]
    S5["5 모델링·집계"]
    S6["6 해석·시각화·리포트"]
    S7["7 파이프라인·스케줄"]
    ENGINE["연산 엔진<br/>pandas · numpy · DuckDB"]
    VALID["계획 검증기<br/>스키마 + 허용 연산 화이트리스트"]
  end

  subgraph LLM["AI 게이트웨이"]
    CL["Claude API<br/>계획(JSON)·인사이트·질문 답변"]
    GUARD["프롬프트 템플릿 · 토큰 한도 · 캐시"]
  end

  subgraph Docs["문서 생성"]
    PDF["WeasyPrint (PDF)"]
    DOCX["python-docx (Word)"]
    IMG["차트 이미지 (matplotlib, 보고서 전용)"]
  end

  subgraph Auto["자동화 (v2)"]
    N8N["n8n<br/>스케줄·웹훅·알림"]
  end

  UI1 & UI2 & UI3 & UI4 --> API
  API --> AUTH
  API --> DB
  API -->|"서명된 업로드 URL"| STORE
  API -->|"실행 요청"| ORCH
  ORCH --> S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7
  S2 & S3 & S4 & S5 --> ENGINE
  ENGINE <--> STORE
  S1 & S3 & S6 --> GUARD --> CL
  CL -->|"계획 JSON"| VALID
  VALID -->|"허용된 연산만"| ENGINE
  ORCH --> DB
  ORCH --> RT --> UI2
  S6 --> IMG --> PDF & DOCX
  PDF & DOCX --> STORE
  N8N -->|"웹훅: 재실행"| API
  S7 --> N8N
```

### 구성요소 역할

| 구성요소 | 역할 | 선택 이유 |
|---|---|---|
| Next.js (App Router) | 화면, 인증 세션, BFF | 기존 스택과 동일, 서버 컴포넌트로 초기 로딩 단축 |
| shadcn/ui + Tailwind | 디자인 시스템(시안과 동일 토큰) | 빠른 UI, 접근성 기본 제공 |
| FastAPI + Pydantic | 분석 API, 계획 검증 | pandas·DuckDB 생태계, 타입 검증 |
| pandas / numpy | 전처리·통계 | 표준 |
| DuckDB | 큰 CSV를 SQL로 집계 | 파일 기반이라 서버 구성이 단순 |
| Supabase | 인증·DB·파일·실시간 | 1인 운영에 적합, RLS로 데이터 격리 |
| Claude API | 정리 제안·인사이트·질문 답변 | JSON 출력 + 긴 문맥 |
| WeasyPrint / python-docx | PDF·Word | 기존 출판 파이프라인 경험 재사용 |
| n8n | 스케줄·알림(7단계) | 코드 없이 자동화, v2 |

## 2. AI 계획 → 검증 → 실행 구조 (핵심 안전장치)

```mermaid
flowchart LR
  A["프로파일 요약<br/>(컬럼·통계·샘플 일부)"] --> B["Claude: 분석/정리 계획 JSON"]
  B --> C{"스키마 검증<br/>(Pydantic)"}
  C -- 실패 --> B2["오류 내용과 함께 1회 재요청"] --> B
  C -- 통과 --> D{"허용 연산인가?<br/>(화이트리스트)"}
  D -- 아니오 --> E["해당 항목 제외 + 사용자에게 사유 표시"]
  D -- 예 --> F["미리보기 계산<br/>(변경 전·후 샘플)"]
  F --> G["사용자 승인"]
  G --> H["엔진 실행 → 새 데이터 버전 저장"]
  H --> I["변경 이력(change_log) 기록"]
```

- **허용 연산 예시**: 날짜 형식 변환, 문자열 정규화(대소문자·공백), 중복 제거, 결측 채움(0/평균/중앙값/제외), 타입 변환, 이상치 표시/제외, 컬럼 이름 변경.
- **금지**: 임의 코드 실행, 외부 네트워크 호출, 파일 시스템 접근.
- AI에 보내는 데이터는 **요약 통계와 소수의 샘플 행**으로 제한합니다(전체 원본 전송 금지, 개인정보 컬럼은 마스킹 옵션).

## 3. 데이터 모델 (ER)

```mermaid
erDiagram
  PROFILES ||--o{ PROJECTS : owns
  PROJECTS ||--o{ DATASETS : has
  DATASETS ||--o{ DATASET_VERSIONS : versions
  PROJECTS ||--o{ ANALYSIS_RUNS : runs
  ANALYSIS_RUNS ||--|{ RUN_STEPS : "7 steps"
  ANALYSIS_RUNS ||--o{ CLEANING_PROPOSALS : proposes
  CLEANING_PROPOSALS ||--o{ CHANGE_LOGS : applied
  ANALYSIS_RUNS ||--o| DASHBOARDS : produces
  DASHBOARDS ||--o{ WIDGETS : contains
  ANALYSIS_RUNS ||--o{ REPORTS : produces
  REPORTS ||--o{ REPORT_SECTIONS : contains
  PROJECTS ||--o{ PIPELINES : saves
  PIPELINES ||--o{ SCHEDULES : triggers
  PROFILES ||--o{ USAGE_EVENTS : consumes

  PROJECTS { uuid id PK
    text name
    text status
    jsonb goal }
  DATASETS { uuid id PK
    text file_name
    text file_type
    jsonb profile }
  DATASET_VERSIONS { uuid id PK
    int version
    text storage_path
    text kind }
  ANALYSIS_RUNS { uuid id PK
    text status
    int current_step
    timestamptz started_at }
  RUN_STEPS { uuid id PK
    int step_no
    text status
    jsonb output
    text error }
  CLEANING_PROPOSALS { uuid id PK
    text kind
    jsonb plan
    jsonb preview
    text decision }
  CHANGE_LOGS { uuid id PK
    jsonb detail
    int affected_rows }
  DASHBOARDS { uuid id PK
    jsonb spec }
  WIDGETS { uuid id PK
    text chart_type
    jsonb config }
  REPORTS { uuid id PK
    int version
    text pdf_path
    text docx_path }
  REPORT_SECTIONS { uuid id PK
    text kind
    jsonb content
    int sort }
  PIPELINES { uuid id PK
    jsonb settings }
  SCHEDULES { uuid id PK
    text cron
    text channel }
  USAGE_EVENTS { uuid id PK
    text kind
    int amount }
```

핵심 규칙
- 모든 테이블에 `owner_id`(또는 프로젝트 경유) 기준 **RLS(행 수준 보안)** 적용.
- `dataset_versions.kind`: `raw`(원본) / `cleaned`(정제) — **원본은 수정 불가**, 정제는 새 버전.
- `run_steps.output`에 단계 산출물(JSON)을 저장하고 보고서는 이 값을 조립해서 만듭니다.
- `cleaning_proposals.decision`: `pending / applied / skipped`.

## 4. 실행·실시간 진행 구조

```mermaid
sequenceDiagram
  autonumber
  actor U as 사용자
  participant W as Web (Next.js)
  participant S as Supabase
  participant A as Analysis (FastAPI)
  participant L as Claude API

  U->>W: 파일 업로드 → "데이터분석하기"
  W->>S: 원본 저장 (Storage) · run 생성 (DB)
  W->>A: POST /runs/{id}/start
  A->>S: step1 running (Realtime으로 화면 갱신)
  A->>L: 컬럼 요약 → 목표 후보
  A-->>W: 목표 확인 요청 (체크포인트)
  U->>W: 목표 확인
  W->>A: POST /runs/{id}/goal
  A->>A: step2 수집·프로파일링
  A->>L: 프로파일 → 정리 계획 JSON
  A->>A: 검증 + 미리보기 계산
  A-->>W: 정리 제안 (체크포인트)
  U->>W: 제안별 적용/건너뛰기
  W->>A: POST /runs/{id}/proposals
  A->>A: step3 실행 → step4 EDA → step5 분석
  A->>L: 결과 요약 → 인사이트 문장(근거 포함)
  A->>A: step6 대시보드 spec · 보고서 조립
  A->>S: 리포트 파일 저장 · run 완료
  S-->>W: Realtime: 완료
  W-->>U: 대시보드 + 보고서 열기
```

- 단계 상태는 `pending → running → waiting_user → done / failed` 로 관리하며, 화면의 7단계 진행 애니메이션이 이 값을 그대로 표시합니다.
- 실패 시 해당 단계부터 **재시도**할 수 있고, 이전 단계 산출물은 재사용합니다.

## 5. 배포 구조

```mermaid
flowchart LR
  Dev["GitHub (analytics-flow-studio)"] -->|"push"| Vercel["Vercel<br/>apps/web"]
  Dev -->|"push"| Container["컨테이너 배포<br/>services/analysis (FastAPI)"]
  Vercel --> SB["Supabase (Auth·DB·Storage)"]
  Container --> SB
  Container --> Claude["Claude API"]
  Container --> Sentry["오류 추적 · 로그"]
  Vercel --> Sentry
```

| 환경 | 용도 | 비고 |
|---|---|---|
| local | 개발 | Supabase CLI, 로컬 FastAPI |
| preview | PR 확인 | Vercel Preview + 테스트용 Supabase 브랜치 |
| production | 서비스 | 비밀키는 환경변수, 서비스 키는 서버에만 |

## 6. 보안·운영 기준

| 항목 | 기준 |
|---|---|
| 데이터 격리 | RLS 필수, 스토리지 경로에 사용자 ID 포함, 서명 URL 만료 짧게 |
| 업로드 | 형식 CSV·XLSX, 파일당 50MB(초기), 확장자·MIME·내용 검사, 매크로(xlsm) 거부 |
| AI 전송 | 요약·샘플만, 개인정보 컬럼 마스킹 옵션, 프롬프트·응답 로그에서 원본 값 제외 |
| 실행 안전 | 허용 연산만, 시간·메모리 제한, 실행 단위 타임아웃 |
| 비용 통제 | 사용자별 AI 호출 한도, 같은 프로파일 결과 캐시 |
| 백업·삭제 | 프로젝트 삭제 시 파일·버전 즉시 삭제 옵션, 정기 DB 백업 |
| 관측 | 단계별 소요시간·실패율·AI 토큰 사용량 기록 |
