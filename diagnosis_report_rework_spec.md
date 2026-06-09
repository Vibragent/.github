# 진단 보고서 개편 명세서

## 1. 문서 목적

본 문서는 현재 진동 진단 파이프라인의 결과 생성/저장/조회/표시 구조를 기준으로, 최종 사용자용 진단 보고서 템플릿 개편을 위해 필요한 전체 수정안을 정리한다.

포함 범위:

- 백엔드 수정 명세서
- 프론트 수정 명세서
- 프롬프트 수정 초안
- 최종 API 응답 JSON 예시

본 개편의 목표는 다음과 같다.

1. 최종 사용자용 보고서를 구조화된 섹션 기반으로 안정적으로 출력한다.
2. 복합 결함 판단 결과를 후보 순위와 근거 중심으로 명확하게 표현한다.
3. 현재 생성되고 있으나 조회 API에서 누락되는 `details`, `spectrum_snapshot`, `diagnostic_decisions`를 실제 화면에서 활용 가능하게 복원한다.
4. `report_markdown` 단일 의존 구조를 줄이고, 백엔드 구조 데이터와 프론트 고정 섹션 렌더링으로 전환한다.

---

## 2. 개편 대상 최종 보고서 템플릿

최종 사용자용 보고서는 아래 구조를 기준으로 한다.

1. 설비 정보
2. 진단 결론
3. 결함 후보 순위 및 근거
4. 권장 조치
5. 데이터 요약
6. Trend 그래프
7. Spectrum 그래프

세부 원칙:

- 설비 정보는 LLM 자유문장이 아니라 백엔드 구조 데이터로 생성한다.
- 진단 결론은 LLM이 작성하되, 근거 데이터는 백엔드가 구조화해 제공한다.
- 결함 후보 순위는 Rule Engine 구조 결과를 우선 사용한다.
- 권장 조치는 종합 조치와 후보별 조치를 구분 가능하도록 설계한다.
- 데이터 요약은 `Trend 요약`, `Spectrum 요약` 중심으로 단순화한다.
- 그래프는 기존 차트 컴포넌트를 재사용한다.

---

## 3. 현재 구조 요약 및 핵심 문제

### 3.1 현재 파이프라인 개요

현재 진단 파이프라인은 다음 순서로 동작한다.

1. 설비/채널 컨텍스트 로드
2. 진단 이력 조회
3. Trend 분석
4. Waveform 분석
5. RAG 컨텍스트 조회
6. LLM 보고서 생성
7. 보고서 검증
8. DB 저장

핵심 파일:

- `BE/app/diagnosis/graph.py`
- `BE/app/diagnosis/nodes/load_context.py`
- `BE/app/diagnosis/nodes/analyze_trend.py`
- `BE/app/diagnosis/nodes/analyze_waveform.py`
- `BE/app/diagnosis/nodes/llm_interpret.py`
- `BE/app/diagnosis/nodes/verify_report.py`
- `BE/app/diagnosis/nodes/save_report.py`
- `BE/app/services/diagnosis_pipeline.py`
- `FE/src/components/alarm/DiagnosisModal.vue`

### 3.2 현재 구조의 문제점

#### 문제 1. 보고서 스키마가 기존 문서형 출력 중심

현재 `DiagnosisReport`는 다음 필드 중심이다.

- `severity`
- `fault_candidates`
- `fault_confidence`
- `diagnosis_basis`
- `trend_summary`
- `waveform_summary`
- `history_comparison`
- `recommended_action`
- `data_limitations`
- `report_markdown`

즉 최종 템플릿의 개별 섹션을 구조적으로 표현하기 어려운 상태다.

#### 문제 2. 프롬프트는 복합 결함 해석 방향은 맞지만 템플릿 구조 강제가 약함

현재 프롬프트는 `diagnostic_decisions`를 독립 블록으로 해석하라고 유도하지만, 최종 보고서의 표 구조/섹션 순서/필드 의미를 강제하지는 않는다.

#### 문제 3. 생성된 세부 근거 데이터가 조회 API에서 누락됨

Waveform 분석 단계에서는 다음 데이터가 생성된다.

- `feature_details`
- `spectrum_snapshot`
- `dominant_peaks`
- `rpm_value`
- `fault_frequency_source`

또 Rule Engine은 다음 구조를 생성한다.

