# analytics-flow-studio

CSV·엑셀 파일을 올리면 **데이터분석처리 7단계**를 거쳐 대시보드와 **분석 보고서**를 만들어 주는 AI 데이터 분석 서비스 (Tableau / Power BI 대안, 기획 단계).

> 서비스명은 가칭입니다. 가격은 확정하지 않았고, 경쟁 서비스 비교는 공식 문서로 재확인이 필요합니다.

## 핵심 흐름

`파일 업로드` → `1 문제 정의` → `2 데이터 수집` → `3 전처리·정제` → `4 EDA` → `5 모델링·심층 분석` → `6 해석·시각화·보고서` → `7 자동화(v2)`

- 사용자 확인 지점 2곳: 목표 확인(1단계), 정리 제안 승인(3단계)
- AI는 계획만 세우고 서버가 검증한 연산만 실행, 원본은 변경하지 않음

## 문서

| 문서 | 내용 |
|---|---|
| [01-product-plan](docs/01-product-plan.md) | 제품 기획, 사용자 흐름, 화면 목록 |
| [02-architecture](docs/02-architecture.md) | 시스템 구조도, ER, 시퀀스, 배포, 보안 |
| [03-flowcharts](docs/03-flowcharts.md) | 전체 흐름, 7단계 상태, 정리 승인, 보고서, 그래프 선택 |
| [04-development-plan](docs/04-development-plan.md) | 린 검증 로드맵, 범위, 6주 마일스톤, 리스크 |

다이어그램은 Mermaid(GitHub에서 바로 표시)이며, 렌더링된 SVG는 [docs/images](docs/images)에 있습니다.

## 디자인 시안

[design/](design/) 폴더에 화면 11종 원본이 있습니다. ([안내](design/README.md))

## 기술 스택(계획)

Next.js (App Router) · shadcn/ui · Tailwind · FastAPI · pandas/numpy/DuckDB · Supabase · Claude API · WeasyPrint/python-docx · n8n(v2)

## 상태

기획·설계 완료, 구현 전. 다음 단계는 수작업(컨시어지) 검증 → 랜딩 사전예약 → 6주 MVP 개발.
