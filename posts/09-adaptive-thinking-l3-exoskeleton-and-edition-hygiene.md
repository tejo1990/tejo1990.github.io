# [9부] 로컬 LLM의 추론 한계를 넘는 법: 'Adaptive Thinking' 제어, '139종 L3 파이썬 기호 외골격(2-Pass 자율 계산)', 그리고 배포 에디션 위생 분리

> **핵심 요약 (Executive Summary)**
> - **발전시켜 나간 이유 1 (프런티어 추론 모델과의 격차)**: 로컬에 올라가는 `14B` 모델은 거대 클라우드 프런티어 모델 대비 깊은 다단계 산술·수리 추론에서 본질적인 체급 한계가 있습니다. 또한 추론(`Thinking`) 모드를 항상 켜두면 단순 정의 질문에도 수십 초씩 `<think>` 토큰을 소모하고, 항상 꺼두면 복합 조건 분기 문제에서 정답률이 떨어집니다.
> - **발전시켜 나간 이유 2 (타 PC 배포 시 개인 데이터 혼입 문제)**: 내 로컬 요새에는 내 개인 문서·의사결정 기록(`Layer-B`)과 누구나 쓸 수 있는 보편 법령·공학·재무 지식(`Layer-A`)이 함께 들어 있습니다. 이를 다른 PC나 고객사 환경에 세팅해 줄 때 개인 데이터가 단 1건이라도 섞여 들어가면 안 됩니다.
> - **최종 진화 해법**: **① 질문 난이도에 따라 추론을 자동 점화하는 `Adaptive Thinking` + 사고 예산 제어**, **② 마스터 금고에서 자동 컴파일된 `139종` 순수 파이썬 함수(`vault_stdlib.py`)를 LLM이 직접 호출해 계산 오차 `0.0%`를 달성하는 `2-Pass 신경-기호(Neuro-Symbolic) 외골격`**, **③ 빌드 시점에 개인 레이어를 물리적으로 100% 도려내는 `Edition Isolation`**으로 아키텍처를 완결했습니다.

---

## 1. 왜 로컬 LLM에게 직접 계산시키지 않고 '2-Pass 자율 정형화(Autoformalization)'를 쓰는가?

프런티어 모델조차 5단계 이상의 세금·이자·상각·공제 조건이 얽히면 암산 실수를 합니다. 본 시스템은 로컬 `Qwen3-14B`가 가장 잘하는 **'자연어에서 구조화된 변수(JSON) 추출하기'**와 파이썬이 완벽하게 해내는 **'결정론적 수식 계산'**을 `:8081/api/l3-tools` 및 `:8081/api/l3-run` 브리지로 결합했습니다.

```text
[사용자 질문] "납품 완료 통지 후 15일이 지났는데 원청이 서면 이의제기 없이 검수를 미루고 수령일로부터 90일째 대금 1억 원을 안 줍니다. 지연이자가 얼마인가요?"
     |
     v
[Pass 1 — Autoformalization] 로컬 Qwen3-14B가 매칭된 L3 함수(evaluate_subcontract_claim_and_defenses)의 JSON 스키마에 맞춰 변수만 추출
     -> {"contract_amount_krw": 100000000, "days_since_delivery_notice": 15, "written_rejection_within_10_days": false, "days_since_receipt": 90}
     |
     v
[Deterministic Execution] :8081 브리지가 vault_stdlib.py 내 순수 파이썬 함수를 실행 (@dataclass 타입 자동 보정 & 화이트리스트 검증)
     -> 간주검수 성립(제9조 10일 경과) + 60일 초과분(30일) 연 15.5% 법정 지연이자 정확히 산출 (오차 0.0%)
     |
     v
[Pass 2 — Grounded Synthesis] 산출된 확정 수치와 법령 조문을 근거로 읽기 쉬운 최종 법률·실무 답변 합성
```

또한 대시보드 상단에 **`Thinking 모드 (Adaptive / ON / OFF)` 및 사고 토큰 예산(`512 ~ 4,096 tok`) 제어기**를 두어, 일반 백과사전·단순 조문 조회는 `Thinking OFF`로 즉답하고 수식·비교·시뮬레이션 질의에서만 깊은 추론을 활성화합니다.

---

## 2. `@dataclass` 스키마 자동 보정 기반 L3 기호 외골격 실행기 (Pure Python)

