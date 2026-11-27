# Meoning-Reosin-infra

워크크 서비스의 Azure 인프라 구성과 리소스 기록입니다.
리소스를 만들거나 지울 때마다 이 문서를 업데이트합니다.

> ⚠️ 이 레포는 public입니다. **키, 비밀번호, 연결 문자열, 개인 이메일은 절대 적지 않습니다.**

---

## 전체 구조

~~~
[사용자 휴대폰]
      │
      ▼
[프론트엔드] Next.js ── Vercel
      │  API 호출
      ▼
[백엔드] FastAPI ── ACA (ca-storywalk-api)
      ├─ PostgreSQL + pgvector : 장소, 해설, 벡터, 회원, 일정
      ├─ Blob Storage          : 원본 데이터(raw), 정답 사진(curated)
      ├─ Azure AI Vision       : 사진 → 벡터
      └─ Azure OpenAI          : 챗봇, 추천 문구, 문단 임베딩

[배포] GitHub Actions → Docker 빌드 → ACR → ACA
~~~

---

## 기본 정보

| 항목 | 값 |
|---|---|
| 구독 | (구독 이름) |
| 리소스 그룹 | `rg-meoningreosin-dev` |
| 기본 지역 | **West US 2** (Vision만 Korea Central) |
| 이름 규칙 | `종류약어-meoningreosin` (예: `log-meoningreosin`, `id-meoingreosin-api`) |
| 공통 태그 | `project=meoingreosin`, `env=dev`, `owner=infra` |

---

## Azure 리소스 목록

| # | 리소스 이름 | 종류 | 지역 | 사양 | 용도 | 생성일 | 상태 |
|---|---|---|---|---|---|---|---|
| 0 | `rg-meoningreosin-dev` | 리소스 그룹 | West US 2 | - | 전체 리소스 묶음 | 11/25 | ✅ |
| 1 | `cv-5-project` | Azure AI Vision | Korea Central | S1 | 사진 → 벡터 | 11/27 | ✅ |
| 2 | `log-meoningreosin` | Log Analytics 작업 영역 | West US 2 | 보존 30일, 일일 한도 1 GB | ACA 로그 | 11/27 | ✅ |
| 3 | `id-meoningreosin-api` | 사용자 할당 관리 ID | West US 2 | - | ACA의 접근 권한(신분증) | | ✅ |
| 4 | `acrmeoningreosin` | Container Registry | West US 2 | Basic | Docker 이미지 보관 | | ✅ |
| 5 | `cae-storywalk` | Container Apps 환경 | West US 2 | Consumption | 앱 실행 공간 | | ⬜ |
| 6 | `ca-storywalk-api` | Container App | West US 2 | 0.5 vCPU / 1 GiB, 0~2대 | 백엔드 서버 | | ⬜ |
| 7 | `psql-storywalk` | PostgreSQL Flexible Server | West US 2 | B1ms, 32 GiB, PG16 | 데이터베이스 | | ⬜ |
| 8 | `ststorywalk` | Storage 계정 | West US 2 | StorageV2, LRS, Hot | 원본·사진 파일 | | ⬜ |
| 9 | `aoai-storywalk` | Azure OpenAI | West US 2 | S0 | 챗봇, 추천 문구, 임베딩 | | ⬜ |

상태: ✅ 완료 · 🔄 진행 중 · ⬜ 예정 · 🗑️ 삭제

### 자동 생성된 리소스

| 이름 | 생성 원인 | 비고 |
|---|---|---|
| (예: ME_cae-meoningreosin_...) | ACA 환경 생성 시 자동 | 삭제 금지 |

---

## 리소스 내부 구성

| 상위 리소스 | 항목 | 이름 | 비고 |
|---|---|---|---|
| PostgreSQL | 데이터베이스 | `meoningreosin` | |
| PostgreSQL | 계정 | `meoningreosin_app` | 앱 전용 |
| PostgreSQL | 계정 | (읽기 전용 계정) | 팀원 조회용 |
| PostgreSQL | 확장 | `vector` | pgvector |
| Storage | 컨테이너 | `raw` | 받은 원본 그대로 |
| Storage | 컨테이너 | `curated` | 장소별 정답 사진 (`place_id` 폴더) |
| Azure OpenAI | 모델 배포 | `emb-small` | text-embedding-3-small |
| Azure OpenAI | 모델 배포 | `chat-mini` | GPT mini |

