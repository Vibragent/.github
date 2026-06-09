# 2차 구현 변경 정리

## 1. 문서 목적

이 문서는 진단 보고서 개편 작업 중 `2차 구현`에서 실제로 반영한 변경 사항을 정리한 문서다.

2차 구현의 핵심 목표는 다음과 같다.

- 기존 Rule Engine 기반 판정 로직은 유지한다.
- LLM은 판정을 새로 내리지 않고, 이미 나온 판정 결과를 사용자용 보고서 문안으로 정리하는 역할만 수행한다.
- 저장 구조는 DB 컬럼을 크게 바꾸지 않고 `ORIGINAL_REPORT_JSON` 내부의 V2 구조를 확장하는 방식으로 간다.
- 프론트는 V2 구조를 읽어 최종 사용자용 보고서 템플릿 형태로 보여준다.

즉, 2차 구현은 진단 엔진 교체가 아니라 진단 결과의 저장 구조, 서술 구조, 표시 구조를 재정렬하는 작업이다.

---

## 2. 2차 구현의 핵심 원칙

### 2.1 Rule Engine 우선

정량 판정은 기존 Rule Engine이 담당한다.

- 결함 후보 순위
- `fault_type`
- 점수 `score`
- 판정 강도 `decision_strength`
- supporting channel
- evidence detail

이 값들은 2차 구현 이후에도 Rule Engine 결과를 그대로 사용한다.

### 2.2 LLM 역할 제한

LLM은 아래 역할만 수행한다.

- 결론 문장 정리
- 후보별 설명 문장 정리
- 권장 조치 문장 정리
- Trend/Spectrum 요약 문장 정리

LLM이 직접 만들면 안 되는 것은 다음과 같다.

- 새로운 결함 후보 생성
- 점수 변경
- 순위 변경
- 판정 강도 변경
- 운전 허용/중지 같은 최종 운영 판단

### 2.3 DB 스키마 최소 변경

이번 2차 구현에서는 별도 DB 컬럼 추가 없이 진행했다.

- 기존 `TB_DIAGNOSIS_RESULT` 컬럼 유지
- 기존 `TB_DIAGNOSIS_RUN.CHANNEL_RESULTS` 유지
- `ORIGINAL_REPORT_JSON` 내부에 V2 narrative payload 저장

---

## 3. 백엔드 변경 요약

### 3.1 V2 narrative 모델 추가

대상 파일:

- `BE/app/diagnosis/models.py`

추가된 주요 모델:

- `ConclusionSection`
- `CandidateNarrative`
- `ActionPlanSection`
- `DataSummarySection`
- `DiagnosisNarrativeReport`

목적:

- LLM이 출력해야 하는 최소 구조를 명확히 제한
- 기존 V1 `DiagnosisReport`와 V2 narrative 구조를 분리

V2 구조의 기본 형태:

```json
{
  "schema_version": 2,
  "conclusion": {},
  "candidate_narratives": [],
  "action_plan": {},
  "data_summary": {}
}
```

### 3.2 상태 객체에 narrative payload 추가

대상 파일:

- `BE/app/diagnosis/diagnosis_state.py`
- `BE/app/diagnosis/graph.py`

변경 내용:

- 파이프라인 상태에 `narrative_payload` 필드를 추가
- 그래프 초기 상태에도 `narrative_payload: None` 추가

목적:

- LLM이 생성한 V2 narrative를 legacy report와 분리해 보존

### 3.3 LLM 해석 노드 개편

대상 파일:

- `BE/app/diagnosis/nodes/llm_interpret.py`

핵심 변경:

1. 시스템 프롬프트를 V2 narrative 전용으로 정리
2. LLM 출력 스키마를 `DiagnosisNarrativeReport` 기준으로 제한
3. LLM 실패 시 V2 fallback narrative 생성
4. narrative를 기반으로 기존 저장 호환용 legacy report를 함께 생성

현재 흐름은 다음과 같다.