- `ranked_fault_candidates`
- `diagnostic_decisions`
- `evidence_details`
- `actions`
- `supporting_channels`

하지만 현재 pipeline 조회 응답에서는 이 중 일부가 실제 payload에 반영되지 않는다.

대표적으로:

- `details`가 빈 배열로 고정됨
- `spectrum_snapshot`이 `None`으로 고정됨
- `diagnostic_decisions`가 프론트용 구조로 정규화되지 않음

#### 문제 4. 프론트가 구조화 렌더러가 아니라 Markdown 중심

현재 프론트는 사실상 다음 구조다.

- Conclusion: `ai_summary` markdown 렌더링
- Recommendation: 별도 필드 렌더링
- Trend / Spectrum 차트: 별도 렌더링
- Detail 테이블: 데이터가 있을 때만 조건부 렌더링

즉 새 템플릿을 그대로 맞추기 어렵고, `report_markdown`에 과도하게 의존한다.

---

## 4. 전체 개편 방향

개편의 핵심 원칙은 아래와 같다.

1. 구조 데이터는 백엔드가 확정한다.
2. LLM은 구조를 만들기보다 서술을 보강한다.
3. 결함 후보 순위는 Rule Engine 구조를 우선한다.
4. 프론트는 Markdown 통짜 렌더링을 줄이고 구조 섹션 렌더링으로 전환한다.
5. DB는 컬럼을 무리하게 늘리지 않고 JSON 확장을 우선한다.

---

## 5. 백엔드 수정 명세서

## 5.1 수정 목표

백엔드 수정의 목적은 다음 네 가지다.

1. 새 보고서 템플릿에 필요한 구조 데이터를 확정적으로 제공
2. LLM 출력 계약을 템플릿 구조에 맞게 재정의
3. 저장된 원본 분석 데이터를 조회 응답에서 복원
4. 프론트가 추가 가공 없이 섹션 렌더링할 수 있는 API 제공

## 5.2 수정 범위

수정 대상:

- `BE/app/diagnosis/models.py`
- `BE/app/diagnosis/nodes/llm_interpret.py`
- `BE/app/diagnosis/nodes/verify_report.py`
- `BE/app/diagnosis/nodes/error_handler.py`
- `BE/app/diagnosis/nodes/save_report.py`
- `BE/app/services/diagnosis_pipeline.py`
- 필요 시 `BE/app/repositories/diagnosis_pipeline.py`

## 5.3 보고서 스키마 개편안

### 5.3.1 권장 방향

기존 `DiagnosisReport`를 단순 문서형 모델에서 구조화 보고서 모델로 확장한다.

권장 구조:

```json
{
  "severity": "주의",
  "fault_confidence": "MEDIUM",
  "fault_candidates": ["MISALIGNMENT", "UNBALANCE"],
  "equipment_info": {
    "display_name": "Motor Train A",
    "measurement_location": "DE Horizontal",
    "measured_at": "2026-06-09T10:31:20+09:00",
    "status_grade": "주의"
  },
  "conclusion": {
    "title": "정렬 이상 가능성이 우선 검토됩니다.",
    "summary": "1x와 2x 성분이 동시에 높고 ...",
    "diagnosis_basis": "..."
  },
  "candidate_rankings": [
    {
      "rank": 1,
      "fault_type": "MISALIGNMENT",
      "title": "축 정렬 이상",
      "decision_strength": "HIGH",
      "reason": "2x 및 다중 조화성분 ...",
      "supporting_channels": ["DE Horizontal"],
      "evidence_details": []
    }
  ],
  "action_plan": {
    "summary": "정렬 상태 우선 점검 권장",
    "immediate": ["..."],
    "short_term": ["..."],
    "long_term": ["..."]
  },
  "data_summary": {
    "trend": "...",
    "spectrum": "...",
    "envelope": null
  },
  "data_limitations": ["..."],
  "report_markdown": "..."
}
```

### 5.3.2 현실적 적용 방식

초기 적용은 아래처럼 두 단계로 나눈다.

1. DB 컬럼은 유지
2. 새 구조 데이터는 `ORIGINAL_REPORT_JSON` 내부에 저장

즉 기존 컬럼과 호환을 유지하면서도, 새 구조를 JSON에 담아 API 응답에서 활용한다.

## 5.4 Rule Engine 결과 활용 명세

### 5.4.1 핵심 원칙

