# 디자인 시안 원본

Claude 디자인 캔버스에서 만든 화면 원본(`*.dc.html`)과 배치 정보(`canvas.json`)입니다.
캔버스 런타임(`support.js`)은 포함되어 있지 않아 단독으로 열면 표시되지 않을 수 있습니다. 구현 시 참고용 원본으로 사용하세요.

| 파일 | 화면 | 경로 |
|---|---|---|
| Home | 대시보드 | /dashboard |
| Upload | 업로드 | /upload |
| Projects | 프로젝트 | /projects |
| Steps | 데이터분석단계(7단계 애니메이션) | /projects/[id]/analysis |
| Check | 정리 제안 | /projects/[id]/analysis/cleaning |
| Main | 프로젝트 상세 | /projects/[id] |
| Generated | 생성된 대시보드 | /projects/[id]/dashboard |
| Report | 보고서 | /projects/[id]/report |
| Charts | 그래프 선택 규칙 | 내부 |
| DashboardTablet / DashboardMobile | 반응형 | - |

참고: 샘플 수치(예: ROAS)는 화면마다 일부 다릅니다(4.8 / 4.9). 예시 값입니다.