1. Trend/Waveform/Rule Engine 결과 수집
2. LLM이 V2 narrative JSON 생성
3. narrative payload 검증
4. narrative payload를 기반으로 legacy report 재구성
5. 둘 다 상태에 저장

효과:

- 기존 DB 컬럼 호환성 유지
- 프론트 조회 시 V2 구조 활용 가능

### 3.4 검증 노드 개편

대상 파일:

- `BE/app/diagnosis/nodes/verify_report.py`

변경 내용:

- `narrative_payload`를 `DiagnosisNarrativeReport` 기준으로 검증
- 기존 `draft_report`는 legacy 호환 구조로 별도 검증
- 금칙어 검증 유지
  - `%`
  - `confidence`
  - `정량점수`

추가 검증 포인트:

- `candidate_narratives`가 비어 있지 않은지
- 결론/후보 설명에 금칙어가 없는지
- Zone 대비 severity가 비논리적이지 않은지

### 3.5 에러 처리 개편

대상 파일:

- `BE/app/diagnosis/nodes/error_handler.py`

변경 내용:

- 오류 발생 시 legacy report만 만드는 것이 아니라 V2 narrative fallback도 함께 생성

fallback 구성:

- `conclusion`
- `candidate_narratives`
- `action_plan`
- `data_summary`

효과:

- LLM 실패나 데이터 부족 상황에서도 프론트는 동일한 V2 구조를 받을 수 있다.

### 3.6 저장 노드 개편

대상 파일:

- `BE/app/diagnosis/nodes/save_report.py`

변경 내용:

- 저장 시 legacy report와 narrative payload를 병합한 JSON을 `ORIGINAL_REPORT_JSON`에 저장

계속 유지되는 기존 컬럼:

- `SEVERITY`
- `FAULT_CONFIDENCE`
- `FAULT_CANDIDATES`
- `DIAGNOSIS_BASIS`
- `TREND_SUMMARY`
- `WAVEFORM_SUMMARY`
- `HISTORY_COMPARISON`
- `RECOMMENDED_ACTION`
- `DATA_LIMITATIONS`

추가 보존되는 JSON 내부 구조:

- `schema_version`
- `conclusion`
- `candidate_narratives`
- `action_plan`
- `data_summary`

효과:

- DB 스키마를 바꾸지 않고도 V2 구조 저장 가능

### 3.7 조회/조합 로직 개편

대상 파일:

- `BE/app/services/diagnosis_pipeline.py`

중요 변경:

#### 3.7.1 V2 schema 인식

- `ORIGINAL_REPORT_JSON.schema_version` 확인
- 없으면 V1로 간주

#### 3.7.2 candidate_narratives 추출

- `report_json.candidate_narratives`를 별도 추출

#### 3.7.3 Hybrid 병합

최종 `candidate_rankings`는 다음 두 소스를 병합해 생성한다.

Rule Engine 제공:

- `rank`
- `fault_type`
- `score`
- `decision_strength`
- `supporting_channels`
- `evidence_details`

LLM narrative 제공:

- `title`
- `reason`
- `recommended_checks`
- `data_limitations`

병합 방식:

- `fault_type`를 key로 매핑
- narrative가 있으면 설명/확인 항목/한계를 덮어쓴다
- narrative가 없으면 기존 fallback reason 생성 로직 사용

#### 3.7.4 conclusion/action/data_summary 우선 사용

`conclusion`, `action_plan`, `data_summary`가 `ORIGINAL_REPORT_JSON`에 있으면 그 값을 우선 사용한다.

없으면 다음 legacy 필드를 fallback으로 사용한다.

- `DIAGNOSIS_BASIS`
- `RECOMMENDED_ACTION`
- `TREND_SUMMARY`
- `WAVEFORM_SUMMARY`

#### 3.7.5 수정 API 보강

`edit_ai_summary()` 수정 시 다음도 함께 갱신되도록 보강했다.

- `ORIGINAL_REPORT_JSON.diagnosis_basis`
- `ORIGINAL_REPORT_JSON.conclusion.summary`
- `ORIGINAL_REPORT_JSON.conclusion.diagnosis_basis`