`결함 후보 순위 및 근거`는 LLM 자유 생성보다 Rule Engine 구조 결과를 우선 사용한다.

원천 데이터:

- `rule_engine_result.ranked_fault_candidates`
- `rule_engine_result.diagnostic_decisions`

### 5.4.2 API용 정규화 구조

`diagnostic_decisions`를 아래 구조로 정규화한다.

```json
{
  "rank": 1,
  "fault_type": "MISALIGNMENT",
  "title": "축 정렬 이상",
  "decision_strength": "HIGH",
  "score": 0.87,
  "reason": "2x 성분과 관련 채널 지지가 확인됩니다.",
  "supporting_channels": ["DE Horizontal", "NDE Horizontal"],
  "evidence_summary": [
    "2x 진폭 증가",
    "다중 조화성분 관찰"
  ],
  "evidence_details": [],
  "recommended_checks": [
    "커플링 정렬 점검",
    "베이스 체결 상태 점검"
  ],
  "data_limitations": []
}
```

### 5.4.3 구현 요구사항

- `diagnostic_decisions`가 존재하면 이를 우선 사용
- 없으면 `ranked_fault_candidates`를 fallback으로 사용
- `fault_candidates`는 여전히 요약용 필드로 유지
- `candidate_rankings`는 최종 보고서용 구조 필드로 추가

## 5.5 details / spectrum_snapshot 복원 명세

### 5.5.1 현재 문제

Waveform 분석 시 `feature_details`와 `spectrum_snapshot`은 생성되지만, 현재 pipeline 조회 응답에서는 누락된다.

### 5.5.2 수정 요구사항

조회 API 정규화 시 `CHANNEL_RESULTS.waveform_results`에서 대표 채널을 선택하고 아래처럼 매핑한다.

- `primary_waveform.feature_details -> details`
- `primary_waveform.spectrum_snapshot -> spectrum_snapshot`
- `primary_waveform.dominant_peaks -> peaks`
- `primary_waveform.rpm_value -> rpm_value`
- `primary_waveform.fault_frequency_source -> fault_frequency_source`

### 5.5.3 대표 채널 선택 규칙

우선순위:

1. run target과 일치하는 `point_id + measurement_id`
2. 상태가 `ANALYZED`인 채널
3. dominant peak 수가 가장 많은 채널
4. 첫 번째 waveform result

## 5.6 설비 정보 생성 명세

### 5.6.1 필수 필드

- `display_name`
- `measurement_location`
- `measured_at`
- `status_grade`

### 5.6.2 생성 원칙

- `display_name`: 현재 `equipment_name`을 기본으로 사용
- 추후 `ICON_GRP -> train` 형태가 필요하면 트리 path 조합 로직 추가
- `measurement_location`: `point_name`, `measurement_nm`, `source_label` 기반 조합
- `measured_at`: 생성 시각과 실제 측정시각을 구분 가능하게 설계
- `status_grade`: `severity` 직접 매핑

### 5.6.3 추가 응답 필드

프론트 전용 필드로 아래를 추가한다.

- `display_equipment_name`
- `display_measurement_location`
- `measured_at`

## 5.7 권장 조치 구조화 명세

현재 `recommended_action`은 단일 문자열이다. 새 구조에서는 다음을 지원해야 한다.

```json
{
  "action_plan": {
    "summary": "정렬 상태 우선 점검 권장",
    "immediate": ["이상 진동 증대 여부 확인", "체결 이상 여부 점검"],
    "short_term": ["정렬 상태 계측", "재측정 수행"],
    "long_term": ["정기 정렬 관리 기준 수립"]
  }
}
```

단, 최종 사용자 오해 방지를 위해 표현은 판단 단정형보다 점검 권고형 문구를 사용한다.

## 5.8 저장 구조 명세

### 5.8.1 유지 사항

- `TB_DIAGNOSIS_RESULT` 기존 컬럼 유지
- `TB_DIAGNOSIS_RUN.CHANNEL_RESULTS`는 raw analysis 저장 유지

### 5.8.2 추가 저장 사항

`ORIGINAL_REPORT_JSON`에 아래 구조 전체 저장:

- `equipment_info`
- `conclusion`
- `candidate_rankings`
- `action_plan`
- `data_summary`
- `report_markdown`

### 5.8.3 이유