---

## 권한 (역할 할당)

| 대상 리소스 | 역할 | 받는 쪽 | 이유 |
|---|---|---|---|
| `acrstorywalk` | AcrPull | `id-meoningreosin-api` | ACA가 이미지를 가져옴 |
| `acrstorywalk` | AcrPush | `gh-meoningreosin-deploy` | GitHub Actions가 이미지를 올림 |
| 리소스 그룹 | Contributor | `gh-meoningreosin-deploy` | GitHub Actions가 ACA를 업데이트 |
| `ststorywalk` | Storage Blob Data Contributor | AI 담당 | curated 사진 업로드 |
| `ststorywalk` | Storage Blob Data Reader | 적재 스크립트 실행자 | 사진 읽기 |

---

## ⚙️ Container App 설정

### 시크릿 (이름만 기록, 값은 ACA에만 저장)

| 시크릿 이름 | 내용 |
|---|---|
| `DATABASE_URL` | DB 연결 정보 (앱 계정) |
| `AZURE_VISION_KEY` | Vision 키 |
| `AZURE_OPENAI_KEY` | OpenAI 키 |
| `TMAP_KEY` | TMAP API 키 |
| `SEOUL_API_KEY` | 서울 열린데이터 키 |
| `JWT_SECRET` | 로그인 토큰 서명 키 |

### 환경변수

| 이름 | 값 (예시) |
|---|---|
| `AZURE_VISION_ENDPOINT` | `https://(Vision 이름).cognitiveservices.azure.com/` |
| `VISION_MODEL_VERSION` | `2023-04-15` |
| `AZURE_OPENAI_ENDPOINT` | `https://aoai-storywalk.openai.azure.com/` |
| `AZURE_OPENAI_CHAT_DEPLOYMENT` | `chat-mini` |
| `AZURE_OPENAI_EMBED_DEPLOYMENT` | `emb-small` |
| `RECOGNITION_THRESHOLD` | `0.75` |
| `ALLOWED_ORIGINS` | (프론트 주소), `http://localhost:3000` |

---

## 주소

| 구분 | 주소 |
|---|---|
| 백엔드 (ACA) | `https://(ACA 주소)` |
| 상태 확인 | `https://(ACA 주소)/health` |
| 프론트 (Vercel) | `https://(Vercel 주소)` |
| ACR 로그인 서버 | `acrstorywalk.azurecr.io` |

---

## 비용 관리

| 항목 | 값 |
|---|---|
| 예산 | 1,000,000원 (범위: 리소스 그룹) |
| 실제 비용 알림 | 30%, 50%, 80%, 100% |
| 예측 비용 알림 | 100% |
| 수신자 | infra 담당 |

> 예산은 **알림만 보내고 리소스를 멈추지 않습니다.**
> 비용 데이터는 최대 하루 정도 늦게 반영됩니다.

**비용 절약 규칙**
- 평소: PostgreSQL 야간 정지, ACA 최소 0대
- 시연 기간: ACA 최소 1대
- 시연 후: 데이터 백업 → 리소스 그룹 삭제

---

## GitHub ↔ Azure 연결

| 항목 | 값 |
|---|---|
| Entra 앱 | `gh-storywalk-deploy` |
| 방식 | OIDC (비밀번호 없이 로그인) |
| 연결 대상 | `Meoning-Reosin/Meoning-Reosin-backend`, `main` 브랜치 |
| GitHub Secrets | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` |

---

## 작업 기록

| 날짜 | 내용 | 작업자 |
|---|---|---|
| 11/25 | 리소스 그룹 생성 | infra |
| 11/27 | 예산 알림 설정, Vision 생성 | infra |
| 11/27 | Log Analytics 생성 | infra |
