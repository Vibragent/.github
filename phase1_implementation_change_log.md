# 1차 구현 변경사항 정리

## 1. 문서 목적

이 문서는 진단 보고서 개편 작업 중 **1차 구현**에서 실제로 반영한 코드 변경사항을 정리한 문서다.

1차 구현의 목표는 다음과 같았다.

- 기존 진단 엔진은 변경하지 않는다.
- 이미 저장된 진단 결과를 조회 응답에서 다시 살린다.
- 프론트에서 구조화된 보고서 섹션을 우선 표시한다.
- 기존 `ai_summary` 기반 화면과 편집 흐름은 fallback으로 유지한다.

즉 이번 변경은 **진단 로직 재작성**이 아니라 **기존 진단 결과 재사용 + 조회/표시 구조 개선**이다.

---

## 2. 변경 파일

### 백엔드

- `BE/app/services/diagnosis_pipeline.py`

### 프론트

- `FE/src/components/alarm/DiagnosisModal.vue`

### 이번 1차 구현에서 의도적으로 수정하지 않은 파일

- `BE/app/diagnosis/nodes/analyze_trend.py`
- `BE/app/diagnosis/nodes/analyze_waveform.py`
- `BE/app/diagnosis/rule_engine.py`
- `BE/app/diagnosis/nodes/llm_interpret.py`
- `BE/app/diagnosis/nodes/save_report.py`
- `BE/app/diagnosis/nodes/verify_report.py`
- `BE/app/diagnosis/nodes/error_handler.py`

---

## 3. 변경 핵심 요약

이번 변경의 핵심은 아래 네 가지다.

1. `details` 복원
2. `spectrum_snapshot` 복원
3. Rule Engine 결과 기반 `candidate_rankings` 응답 추가
4. 프론트 구조화 섹션 우선 렌더링 추가

---

## 4. 백엔드 변경 상세

## 4.1 변경 대상

- `BE/app/services/diagnosis_pipeline.py`

기존 병목은 `_normalize_result()`에서 이미 계산된 분석 결과를 최종 응답으로 내보내지 않고 잘라내는 부분이었다.

기존 상태:

- `details: []`
- `spectrum_snapshot: None`

즉 waveform 분석 결과가 DB와 `CHANNEL_RESULTS`에는 존재해도 API 응답에서는 버려지고 있었다.

---

## 4.2 `_normalize_run()` 변경

`_normalize_run()`은 기존에 `result`를 `targets: [result]`로만 감싸고 있었다.

이번 변경으로 아래 구조화 필드를 run 상단에도 같이 노출하도록 수정했다.

- `equipment_info`
- `conclusion`
- `candidate_rankings`
- `action_plan`
- `data_summary`

목적:

- run 모드와 result 모드가 동일한 렌더링 경로를 타게 하기 위함
- 프론트에서 `selectedTarget` 또는 상위 payload 둘 다 fallback 가능하게 하기 위함

---

## 4.3 `_normalize_result()` 변경

`_normalize_result()`에 아래 조립 로직을 추가했다.

### 추가된 처리 흐름

1. `CHANNEL_RESULTS`에서 `waveform_results` 추출
2. 대표 waveform 1건 선택
3. 대표 waveform에서 `details`, `spectrum_snapshot`, `peaks`, `rpm_value` 복원
4. `rule_engine_result.diagnostic_decisions` 추출
5. `candidate_rankings` 생성
6. `equipment_info`, `conclusion`, `action_plan`, `data_summary` 생성

---

## 4.4 추가된 helper 함수

이번 1차 구현에서 아래 helper를 추가했다.

- `_extract_waveform_results()`
- `_pick_primary_waveform_result()`
- `_feature_details()`
- `_spectrum_snapshot()`
- `_dominant_peaks()`
- `_waveform_source_label()`
- `_extract_rule_engine_result()`
- `_extract_diagnostic_decisions()`
- `_build_candidate_rankings()`
- `_build_candidate_reason()`
- `_build_equipment_info()`
- `_build_conclusion_section()`
- `_build_action_plan_section()`
- `_build_data_summary_section()`

이렇게 분리한 이유:

- `_normalize_result()` 본문 비대화 방지
- 2차 구현 시 재사용 가능성 확보
- waveform 복원 / rule-engine 병합 / 구조화 섹션 생성을 분리하기 위함

---

## 4.5 waveform 결과 복원 방식

`CHANNEL_RESULTS.waveform_results`에서 분석 결과 배열을 읽고, 대표 waveform 1건을 골라 응답에 매핑하도록 했다.

### 대표 waveform 선택 우선순위

1. `point_id + measurement_id` 완전 일치
2. `point_id` 일치
3. `status == "ANALYZED"`
4. `dominant_peaks` 개수 많은 순

### 복원된 응답 필드