로컬 LLM이 숫자 문자열(`"100000000"`)이나 일부 누락된 필드를 포함한 JSON을 내놓더라도 에러 없이 안전하게 실행되도록, 화이트리스트에 등록된 `139종` 검증 함수의 `@dataclass` 기본값과 타입 힌트를 읽어 자동으로 타입을 강제 변환(Coercion)합니다.

```python
import dataclasses
from dataclasses import dataclass
from typing import Any, Dict, get_type_hints


@dataclass(frozen=True)
class SubcontractQuickSpec:
    contract_amount_krw: int = 100_000_000
    days_since_delivery_notice: int = 10
    written_rejection_within_10_days: bool = False
    days_since_receipt: int = 60


def evaluate_subcontract_delay_interest(spec: SubcontractQuickSpec) -> Dict[str, object]:
    deemed_accepted = (spec.days_since_delivery_notice > 10) and (not spec.written_rejection_within_10_days)
    overdue_days = max(0, spec.days_since_receipt - 60)
    # 하도급법 제13조 공정위 고시 연 15.5% 지연이자
    delay_interest_krw = round(spec.contract_amount_krw * 0.155 * (overdue_days / 365.0))
    return {
        "deemed_inspection_passed": deemed_accepted,
        "overdue_days_beyond_60": overdue_days,
        "statutory_delay_interest_krw": delay_interest_krw,
    }


def coerce_llm_json_to_dataclass(dc_cls: Any, raw_args: Dict[str, Any]) -> Any:
    """LLM이 추출한 느슨한 JSON 인자를 @dataclass 정적 타입과 기본값에 맞춰 안전하게 변환한다."""
    hints = get_type_hints(dc_cls)
    clean_kwargs: Dict[str, Any] = {}
    for f in dataclasses.fields(dc_cls):
        if f.name not in raw_args:
            continue
        val = raw_args[f.name]
        target_t = hints.get(f.name, type(val))
        if target_t is bool and isinstance(val, str):
            clean_kwargs[f.name] = val.strip().lower() in ("true", "1", "yes")
        elif target_t is int:
            clean_kwargs[f.name] = int(float(val))
        else:
            clean_kwargs[f.name] = val
    return dc_cls(**clean_kwargs)


if __name__ == "__main__":
    # LLM이 문자열 숫자를 섞어 추출하더라도 결정론적으로 정확히 실행됨
    llm_extracted = {
        "contract_amount_krw": "100000000",
        "days_since_delivery_notice": "15",
        "written_rejection_within_10_days": "false",
        "days_since_receipt": 90,
    }
    spec_obj = coerce_llm_json_to_dataclass(SubcontractQuickSpec, llm_extracted)
    out = evaluate_subcontract_delay_interest(spec_obj)
    assert out["deemed_inspection_passed"] is True
    assert out["overdue_days_beyond_60"] == 30
    assert out["statutory_delay_interest_krw"] == 1_273_973
```

---

## 3. 배포 에디션 위생 분리 (`Edition Isolation`: `my_own` vs. `client`)

시스템이 고도화될수록 **"내 개인용 두뇌(`my_own`)"**와 **"외부 납품·이식용 패키지(`personal_client` / `domain_client`)"**의 경계가 명확해야 합니다. 본 프로비저너는 빌드 옵션(`--edition`) 하나로 다음 3단계 위생 분리를 자동 수행합니다:

1. **카트리지 & 대시보드 물리적 정화**: 클라이언트 에디션 빌드 시 `asset_layer: LAYER_B_PERSONAL`로 태깅된 개인 문서와 개인 전용 캐시 시드를 SQLite DB 및 대시보드 내장 JSON에서 **물리적으로 `100%` 제외(`EXCLUDED_LAYER_B` 검증)**하고, 누구나 보편적으로 쓰는 **순수 `Layer-A` 마스터 코어(법령·재무·공학·SW 아키텍처)**만 남깁니다.
2. **L3 계산기 화이트리스트 연동 필터링**: `:8081/api/l3-tools` 역시 활성 에디션에 맞춰 보편 검증기만 노출합니다.
3. **25종 표준 입력 템플릿 자동 생성**: 빈손으로 시작하는 사용자를 위해 근로·임대차·보험·DSCR·하도급·BOM 단가표 등 **빈칸만 채워 `custom_docs/`에 넣으면 즉시 색인되는 25종 표준 마크다운 템플릿**을 자동 생성합니다.