- DB migration 비용을 줄일 수 있음
- 보고서 스키마 변경에 유연함
- FE/BE 개발 속도를 높일 수 있음

## 5.9 조회 API 수정 명세

### 5.9.1 응답 구조 개편 방향

현재 `ai_summary` 중심 응답을 다음 구조로 확장한다.

- `equipment_info`
- `conclusion`
- `candidate_rankings`
- `action_plan`
- `data_summary`
- `details`
- `spectrum_snapshot`
- `report_markdown`

### 5.9.2 응답 필드 요구사항

- `ai_summary`는 하위 호환용으로 유지
- 새 프론트는 `structured_report` 계열 필드를 우선 사용
- `report_markdown`은 보조 렌더링/내보내기용으로 유지

## 5.10 검증 로직 수정 명세

`verify_report.py`는 새 구조를 기준으로 재정의한다.

검증 항목:

- `equipment_info` 필수 키 존재 여부
- `conclusion.summary` 최소 길이
- `candidate_rankings` 최대 3개, 최소 1개
- `action_plan.summary` 존재 여부
- `data_summary.trend`, `data_summary.spectrum` 문자열 허용 여부
- zone/severity 일관성 유지

기존의 `report_markdown` 길이만 보는 방식은 보조 검증으로 격하한다.

## 5.11 fallback 보고서 명세

에러 시에도 새 템플릿 구조를 유지해야 한다.

fallback 구조 원칙:

- 설비 정보는 가능한 범위에서 채움
- 결론은 `진단 보류` 또는 `데이터 부족` 명시
- 후보 순위는 `진단 보류` 1건만 제공 가능
- 권장 조치는 재측정/데이터 확인 중심
- 데이터 요약은 이용 가능 데이터 범위만 표시

---

## 6. 프론트 수정 명세서

## 6.1 수정 목표

프론트는 `Markdown 통짜 렌더링` 중심에서 `구조화 보고서 렌더링` 중심으로 전환한다.

## 6.2 수정 대상

- `FE/src/components/alarm/DiagnosisModal.vue`
- 필요 시 하위 테이블/섹션 컴포넌트 분리

## 6.3 레이아웃 구조 개편안

현재:

- Conclusion
- Recommendation
- Trend Graph
- Spectrum Graph
- Detail

개편 후:

1. 설비 정보
2. 진단 결론
3. 결함 후보 순위 및 근거
4. 권장 조치
5. 데이터 요약
6. Trend 그래프
7. Spectrum 그래프
8. 상세 데이터 테이블(선택)

## 6.4 렌더링 원칙

- `equipment_info`: 실제 Vue 테이블 렌더링
- `conclusion`: 카드/문단 렌더링
- `candidate_rankings`: 실제 테이블 렌더링
- `action_plan`: 항목형 렌더링
- `data_summary`: 간단 표 또는 2행 구조 렌더링
- `details`: 실제 테이블 렌더링
- `report_markdown`: 필요 시 보조 표시 또는 PDF 내보내기용

## 6.5 프론트 데이터 매핑 요구사항

### 6.5.1 설비 정보 섹션

필수 입력:

- `equipment_info.display_name`
- `equipment_info.measurement_location`
- `equipment_info.measured_at`
- `equipment_info.status_grade`

### 6.5.2 진단 결론 섹션

필수 입력:

- `conclusion.title`
- `conclusion.summary`
- `conclusion.diagnosis_basis`

### 6.5.3 결함 후보 순위 및 근거 섹션

필수 입력:

- `candidate_rankings[]`

표 예시 컬럼:

- 순위
- 결함 후보
- 판단 강도
- 주요 근거
- 지지 채널
- 권장 점검

### 6.5.4 권장 조치 섹션

필수 입력:

- `action_plan.summary`
- `action_plan.immediate[]`
- `action_plan.short_term[]`
- `action_plan.long_term[]`

### 6.5.5 데이터 요약 섹션

필수 입력:

- `data_summary.trend`
- `data_summary.spectrum`

### 6.5.6 그래프 섹션

기존 차트 재사용:

- Trend chart API 로직 유지
- Spectrum chart API 로직 유지
- `spectrum_snapshot` fallback peak 표시 복원

### 6.5.7 상세 데이터 테이블

필수 입력:

- `details[]`

표 컬럼 예시:

- 항목
- 값
- 단위
- 설명