- `details`
- `spectrum_snapshot`
- `peaks`
- `rpm_value`
- `fault_frequency_source`
- `primary_waveform_channel`
- `source_label`

### 의미

기존에 분석은 됐지만 프론트에서 못 보던 아래 데이터가 다시 살아났다.

- feature detail 테이블 데이터
- 차트가 없을 때 spectrum snapshot peak fallback
- 대표 채널 식별 정보

---

## 4.6 Rule Engine 기반 `candidate_rankings` 생성

`CHANNEL_RESULTS.rule_engine_result.diagnostic_decisions`를 읽어 `candidate_rankings`를 생성하도록 했다.

### 그대로 재사용한 값

- `rank`
- `fault_type`
- `score`
- `decision_strength`
- `supporting_channels`
- `evidence_details`
- `actions`
- `data_limitations`

### 보고서용으로 조립한 값

- `reason`
- `evidence_summary`

### 구현 원칙

- Rule Engine의 정량 판단은 그대로 유지
- 1차 구현에서는 LLM narrative에 의존하지 않음
- 설명 문장만 조립형 fallback으로 생성

### `reason` 생성 방식

아래 정보를 조합해 간단한 설명 문장을 만든다.

- evidence
- supporting_channels
- evidence_details 존재 여부

예:

- "`DE Horizontal` 채널에서 `2x harmonic`, `phase mismatch` 근거가 확인되어 정렬 이상 후보로 분류되었습니다."

---

## 4.7 구조화 섹션 응답 추가

기존 문자열 필드와 `ORIGINAL_REPORT_JSON`을 바탕으로 아래 구조화 필드를 응답에 추가했다.

- `equipment_info`
- `conclusion`
- `candidate_rankings`
- `action_plan`
- `data_summary`

### `equipment_info`

- `display_name`
- `measurement_location`
- `measured_at`
- `status_grade`

### `conclusion`

- `title`
- `summary`
- `diagnosis_basis`
- `decision_strength`

### `action_plan`

- `summary`
- `immediate`
- `short_term`
- `long_term`

1차에서는 `summary` 중심으로만 실질 활용하고, 나머지는 빈 배열 fallback 구조로 두었다.

### `data_summary`

- `trend`
- `spectrum`
- `envelope`

1차에서는 `TREND_SUMMARY`, `WAVEFORM_SUMMARY`를 재배열하는 수준으로만 구현했다.

---

## 5. 프론트 변경 상세

## 5.1 변경 대상

- `FE/src/components/alarm/DiagnosisModal.vue`

기존 화면은 사실상 아래 구조였다.

- Conclusion = `ai_summary`
- Recommendation = `recommended_action`
- Trend chart
- Spectrum chart
- Detail table

이번 1차 구현에서는 “구조화 필드가 있으면 먼저 보여주고, 없으면 기존 fallback 사용” 원칙으로 확장했다.

---

## 5.2 추가된 섹션

### 설비 정보

추가 필드:

- 설비명
- 측정 위치
- 측정일
- 상태등급

렌더링 조건:

- `activeEquipmentInfo` 존재 시

### 진단 결론

우선순위:

1. `activeConclusion`
2. 기존 `ai_summary`

즉 구조화 conclusion이 있으면 그 값을 먼저 보여주고, 없으면 기존 markdown 요약을 렌더링한다.

### 결함 후보 순위 및 근거

표 형태로 추가했다.

표시 항목:

- 순위
- 결함 후보
- 판정 강도
- 점수
- 설명

### 권장 조치

우선순위:

1. `activeActionPlan.summary`
2. 기존 recommendation markdown
3. evidences fallback

### 데이터 요약

표시 항목:

- `Trend`
- `Spectrum`

---

## 5.3 추가된 computed

프론트에 아래 computed를 추가했다.

- `activeEquipmentInfo`
- `activeConclusion`
- `hasStructuredConclusion`
- `activeCandidateRankings`
- `activeActionPlan`
- `activeDataSummary`

### 목적

- run/result 모드 공통 렌더링
- 새 구조화 응답 우선 사용
- 기존 응답 fallback 유지

---

## 5.4 기존 detail/snapshot 렌더링 재사용

이번 변경에서 중요한 점은 detail/snapshot 렌더링을 새로 만들지 않았다는 점이다.

기존 재사용된 흐름:

- `activeDetails`
- `resolvedDiagPeaks`
- `snapshotPeakItems`

즉 백엔드에서 값만 복원되면 기존 UI가 그대로 동작하도록 유지했다.

---

## 5.5 normalize 로직 보강

`normalizeDiagnosisPayload()`에 아래 보강을 추가했다.

- `equipment_info`가 객체가 아니면 `null`
- `conclusion`이 객체가 아니면 `null`
- `candidate_rankings`가 배열이 아니면 `[]`
- `action_plan`이 객체가 아니면 `null`
- `data_summary`가 객체가 아니면 `null`

