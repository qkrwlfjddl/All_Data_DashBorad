# Sales Dashboard

9001 원본 테이블(챔프스터디 매출) 하나로 **연간 성과 · 일간 분석 · 판매상품 분석 · 이벤트 · 교재 순위 · GA 성과 · 검색 트렌드 · AI 분석 리포트**를 한 화면에서 보는 Streamlit 대시보드.


## 프로젝트 구조

```
sales-dashboard/
├── app.py                 # Streamlit 앱 (전체 탭 + OAuth 인증)
├── requirements.txt       # 앱 의존성
├── Dockerfile             # Cloud Run 배포용 컨테이너
├── .dockerignore          # Docker 빌드 제외 목록
├── .gcloudignore          # Cloud Build 업로드 제외 목록
├── cloudbuild.yaml        # CI/CD 자동 배포 (선택)
├── .streamlit/config.toml # Streamlit 설정
├── sql/                   # 참조용 SQL (뷰 DDL 등)
├── dash_board/            # GA ETL 잡 (Cloud Run Job)
    ├── run_ga_etl.py
    ├── Dockerfile
    └── requirements.txt

```

## 문서

| 문서 | 내용 |
|------|------|
| [01. 매출 성과 대시보드](docs/01_매출_성과_대시보드.md) | 탭 구성, 매출 집계 기준, 사이드바 조건, 실행 방법 |
| [02. 검색어 트렌드](docs/02_검색어_트렌드.md) | 네이버 API 연동, 프리셋 저장, 키 발급 방법 |
| [03. AI 분석 리포트](docs/03_AI_분석_리포트.md) | Vertex AI 연결, 8개 섹션, 담당자 의견 |
| [04. GA ETL 잡 배포](docs/04_GA_ETL_잡_배포.md) | Cloud Run Job + Scheduler, 백필, 사업군 추가 |
| [05. 대시보드 배포](docs/05_대시보드_Cloud_Run_배포.md) |  |
| [06. 인증](docs/06_인증_및_사용자_권한.md) |  |


## 배포 & 접근 제어

앱은 **Cloud Run**에 배포되고, **Google OAuth 로그인 + 사업군별 접근 제어**로 보호됩니다.

- 배포 방법: [docs/05_대시보드_Cloud_Run_배포.md](docs/05_대시보드_Cloud_Run_배포.md)
- 인증 & 사용자 관리: [docs/06_인증_및_사용자_권한.md](docs/06_인증_및_사용자_권한.md)

**서비스 URL:** `https://sales-dashboard-397756181081.asia-northeast3.run.app`

빠른 재배포 (코드 수정 후):
```bash
export PROJECT_ID="ga4-bigquery-431807"
cd ~/sales-dashboard
gcloud builds submit --tag asia-northeast3-docker.pkg.dev/$PROJECT_ID/sales-dashboard/app:latest
gcloud run deploy sales-dashboard \
  --image=asia-northeast3-docker.pkg.dev/$PROJECT_ID/sales-dashboard/app:latest \
  --region=asia-northeast3 --platform=managed
```

## 데이터 소스 요약

| 데이터 | 위치 | 적재 |
|--------|------|------|
| 매출 (9001) | `HANABQ.ZTIF9001` | 하나DB → BQ (매일) |
| 이벤트 (9009) | `HANABQ.ZTIF9009` | 하나DB → BQ (매일) |
| 교재 순위 | `lab_asia.BSM_KWD` | 별도 크롤러 |
| GA 방문자/성과 | `DASHBOARD.GA4_DAY_TRAFFIC/PERFORM` | ETL 잡 (매일 07:00) |
| 트렌드 프리셋 | `DASHBOARD.TREND_PRESETS` | 앱에서 저장 |
| 사용자 권한 | `DASHBOARD.AUTH_USERS` | 사용자 관리 탭에서 저장 |
| 검색 트렌드 | 네이버 API (실시간) | 조회 시 호출 |
| AI 분석 | Vertex AI (실시간) | 버튼 클릭 시 호출 |