즉, 사용자가 결론 텍스트를 수정하면 V2 결론 구조도 같이 맞춰진다.

---

## 4. 프론트 변경 요약

### 4.1 보고서 모달 구조 유지 + 후보 표 분리

대상 파일:

- `FE/src/components/alarm/DiagnosisModal.vue`
- `FE/src/components/alarm/DiagnosisCandidateRankingTable.vue`

프론트 2차 구현 방향은 새 화면을 만드는 것이 아니라 기존 모달 구조를 살리면서 V2 구조를 더 잘 보여주도록 보강하는 것이었다.

유지한 섹션:

- 설비 정보
- 진단 결론
- 권장 조치
- 데이터 요약
- Trend / Spectrum 그래프
- Detail 테이블

변경한 부분:

- 결함 후보 순위 표를 별도 컴포넌트로 분리
- 칼럼 폭을 역할에 맞게 재배치

기존 문제:

- 모든 칼럼 폭이 비슷해서 설명 칸이 지나치게 좁았다

개선 내용:

- `순위` 좁게
- `결함 후보` 중간 폭
- `판정 강도` 좁게
- `점수` 좁게
- `설명` 넓게

추가 표시 항목:

- `reason`
- `recommended_checks`
- `data_limitations`

즉, 단순 순위 표가 아니라 후보별 설명과 확인 항목, 한계까지 보여주는 구조로 바꿨다.

### 4.2 candidate ranking 테이블 레이아웃 개선

대상 파일:

- `FE/src/components/alarm/DiagnosisCandidateRankingTable.vue`

적용 내용:

- `colgroup` 기반 칼럼 폭 지정
- 설명 칸을 가변 폭으로 배치
- 작은 칼럼은 순위/강도/점수 중심으로 축소
- 모바일에서도 비교적 덜 무너지도록 구성

### 4.3 한글 깨짐 복구

대상 파일:

- `FE/src/components/alarm/DiagnosisModal.vue`
- `FE/src/components/report/PlantReportPage.vue`
- `FE/src/views/DashboardView.vue`

복구 내용:

- `진단` 버튼 텍스트 복구
- `1x 진동`, `2x 진동`, `3x 진동`, `BPFO 진동`, `BPFI 진동` 등 주요 한글 라벨 복구
- 월간 보고서 관련 한국어 문안 복구
- 주석 및 섹션 타이틀의 깨진 문자열 정리

주의:

- 이 작업은 phase2 본체는 아니지만, phase2 작업 중 프론트 표시 품질 보완 차원에서 함께 반영했다.

---

## 5. 재작성 버튼의 현재 동작

대상:

- 진단 보고서 모달의 `재작성` 버튼

현재 동작:

1. 프론트에서 `rewriteRun(runId, {})` 호출
2. 백엔드가 현재 저장된 run/detail payload를 다시 읽음
3. LLM에 기존 보고서를 바탕으로 다시 정리하라고 요청
4. 결과를 `DIAGNOSIS_BASIS` 및 `ORIGINAL_REPORT_JSON.conclusion.*`에 반영
5. 프론트가 최신 run 데이터를 다시 조회하면서 갱신

중요한 해석:

- 현재 재작성 기능은 보고서 전체를 새로 생성하는 기능이 아니다.
- 현재 구조에서는 주로 결론/요약 텍스트 재작성에 가깝다.

즉, 사용자가 수정한 최신 결론 텍스트는 재작성 입력에 반영되지만, 아래 항목까지 전부 새로 재계산되지는 않는다.

- 결함 점수
- 결함 순위
- Rule Engine 판정 강도
- 전체 candidate ranking 구조 자체

---

## 6. 2차 구현 전후 비교

2차 구현 전:

- LLM 출력은 사실상 V1 report 중심
- 프론트는 `ai_summary` markdown 의존도가 높음
- 후보 순위 설명은 Rule Engine fallback 조합 의존도가 큼

2차 구현 후:

- LLM은 V2 narrative 구조를 출력
- 저장은 V2 narrative와 legacy report를 함께 보존
- 조회 시 Rule Engine 결과와 narrative를 병합
- 프론트는 `conclusion`, `candidate_rankings`, `action_plan`, `data_summary`를 구조적으로 표시

즉, 판단은 엔진, 서술은 LLM, 최종 구성은 조회 계층이라는 역할 분리가 더 명확해졌다.

---

## 7. DB 컬럼 변경 여부

이번 2차 구현에서는 별도 DB 컬럼 변경이 없다.

이유:

- 기존 운영 데이터와의 호환 유지
- 마이그레이션 비용 최소화
- `ORIGINAL_REPORT_JSON` 확장만으로 충분히 처리 가능

정리:

- `TB_DIAGNOSIS_RESULT` 컬럼 구조 유지
- `TB_DIAGNOSIS_RUN.CHANNEL_RESULTS` 유지
- `ORIGINAL_REPORT_JSON` 내부 구조만 V2로 확장

---

## 8. 검증 결과

### 8.1 백엔드

검증 방식:

- 변경 파일 대상 `py_compile`

결과:

- 통과

의미:

- 최소한의 파일 문법 오류 없이 현재 구조가 연결됨

### 8.2 프론트

검증 방식:

- `npm run build` 시도

결과:

- 로컬 환경에서 `vite` 실행 파일 부재로 빌드 미완료

해석:

- 현재 확인 이슈는 코드 문법보다 설치/환경 상태 문제다

---

## 9. 2차 구현의 의미

이번 변경은 단순히 섹션 몇 개를 추가한 것이 아니다.

실질적으로 다음 구조를 확정한 것이다.

1. Rule Engine이 판정 엔진이다.
2. LLM은 보고서 서술 정리기다.
3. 저장 계층은 legacy 호환성과 narrative 확장을 동시에 가진다.
4. 조회 계층은 최종 사용자용 DTO를 조립한다.
5. 프론트는 markdown 덩어리보다 구조화된 필드를 우선 사용한다.

이 구조는 이후 3차, 4차 확장에도 유리하다.

- narrative edit 범위 확장
- rewrite 범위 확장
- 섹션별 독립 편집
- 보고서 PDF 템플릿 고도화

---

## 10. 후속 과제

이번 2차 구현 이후에도 추가 검토가 필요한 항목은 남아 있다.

### 10.1 rewrite 범위 확장

현재 `재작성`은 결론 텍스트 중심이다.

향후 확장 가능:

- `action_plan` 재작성
- `data_summary` 재작성
- `candidate_narratives` 재작성

단, 이 경우에도 Rule Engine의 점수/순위/강도는 고정 유지되어야 한다.

### 10.2 structured edit API 정교화

현재는 `ai_summary` 중심 수정 흐름이 남아 있다.

향후 개선 가능:

- 결론만 수정
- 권장 조치만 수정
- 후보 설명만 수정

### 10.3 프론트 섹션 컴포넌트 분리

현재는 후보 표만 별도 컴포넌트로 분리되어 있다.

향후 분리 가능:

- `DiagnosisConclusionSection`
- `DiagnosisActionPlanSection`
- `DiagnosisDataSummarySection`

---

## 11. 결론

2차 구현은 기존 진단 로직을 최대한 재사용하면서, 보고서 구조를 사용자 관점으로 재편하는 작업이었다.

핵심 결과는 다음과 같다.

- LLM 출력 권한 축소
- V2 narrative 구조 도입
- legacy 호환 저장 유지
- Rule Engine + LLM hybrid 병합 정립
- 프론트 후보 표 가독성 개선
- 사용자 노출 한글 문구 정리

즉, 기존 진단 체계를 건드리지 않고도 보고서의 품질과 구조적 일관성을 높이는 방향으로 2차 구현이 반영되었다.

---

## 12. Phase2 후속 보완

phase2 적용 후 GitHub CI에서 `tests/test_diagnosis_smoke.py` import 에러와 보고서 검증 회귀가 확인되어, 테스트 파일 수정이 아니라 백엔드 구현을 보완했다.

