# 05. 대시보드 Cloud Run 배포

Streamlit 대시보드(`app.py`)를 컨테이너 이미지로 빌드하여 Cloud Run에 배포하는 구성.

## 아키텍처

```
사용자 브라우저
    → Cloud Run (sales-dashboard)
      → app.py (Streamlit, 포트 8080)
        → BigQuery / Vertex AI / 네이버 API
```

- **서비스 URL:** `https://sales-dashboard-397756181081.asia-northeast3.run.app`
- **리전:** asia-northeast3 (서울)
- **실행 서비스 계정:** `ga4-bq-activeusers-bot@ga4-bigquery-431807.iam.gserviceaccount.com`

## 관련 파일

| 파일 | 역할 |
|------|------|
| `Dockerfile` | Streamlit 앱을 컨테이너로 빌드 |
| `.dockerignore` | Docker 빌드 시 제외 (ETL 폴더, docs 등) |
| `.gcloudignore` | Cloud Build 업로드 시 제외 (`.git` 포함 — 없으면 빌드 실패) |
| `cloudbuild.yaml` | GitHub 연동 자동 배포용 (선택) |

## 최초 세팅 (1회)

```bash
export PROJECT_ID="ga4-bigquery-431807"

# 필요한 API 활성화
gcloud services enable \
  cloudbuild.googleapis.com \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  people.googleapis.com

# Artifact Registry 저장소 생성
gcloud artifacts repositories create sales-dashboard \
  --repository-format=docker \
  --location=asia-northeast3
```

## 빌드 & 배포

```bash
export PROJECT_ID="ga4-bigquery-431807"
cd ~/sales-dashboard

# 1. 이미지 빌드
gcloud builds submit \
  --tag asia-northeast3-docker.pkg.dev/$PROJECT_ID/sales-dashboard/app:latest

# 2. 배포 (OAuth 환경변수 포함)
gcloud run deploy sales-dashboard \
  --image=asia-northeast3-docker.pkg.dev/$PROJECT_ID/sales-dashboard/app:latest \
  --region=asia-northeast3 \
  --platform=managed \
  --memory=2Gi \
  --cpu=1 \
  --timeout=300 \
  --session-affinity \
  --min-instances=0 \
  --max-instances=3 \
  --service-account=ga4-bq-activeusers-bot@ga4-bigquery-431807.iam.gserviceaccount.com \
  --update-env-vars=OAUTH_CLIENT_ID='<클라이언트ID>' \
  --update-env-vars=OAUTH_CLIENT_SECRET='<시크릿>' \
  --update-env-vars=CLOUD_RUN_URL='https://sales-dashboard-397756181081.asia-northeast3.run.app'
```

> 코드만 수정하고 재배포할 때는 환경변수가 이미 저장되어 있으므로
> `--update-env-vars` 없이 빌드 + `deploy`만 해도 됩니다.

## 주요 옵션 설명

| 옵션 | 값 | 이유 |
|------|-----|------|
| `--memory` | 2Gi | BQ 데이터 로딩 여유 |
| `--session-affinity` | on | Streamlit WebSocket 안정성 |
| `--min-instances` | 0 | 트래픽 없으면 0으로 (비용 절감) |
| `--max-instances` | 3 | 동시 사용 상한 |
| `--service-account` | activeusers-bot | BQ/Vertex 권한 보유 계정 |

## URL 공개 (1회, run.admin 권한 필요)

Cloud Run은 기본적으로 인증 필수라 브라우저 접속 시 403이 뜬다.
접근 제어는 앱 내부 OAuth로 처리하므로, URL 자체는 공개로 연다.

```bash
gcloud run services add-iam-policy-binding sales-dashboard \
  --region=asia-northeast3 \
  --member=allUsers \
  --role=roles/run.invoker
```

> URL이 공개되어도 앱에서 Google 로그인 + 사업군 권한을 확인하므로
> 등록되지 않은 사용자는 데이터를 볼 수 없다.

## 배포에 필요한 IAM 권한

배포하는 사람 계정에 필요:

| 역할 | 용도 |
|------|------|
| `roles/run.admin` | Cloud Run 배포 · IAM 정책 변경 |
| `roles/iam.serviceAccountUser` | 서비스 계정을 앱에 연결 (actAs) |
| `roles/cloudbuild.builds.editor` | 이미지 빌드 |

인수인계 시 다음 담당자에게 위 3개를 부여하면 됨:
```bash
gcloud projects add-iam-policy-binding ga4-bigquery-431807 \
  --member="user:다음담당자@회사.com" \
  --role="roles/run.admin"
gcloud projects add-iam-policy-binding ga4-bigquery-431807 \
  --member="user:다음담당자@회사.com" \
  --role="roles/iam.serviceAccountUser"
```

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `build ... status 125` + `invalid reference format` | `$PROJECT_ID` 비어있음 | `export PROJECT_ID="ga4-bigquery-431807"` |
| `build ... status 125` (UnicodeDecodeError) | `.git` 등 바이너리 업로드됨 | `.gcloudignore` 생성 |
| `UnicodeDecodeError: 0xff` | 파일이 UTF-16 인코딩 | `iconv -f UTF-16 -t UTF-8` 변환 |
| `Dockerfile required` | 프로젝트 폴더 밖에서 실행 | `cd ~/sales-dashboard` |
| `iam.serviceaccounts.actAs denied` | actAs 권한 없음 | `roles/iam.serviceAccountUser` 부여 |
| 배포 `Retry` 에러 | Cloud Shell 네트워크 일시 오류 | 재실행 |
| `StreamlitSecretNotFoundError` | secrets.toml 없음 | 환경변수 우선 읽도록 코드 처리됨 (재빌드) |

## .gcloudignore 내용

```
.git
.gitignore
__pycache__
*.pyc
.venv
venv
docs/
dash_board/
dash_hana/
*.md
```