## 6.6 구현 방식 권장안

### 6.6.1 1차 단계

- 기존 `DiagnosisModal.vue` 내부에 새 섹션 추가
- `ai_summary` markdown 렌더링은 임시 유지
- 새 구조 필드가 있으면 우선 표시

### 6.6.2 2차 단계

- 섹션별 하위 컴포넌트 분리
- `report_markdown` 직접 렌더링 축소
- PDF 출력 구조 개선

## 6.7 편집 기능 재정의

현재 `edit_ai_summary()`는 `diagnosis_basis`만 수정하는 구조와 충돌할 수 있다.

개편 후 권장안:

- `conclusion.summary` 편집
- 필요 시 `action_plan.summary` 편집
- Markdown 통짜 편집은 비권장

---

## 7. 프롬프트 수정 초안

아래는 `llm_interpret.py`의 시스템 프롬프트를 개편하기 위한 초안이다.

```text
당신은 진동 진단 보고서 작성 전문가입니다.

분석 패키지, Rule Engine 결과, 참고 문서를 바탕으로 최종 사용자용 진단 보고서를 JSON 객체 하나로만 작성하세요.

반드시 아래 원칙을 지키세요.

1. 단일 원인으로 억지 수렴하지 마세요.
2. Rule Engine의 diagnostic_decisions를 우선 해석하세요.
3. 결함 후보 순위는 diagnostic_decisions의 priority 순서를 따르세요.
4. candidate_rankings의 각 항목은 수치 근거와 판단 이유를 분리해서 작성하세요.
5. conclusion은 종합 결론만 작성하고, 후보별 상세 근거는 candidate_rankings에 작성하세요.
6. action_plan은 점검 권고형 문구를 사용하세요.
7. 데이터가 부족하면 추정 단정 대신 "진단 보류", "데이터 부족", "추가 확인 필요"를 사용하세요.
8. measured_at, display_name, measurement_location, status_grade는 입력 구조 데이터를 우선 반영하세요.
9. trend 요약과 spectrum 요약은 각각 1~3문장으로 간결하게 작성하세요.
10. report_markdown은 최종 사용자에게 보여줄 한국어 보고서이며 아래 섹션 순서를 반드시 지키세요.

섹션 순서:
1. 설비 정보
2. 진단 결론
3. 결함 후보 순위 및 근거
4. 권장 조치
5. 데이터 요약

반드시 아래 키만 사용하세요.

{
  "severity": "정상|관찰|주의|경고|위험",
  "fault_confidence": "HIGH|MEDIUM|LOW",
  "fault_candidates": ["후보1", "후보2", "후보3"],
  "equipment_info": {
    "display_name": "문자열",
    "measurement_location": "문자열",
    "measured_at": "ISO 날짜 문자열",
    "status_grade": "문자열"
  },
  "conclusion": {
    "title": "한 줄 결론",
    "summary": "종합 결론 설명",
    "diagnosis_basis": "판단 배경 설명"
  },
  "candidate_rankings": [
    {
      "rank": 1,
      "fault_type": "결함 코드 또는 이름",
      "title": "사용자용 결함명",
      "decision_strength": "HIGH|MEDIUM|LOW",
      "reason": "판단 이유",
      "supporting_channels": ["채널1", "채널2"],
      "evidence_summary": ["근거1", "근거2"],
      "recommended_checks": ["점검1", "점검2"],
      "data_limitations": ["제약1"]
    }
  ],
  "action_plan": {
    "summary": "종합 조치 요약",
    "immediate": ["즉시 조치 1"],
    "short_term": ["단기 조치 1"],
    "long_term": ["장기 조치 1"]
  },
  "data_summary": {
    "trend": "Trend 요약",
    "spectrum": "Spectrum 요약",
    "envelope": null
  },
  "data_limitations": ["항목1", "항목2"],
  "report_markdown": "최종 사용자용 Markdown 보고서"
}

응답 본문에는 JSON 외 텍스트를 쓰지 마세요.
report_markdown 바깥에는 Markdown을 쓰지 마세요.
```

### 7.1 프롬프트 운용 검토

- 이 프롬프트는 단독 사용보다 서버 구조 보강과 함께 사용해야 한다.
- `candidate_rankings`의 rank와 fault_type은 가능하면 백엔드에서 먼저 결정하고, LLM은 서술만 보강하는 방식이 더 안정적이다.
- `equipment_info`는 가능하면 프롬프트 입력에 직접 넣고, LLM이 새로 추론하지 않게 해야 한다.