### 12.1 보완 배경

- `BE/app/diagnosis/nodes/llm_interpret.py` 개편 과정에서 기존 테스트와 내부 호출부가 기대하던 `_format_report_number` 심볼이 제거되었다.
- 또한 `llm_interpret`가 `phase2 narrative JSON`만을 전제로 동작하면서, 기존 `legacy DiagnosisReport JSON` 응답을 그대로 처리하던 호환 경로가 약해졌다.
- `BE/app/diagnosis/nodes/verify_report.py` 역시 `narrative_payload`를 항상 필수로 간주해, 단위 테스트처럼 `draft_report`만 주어지는 경우에도 실패했다.

즉, 문제의 본질은 테스트가 낡았기 때문이 아니라, phase2 적용 과정에서 기존 계약 일부가 구현에서 누락된 것이었다.

### 12.2 왜 테스트를 수정하지 않았는가

- `tests/test_diagnosis_smoke.py`는 단순 예시 테스트가 아니라, 기존 보고서 파이프라인이 유지해야 하는 호환 동작을 검증하고 있었다.
- `_format_report_number` import 실패는 테스트 오기보다 구현 회귀에 가깝다.
- 현재 구조는 phase2 narrative 확장과 legacy 저장/조회 호환이 동시에 필요하므로, 구현이 두 경로를 모두 수용해야 한다.
- 테스트를 바꾸면 CI는 조용해질 수 있으나, 실제 런타임에서 기존 fallback/저장/검증 흐름이 깨진 채 남을 위험이 더 크다.

따라서 이번 보완은 테스트 맞춤 수정이 아니라, `기존 계약 복원 + phase2 확장 유지`를 위한 구현 보정이다.

### 12.3 실제 수정 파일

- `BE/app/diagnosis/nodes/llm_interpret.py`
- `BE/app/diagnosis/nodes/verify_report.py`

### 12.4 llm_interpret 보완 내용

1. `_format_report_number` 호환 함수를 복구했다.
2. LLM 응답이 다음 두 형태를 모두 수용하도록 정리했다.
   - phase2 `DiagnosisNarrativeReport`
   - 기존 `DiagnosisReport`
3. legacy 응답이 들어와도 내부에서 narrative payload를 재구성하도록 보완했다.
4. 기존 fallback 후보 선정, 진단 근거 문장, 권장 조치 생성, analysis summary 생성 로직을 다시 정리했다.
5. `report_markdown`이 비어 있을 경우 decision markdown 또는 legacy markdown을 자동 생성하도록 보완했다.

### 12.5 verify_report 보완 내용

1. `narrative_payload`가 없으면 무조건 실패시키지 않고, 있는 경우에만 phase2 narrative 검증을 수행하도록 변경했다.
2. `draft_report`만 존재해도 legacy 검증은 계속 수행하도록 유지했다.
3. Zone별 severity 검증 메시지를 더 명확하게 정리했다.
   - Zone D는 `위험 또는 경고`
   - Zone C는 `경고 이상`
   - Zone A/B는 `정상 또는 관찰`
4. 금칙어 검증 메시지를 테스트 기대와 맞도록 정리했다.

### 12.6 검증 결과

- `uv run pytest -q tests/test_diagnosis_smoke.py` 통과
- `uv run pytest -q tests/test_diagnosis_verify_report.py` 통과
- `uv run pytest -q` 전체 통과
- 최종 결과: `374 passed`

### 12.7 정리

이 후속 보완은 phase2 설계를 뒤집은 것이 아니다. 오히려 아래 원칙을 더 명확히 지키도록 만든 수정이다.

- Rule Engine이 판정을 만든다.
- LLM은 그 판정을 narrative 구조로 정리한다.
- 저장과 조회는 legacy 호환성과 phase2 narrative 확장을 동시에 유지한다.
- 검증 노드는 narrative 경로와 legacy 경로를 모두 다룰 수 있어야 한다.

---

## 13. 프론트 후속 보완