목적:

- 구조화 필드가 없는 구 데이터와의 호환성 유지
- 프론트 렌더링 예외 방지

---

## 6. 검증 결과

## 6.1 백엔드

실행:

- `python -m py_compile BE/app/services/diagnosis_pipeline.py`

결과:

- 통과

의미:

- 최소한 Python 문법 오류는 없음

---

## 6.2 프론트

실행:

- `npm run build`

결과:

- 실패

실패 원인:

- 현재 환경에 `vite`가 설치되어 있지 않아 빌드 명령 자체가 실행되지 않음

즉 이 실패는 이번 코드 수정으로 인한 Vue 문법 오류 확인 실패가 아니라, 로컬 의존성 부재에 따른 실행 불가 상태다.

---

## 7. 이번 1차 구현이 재사용한 것

이번 구현은 기존 진단 엔진을 재사용한다.

### 재사용한 기존 결과

- waveform 분석 결과
  - `feature_details`
  - `spectrum_snapshot`
  - `dominant_peaks`
  - `rpm_value`
  - `fault_frequency_source`

- Rule Engine 결과
  - `diagnostic_decisions`
  - `ranked_fault_candidates`
  - `score`
  - `decision_strength`
  - `supporting_channels`
  - `actions`
  - `evidence_details`

- 기존 보고서 문자열
  - `DIAGNOSIS_BASIS`
  - `TREND_SUMMARY`
  - `WAVEFORM_SUMMARY`
  - `RECOMMENDED_ACTION`
  - `report_markdown`
  - `ai_summary`

### 재사용하지 않고 새로 만든 것

- `candidate_rankings` 응답 구조
- `equipment_info` 응답 구조
- `conclusion` 응답 구조
- `action_plan` 응답 구조
- `data_summary` 응답 구조

단, 이 새 구조 역시 **새 진단 판단을 만든 것**이 아니라 **기존 저장 결과를 보고서용으로 재구성한 것**이다.

---

## 8. 1차 구현의 한계

이번 구현은 의도적으로 아래를 하지 않았다.

- LLM 출력 스키마 개편
- 저장 구조 개편
- `verify_report.py` 개편
- `error_handler.py` fallback 개편
- edit/rewrite API 구조 개편
- run 다채널 전체 전개

즉 아래 한계는 남아 있다.

### 한계 1. conclusion 편집과 구조화 conclusion이 완전히 분리되지 않음

현재 편집 기능은 여전히 `ai_summary` 기반이다.

### 한계 2. `candidate_rankings.reason`은 조립형 fallback 문장임

아직 LLM narrative 기반 설명 구조는 아니다.

### 한계 3. run 모드 다채널 전체 보고 구조는 아님

현재는 대표 target 1개 중심이다.

### 한계 4. 프론트 빌드 검증은 환경 의존성 문제로 미완료

실브라우저 확인이 추가로 필요하다.

---

## 9. 변경의 적절성 검토

이번 1차 구현은 범위상 적절하다.

### 적절한 이유

1. 진단 엔진을 건드리지 않음
2. 이미 저장된 분석 결과를 재사용함
3. 사용자 체감이 큰 `details`, `snapshot`, `candidate_rankings`를 우선 복원함
4. 기존 `ai_summary` fallback을 유지해 호환성을 확보함
5. 2차 구현의 저장 구조/LLM 계약 개편과 충돌하지 않음

### 과도하지 않은 이유

아래를 건드리지 않았기 때문에 2차 구현 범위를 침범하지 않았다.

- `llm_interpret.py`
- `save_report.py`
- `verify_report.py`
- `error_handler.py`
- `models.py`

즉 이번 변경은 **조회/표시 계층 보강**으로 정확히 제한되어 있다.

---

## 10. 다음 권장 검증

실제 화면과 응답 기준으로 아래를 확인하는 것이 좋다.

1. waveform 결과가 있는 샘플에서 `details` 테이블이 실제로 채워지는지
2. spectrum 차트가 없을 때 snapshot peak fallback이 보이는지
3. `candidate_rankings` 순위가 Rule Engine 순위와 같은지
4. `equipment_info.measurement_location`이 기대한 채널명으로 보이는지
5. 구조화 필드가 없는 구 데이터에서 기존 `ai_summary` fallback이 유지되는지
6. run 모드와 result 모드 둘 다 동일하게 구조화 섹션이 보이는지

---

## 11. 최종 요약

이번 1차 구현은 다음을 달성했다.

- 기존 진단 결과 재사용
- waveform detail/snapshot 복원
- Rule Engine 기반 후보 순위 응답 추가
- 구조화 보고서 섹션 우선 렌더링 추가
- 기존 markdown fallback 유지

즉 이번 변경은 “새 판단을 만드는 작업”이 아니라 “이미 있는 판단을 더 잘 보여주는 작업”으로 정리할 수 있다.