---

## 8. 최종 API 응답 JSON 예시

아래는 프론트가 최종적으로 소비하기 위한 권장 응답 예시다.

```json
{
  "diagnosis_id": 1284,
  "run_id": 991,
  "sys1_id": "04",
  "local_id": 32014,
  "event_id": "EVT-20260609-00031",
  "point_id": 451002,
  "measurement_id": 7,
  "created_at": "2026-06-09T10:35:42+09:00",
  "severity": "주의",
  "severity_score": 3.0,
  "fault_type": "MISALIGNMENT",
  "fault_confidence": "MEDIUM",
  "fault_candidates": [
    "MISALIGNMENT",
    "UNBALANCE",
    "LOOSENESS"
  ],
  "equipment_info": {
    "display_name": "Motor Train A",
    "measurement_location": "DE Horizontal",
    "measured_at": "2026-06-09T10:31:20+09:00",
    "status_grade": "주의"
  },
  "conclusion": {
    "title": "축 정렬 이상 가능성이 우선 검토됩니다.",
    "summary": "1x와 2x 성분이 함께 높고, 관련 채널에서 유사한 진동 증가 패턴이 확인되어 축 정렬 이상 가능성이 우선 검토됩니다. 다만 일부 성분은 불평형 또는 체결 느슨함과 중첩될 수 있어 후보 순위 기준으로 해석이 필요합니다.",
    "diagnosis_basis": "Trend에서 최근 상승 추세가 확인되며, Spectrum에서 1x 및 2x 성분이 동시에 두드러집니다. Rule Engine의 판정 결과에서도 정렬 이상 관련 근거가 가장 우선 순위로 수집되었습니다."
  },
  "candidate_rankings": [
    {
      "rank": 1,
      "fault_type": "MISALIGNMENT",
      "title": "축 정렬 이상",
      "decision_strength": "HIGH",
      "score": 0.87,
      "reason": "2x 성분과 다중 조화성분이 함께 관찰되고, 복수 채널에서 유사한 패턴이 확인됩니다.",
      "supporting_channels": [
        "DE Horizontal",
        "NDE Horizontal"
      ],
      "evidence_summary": [
        "2x 진폭 상승",
        "1x와 2x 동시 우세",
        "복수 채널 유사 패턴"
      ],
      "evidence_details": [
        {
          "type": "harmonic",
          "channel_type": "VEL",
          "label": "2x peak",
          "frequency_hz": 58.8,
          "amplitude": 4.62,
          "unit": "mm/s"
        }
      ],
      "recommended_checks": [
        "커플링 정렬 상태 점검",
        "축심 편차 측정",
        "재측정 후 비교 확인"
      ],
      "data_limitations": []
    },
    {
      "rank": 2,
      "fault_type": "UNBALANCE",
      "title": "불평형",
      "decision_strength": "MEDIUM",
      "score": 0.64,
      "reason": "1x 성분이 높으나 2x 성분 동반으로 인해 단독 불평형으로 단정하기는 어렵습니다.",
      "supporting_channels": [
        "DE Horizontal"
      ],
      "evidence_summary": [
        "1x peak 우세"
      ],
      "evidence_details": [],
      "recommended_checks": [
        "회전체 불평형 가능성 점검"
      ],
      "data_limitations": [
        "단일 원인으로 확정하기 어려움"
      ]
    },
    {
      "rank": 3,
      "fault_type": "LOOSENESS",
      "title": "체결 느슨함",
      "decision_strength": "LOW",
      "score": 0.41,
      "reason": "일부 조화성분 패턴이 있으나 지지 근거가 제한적입니다.",
      "supporting_channels": [
        "DE Horizontal"
      ],
      "evidence_summary": [
        "조화성분 일부 관찰"
      ],
      "evidence_details": [],
      "recommended_checks": [
        "베이스 체결 상태 점검"
      ],
      "data_limitations": [
        "지지 채널 수 제한"
      ]
    }
  ],
  "action_plan": {
    "summary": "정렬 상태를 우선 점검하되, 불평형 및 체결 상태도 함께 확인하는 접근이 적절합니다.",
    "immediate": [
      "이상 진동 급증 여부 재확인",
      "체결 상태 육안 점검"
    ],
    "short_term": [
      "정렬 상태 계측",
      "동일 채널 재측정 및 비교"
    ],
    "long_term": [
      "정렬 관리 기준 정비",
      "반복 이상 패턴 이력 관리"
    ]
  },
  "data_summary": {
    "trend": "최근 14일 및 90일 구간에서 상승 추세가 확인됩니다.",
    "spectrum": "1x와 2x 성분이 함께 높게 나타나며 정렬 이상 가능성을 지지합니다.",
    "envelope": null
  },
  "details": [
    {
      "feature_name": "1x_peak",
      "feature_value": 5.21,
      "feature_unit": "mm/s",
      "description": "회전 1배 성분 진폭"
    },
    {
      "feature_name": "2x_peak",
      "feature_value": 4.62,
      "feature_unit": "mm/s",
      "description": "회전 2배 성분 진폭"
    }
  ],
  "spectrum_snapshot": {
    "zone2_harmonics": {
      "one_x": {
        "actual_freq_hz": 29.4,
        "amplitude": 5.21
      },
      "two_x": {
        "actual_freq_hz": 58.8,
        "amplitude": 4.62
      }
    }
  },
  "data_limitations": [
    "일부 후보는 중첩 가능성이 있습니다."
  ],
  "ai_summary": "하위 호환용 요약 또는 report_markdown 기반 문자열",
  "report_markdown": "## 설비 정보\n...\n## 진단 결론\n...\n## 결함 후보 순위 및 근거\n...\n## 권장 조치\n...\n## 데이터 요약\n..."
}
```