phase2 적용 이후 실제 보고서 화면을 검토하면서, 프론트 표시 구조를 추가로 다듬었다. 이번 보완은 판정 로직이나 저장 구조를 바꾸는 것이 아니라, 이미 내려오는 phase2 데이터를 더 읽기 좋게 표현하는 데 초점을 둔다.

### 13.1 권장 조치 섹션 표 전환

대상 파일:

- `FE/src/components/alarm/DiagnosisModal.vue`

변경 내용:

- 기존에는 `action_plan.summary`가 있으면 문단형으로 우선 노출했다.
- 이를 `action_plan`의 실제 항목 개수에 따라 행이 늘어나는 표 형태로 변경했다.

렌더링 규칙:

- `summary`가 있으면 `종합` 행으로 표시
- `immediate` 각 항목은 `즉시 확인` 행으로 표시
- `short_term` 각 항목은 `단기 확인` 행으로 표시
- `long_term` 각 항목은 `장기 개선` 행으로 표시

즉, action plan 데이터가 비어 있지 않다면 고정된 문단이 아니라 실제 데이터 개수만큼 행이 생성된다.

fallback 동작:

- `action_plan` 행이 하나도 없으면 기존 `recommended_action` markdown 렌더링을 유지
- 그것도 없으면 evidence 목록 fallback을 유지

의미:

- phase2가 가진 structured action plan 구조를 프론트가 직접 활용하기 시작했다.
- 이후 백엔드에서 `immediate`, `short_term`, `long_term` 채움률이 높아질수록 표 활용도도 함께 올라간다.

### 13.2 설비 정보 / 데이터 요약 표 레이아웃 조정

대상 파일:

- `FE/src/components/alarm/DiagnosisModal.vue`

변경 내용:

- 설비 정보 표에 `colgroup`을 추가해 라벨 열과 값 열 폭을 분리
- 데이터 요약 표에도 `colgroup`을 적용해 우측 설명 열을 더 넓게 사용
- Detail 표 역시 설명 열 비중을 높이는 방향으로 열 폭을 재조정

추가 조정:

- 설비 정보의 `상태등급` 값 셀에 전용 클래스를 부여해 상하 패딩과 줄간격을 줄였다.
- 데이터 요약의 `Spectrum` 행에는 전용 클래스를 부여해 `Trend`보다 낮아 보이던 행 높이를 보정했다.

의미:

- 짧은 값 하나만 들어가는 셀 때문에 행 높이가 과도하게 커 보이던 문제를 완화
- 긴 요약 문장이 불필요하게 세로로 길어지는 현상을 줄임
- 사용자가 한눈에 읽을 수 있는 표 비율을 더 명확히 맞춤

### 13.3 Detail 표시 한계와 현재 정리

대상 파일:

- `FE/src/components/alarm/DiagnosisModal.vue`

정리 내용:

- 현재 프론트는 `selectedTarget.value?.details`만 표시하므로, 한 번에 여러 설비의 detail을 섞어 보여주지는 않는다.
- 다만 백엔드 detail row는 동일한 `feature_name`이 서로 다른 row로 반복될 수 있다.
- 현재 응답에는 각 detail row의 source/channel 구분 필드가 충분하지 않아, 프론트만으로는 “이 row가 motor 기반인지 pump 기반인지”를 명확히 구분할 수 없다.

이번 프론트 보완:

- detail row 반복 렌더링 시 Vue key 충돌이 나지 않도록 key 구성을 안정화

남은 과제:

- detail row에 `source_label`, `measurement_name`, `channel_type` 같은 구분 필드를 함께 내려주면, 프론트에서 중복처럼 보이는 항목을 더 정확히 분리해 보여줄 수 있다.

### 13.4 정리

이번 프론트 후속 보완은 phase2의 본질을 바꾸지 않는다. 다만 다음 점에서 실사용 품질을 높인다.

- structured `action_plan`을 실제 표 형태로 사용
- 설비 정보 / 데이터 요약 / Detail 표의 가독성 개선
- 짧은 값 셀과 긴 설명 셀의 비율 문제 완화
- detail 반복 렌더링 안정성 보완
