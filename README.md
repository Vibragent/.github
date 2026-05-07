<div align="center">

# 🔧 Vibragent

**진동 데이터 기반 예지 보전 AI 에이전트**

제조 설비의 진동 신호를 실시간으로 수집·분석하여
이상 징후를 사전 감지하고, AI 자동 진단 및 챗봇 기반 의사결정을 지원합니다.

</div>

***

## 📌 프로젝트 개요

Vibragent는 산업 현장 회전 기계(모터, 펌프, 팬, 기어박스 등)의 진동 데이터를 AI로 분석하여 **고장 발생 전 이상 상태를 예측**하는 예지 보전(Predictive Maintenance) 플랫폼입니다.

- **진동 신호 수집 → 알람 평가(AlarmEvaluator) → 자동 진단(DiagnosisAutoRunner) → 보고서 생성 → 챗봇 연계**의 전 과정을 하나의 에이전트 파이프라인으로 처리합니다.
- 진단 기준은 **ISO 20816-3** 표준을 참고합니다.

***

## 🗂️ 레포지토리 구성

| 레포지토리 | 언어/프레임워크 | 역할 |
|---|---|---|
| [vib-frontend-vue](https://github.com/Vibragent/vib-frontend-vue) | Vue 3 + Vite 5 | 대시보드, 알람 관리, 진단 보고서, AI Assistant SPA |
| [vib-backend-python](https://github.com/Vibragent/vib-backend-python) | Python 3.11 / FastAPI | AI 진단 Agent, 알람 평가, 자동 진단, 챗봇(AWS Bedrock) API |
| [vib-backend-java](https://github.com/Vibragent/vib-backend-java) | Java 21 / Spring Boot | 설비·사용자 관리, 인증(JWT), RBAC, REST API |
| [vib-infra-iac](https://github.com/Vibragent/vib-infra-iac) | Kubernetes / GitHub Actions | EKS 배포 매니페스트, CI/CD 자동화 |
| [.github](https://github.com/Vibragent/.github) | — | 조직 공통 문서, 이슈/PR 템플릿 |

***

## 🏗️ 시스템 아키텍처

```
[설비 센서]  →  진동 데이터 (가속도·속도·변위)
                        │
              ┌─────────▼──────────┐
              │  vib-backend-python │  FastAPI · Python 3.11
              │  · AlarmEvaluator   │  ← 알람 발생 평가 (ISO 20816-3)
              │  · DiagnosisAuto    │  ← AI 자동 진단 (설비 단위 Lock)
              │  · 월간 보고서 Agent│  ← 월간 분석 자동 생성
              │  · 챗봇 API (LLM)   │  ← AWS Bedrock 연계
              └─────────┬──────────┘
                        │  REST API / WebSocket
              ┌─────────▼──────────┐
              │  vib-backend-java   │  Spring Boot · Java 21
              │  · 설비/사용자 관리 │  ← MariaDB + Redis(JWT)
              │  · RBAC 인증        │  ← Access 30분 / Refresh Token
              └─────────┬──────────┘
                        │  REST + WebSocket (/api/v1/*, /ws/*)
              ┌─────────▼──────────┐
              │  vib-frontend-vue   │  Vue 3 · Vite 5 · ECharts
              │  · Dashboard        │  ← 실시간 설비 상태 카드
              │  · AlarmView        │  ← ACK / 메모 / 진단 연계
              │  · DiagnosisModal   │  ← 보고서 + PDF 다운로드
              │  · AI Assistant     │  ← 챗봇 UI
              └────────────────────┘

              ┌────────────────────┐
              │  vib-infra-iac      │  Kubernetes (EKS) · GitHub Actions CD
              │  · k8s/backend      │  Kustomize 기반 배포
              │  · k8s/frontend     │  Nginx + Ingress
              └────────────────────┘
```

***

## ⚙️ 핵심 기능

- **실시간 진동 모니터링** — WebSocket 기반 설비 상태 실시간 갱신 (`/ws/dashboard`)
- **알람 자동 평가** — `AlarmEvaluator` 백그라운드 task로 이상 알람 자동 발생
- **AI 자동 진단** — `DiagnosisAutoRunner`가 설비 단위 Lock으로 순차 진단 처리
- **주파수 스펙트럼 분석** — FFT 스펙트럼 1X/2X/3X 마커 시각화 (snapshot fallback 지원)
- **진단 보고서 & PDF** — Conclusion/Recommendation/특징 테이블 + html2canvas+jsPDF 출력
- **월간 보고서 Agent** — 월간 분석 결과 자동 생성 및 템플릿 렌더링
- **AI 챗봇** — 보고서 컨텍스트를 AWS Bedrock LLM으로 전달, 자연어 의사결정 지원
- **사용자 인증/권한** — Spring Security JWT + RBAC (`ADMIN` / `USER` / `NONE`)
- **Kubernetes 배포** — EKS + Kustomize + GitHub Actions CD, ECR 이미지 자동 배포

***

## 🔧 기술 스택

| 영역 | 기술 |
|---|---|
| Frontend | Vue 3, Vite 5, Pinia, Vue Router, Axios, ECharts, Element Plus, html2canvas, jsPDF |
| Backend (AI Agent) | Python 3.11, FastAPI, uvicorn, AWS Bedrock |
| Backend (Management) | Java 21, Spring Boot, JPA, MariaDB, Redis |
| Infra | AWS EKS, ECR, RDS (ap-northeast-2), GitHub Actions, Kubernetes, Kustomize, Nginx |
| DB | MariaDB (RDS), Redis (Refresh Token), Vector DB |

***

## 👥 팀 멤버

| 이름 | 역할 | 주요 담당 |
|---|---|---|
| 최재웅 TL | UL (프로젝트 리더) | 프로젝트 일정·범위·리스크 관리, AI Agent 개발, 알람 로직, QA |
| 고대영 TL | Backend Leader | 백엔드 기술 구조 및 방향 설정, 서버 개발, 인프라 설계, CI/CD |
| 이창민 M | Backend AI Agent / Frontend | AI Agent 개발, 알람 관리 UI, 사이드바, GitHub·API Key 관리 |
| 임효준 M | Backend DB / 데이터 | DB 구현, 데이터 수집·전처리, 월간 보고서 Agent, 회의록 관리 |

***

## 🚀 빠른 시작

각 컴포넌트의 상세 실행 방법은 개별 레포지토리의 README를 참조하세요.

```bash
# Frontend — Node.js 18+
cd FE
npm install
npm run dev
# http://localhost:5173

# Backend Python (AI Agent) — Python 3.11+
cd BE
cp .env.example .env
uv sync --frozen
uv run uvicorn app.main:app --reload --port 8000
# Health: http://localhost:8000/health
# Swagger: http://localhost:8000/docs

# Backend Java (Management) — Java 21+
./gradlew.bat bootRun

# Infra — kubectl + EKS 설정 완료 후
kubectl apply -k k8s/backend
kubectl apply -k k8s/frontend
```

***

## ⚠️ 주의사항

- `vib-backend-python`은 **AWS Bedrock 인증 정보** 없이 챗봇/LLM 기능이 동작하지 않습니다.
- 진단 자동화의 **waveform grace 기본값은 12분**이며, 센서 데이터 지연 적재 환경에서 조정이 필요합니다.
- `.env.eks` 파일은 **절대 Git에 커밋하지 않습니다** (`.gitignore` 적용됨).
- AI 진단 결과는 **ISO 20816-3 기준 참고값**이며, 최종 판단은 도메인 전문가 검토가 필요합니다.
- Backend Python + Java 모두 기동되어야 Frontend가 정상 동작합니다.

***

## 📬 문의 및 기여

이슈 및 기능 제안은 각 레포지토리의 **GitHub Issues**를 통해 등록해 주세요.
기여 가이드는 `vib-backend-python/CONTRIBUTING.md`를 참조하세요.

***

<div align="center">
  <sub>© 2026 Vibragent Organization</sub>
</div>