### 8.1 응답 필드 운용 원칙

- 새 프론트는 `equipment_info`, `conclusion`, `candidate_rankings`, `action_plan`, `data_summary`, `details`, `spectrum_snapshot`를 우선 사용한다.
- `ai_summary`와 `report_markdown`은 하위 호환 및 PDF/공유용 보조 필드로 유지한다.

---

## 9. 구현 우선순위 및 단계별 추진안

## 9.1 1차 단계: 데이터 복원

목표:

- `details` 복원
- `spectrum_snapshot` 복원
- `diagnostic_decisions` 응답 노출
- `fault_confidence` 타입 정합성 정리

효과:

- 현재 생성 중인 데이터를 실제 화면에서 활용 가능
- 프론트 Detail/Spectrum fallback 활성화

## 9.2 2차 단계: 보고서 구조 확장

목표:

- `equipment_info`, `conclusion`, `candidate_rankings`, `action_plan`, `data_summary` 추가
- 프롬프트 구조 강화
- fallback 보고서 구조 일치

효과:

- 새 템플릿의 백엔드 기반 확보

## 9.3 3차 단계: 프론트 렌더링 전환

목표:

- Markdown 통짜 렌더링 축소
- 섹션별 고정 UI 렌더링
- 편집/내보내기 UX 정리

효과:

- 최종 사용자 보고서 품질 향상
- 유지보수성 향상

---

## 10. 최종 검토 의견

### 10.1 방향성 평가

이번 개편은 적절하다. 현재 시스템은 진단 데이터를 충분히 생성하고 있으나, 최종 사용자용 보고서 구조로 전달하는 계층이 약하다. 따라서 이번 작업의 중심은 모델 튜닝보다 아래 세 가지여야 한다.

- 출력 계약 재설계
- 조회 API 구조화
- 프론트 렌더링 재구성

### 10.2 반드시 반영해야 할 항목

- `details` / `spectrum_snapshot` 복원
- `diagnostic_decisions` 기반 후보 순위 구조화
- `fault_confidence` 타입 정리
- 새 템플릿 기반 응답 필드 추가

### 10.3 과도하게 하지 말아야 할 항목

- 초기에 DB 컬럼을 과도하게 늘리는 작업
- 모든 것을 `report_markdown` 하나에 몰아넣는 작업
- 후보 순위를 전적으로 LLM 자유 생성에 맡기는 작업

### 10.4 권장 결론

실행 우선순위는 아래가 맞다.

1. 백엔드 조회 응답 복원
2. 새 구조 필드 추가
3. 프롬프트 개편
4. 프론트 섹션 렌더링 전환

이 순서로 가면 리스크를 낮추면서도 실제 사용자 체감 개선을 빠르게 만들 수 있다.
