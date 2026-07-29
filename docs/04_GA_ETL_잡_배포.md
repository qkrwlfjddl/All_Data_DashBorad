# 04. GA4 ETL 잡 & 배포

GA4 원본(`analytics_343571535.events_*`)에서 하루치를 집계하여
요약 테이블에 적재하는 Cloud Run Job + Scheduler 구성.

## 아키텍처

```
Cloud Scheduler (매일 07:00 KST)
    → Cloud Run Job (dash-board)
      → run_ga_etl.py
        → DASHBOARD.GA4_DAY_TRAFFIC 갱신
        → DASHBOARD.GA4_DAY_PERFORM 갱신
        (× 사업군 수만큼 반복)
```

## 요약 테이블

| 테이블 | 용도 | 컬럼 |
|--------|------|------|
| `DASHBOARD.GA4_DAY_TRAFFIC` | 연간 방문자수 | bsark, dt, device_category, visitors, sessions, pageviews |
| `DASHBOARD.GA4_DAY_PERFORM` | GA 성과 | bsark, dt, source, medium, sessions, purchase_cnt, revenue, signup_cnt |

- `PARTITION BY dt CLUSTER BY bsark`
- 잡이 테이블 자동 생성 (CREATE IF NOT EXISTS)
- 멱등 방식 (같은 날짜 재실행해도 중복 없음)

## 사업군 추가

`dash_board/run_ga_etl.py`의 `GA_SOURCES`에 한 줄 추가:

```python
GA_SOURCES = [
    {"bsark": "3310", "dataset": "analytics_343571535"},
    # {"bsark": "3080", "dataset": "analytics_XXXXXXXXX"},  ← 추가
]
```

## 사용법

### 최초 백필 (Cloud Shell)

```bash
cd dash_board
pip install -r requirements.txt
python run_ga_etl.py --start 2024-09-20 --end 2026-07-28
```

특정 사업군만: `--bsark 3310`

### 이미지 빌드

```bash
cd dash_board
gcloud builds submit --tag asia-northeast3-docker.pkg.dev/ga4-bigquery-431807/kpi-repo/dash-board
```

### Cloud Run Job 생성/업데이트

최초:
```bash
gcloud run jobs create dash-board \
  --image asia-northeast3-docker.pkg.dev/ga4-bigquery-431807/kpi-repo/dash-board \
  --region asia-northeast3 \
  --tasks 1 \
  --max-retries 2 \
  --task-timeout 1800 \
  --service-account ga4-bq-activeusers-bot@ga4-bigquery-431807.iam.gserviceaccount.com
```

업데이트 (코드 변경 시):
```bash
gcloud run jobs update dash-board \
  --image asia-northeast3-docker.pkg.dev/ga4-bigquery-431807/kpi-repo/dash-board \
  --region asia-northeast3
```

### Cloud Scheduler

```bash
gcloud scheduler jobs create http dash-board-daily \
  --location asia-northeast3 \
  --schedule "0 7 * * *" \
  --time-zone "Asia/Seoul" \
  --uri "https://asia-northeast3-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/ga4-bigquery-431807/jobs/dash-board:run" \
  --http-method POST \
  --oauth-service-account-email ga4-bq-activeusers-bot@ga4-bigquery-431807.iam.gserviceaccount.com
```

## 서비스 계정 권한

`ga4-bq-activeusers-bot@ga4-bigquery-431807.iam.gserviceaccount.com`:
- `roles/bigquery.jobUser` (쿼리 실행)
- `roles/bigquery.dataEditor` on DASHBOARD (요약 테이블 쓰기)
- `roles/bigquery.dataViewer` on GA 원본 데이터셋 (읽기)

## 비용

| 단계 | 스캔량 | 빈도 | 비용 |
|------|--------|------|------|
| 백필 (전체) | 2024.09~현재 | 1회 | 수 달러 |
| 매일 증분 | 어제 1일치 | 하루 1번 | 1센트 미만/일 |
| 앱 조회 | 요약 수 MB | 볼 때마다 | 거의 0 |

## 파일 구조

```
dash_board/
├── run_ga_etl.py      # ETL 잡 스크립트
├── Dockerfile         # Cloud Run Job 컨테이너
├── requirements.txt   # google-cloud-bigquery
└── README.md          # 배포 명령 요약
```

## 데이터셋 변경 이력

- 초기: `lab_asia.GA4_DAY_TRAFFIC` / `GA4_DAY_PERFORM`
- 현재: `DASHBOARD.GA4_DAY_TRAFFIC` / `GA4_DAY_PERFORM` / `TREND_PRESETS`
- 변경 시: `run_ga_etl.py`의 `SUMMARY_DATASET` + `app.py`의 상수 3곳 수정 → 빌드 → 백필
