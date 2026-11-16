# [6부 · Stage 6] 프런티어 추론 격차 돌파: 'Adaptive Thinking' 제어와 '139종 L3 순수 파이썬 기호 외골격(2-Pass 자율 계산)' 연동

> **핵심 요약 (Stage 6 — 신경-기호 추론 외골격)**
> - **마주친 현실적 한계 (발전 이유)**: 로컬에 올라가는 `14B` 모델과 초대형 클라우드 프런티어 모델 사이의 가장 큰 격차는 **'다단계 수식·세법·계약 조건 분기 추론'**입니다. 또한 `<think>` 추론 모드를 항상 켜두면 단순 백과사전 질문에도 수십 초씩 사고 토큰을 낭비하고, 항상 꺼두면 복합 조건 문제에서 정답률이 하락했습니다.
> - **도입한 아키텍처 해법**: 대시보드에서 질문 난이도를 자동 판별해 추론을 점화하는 **`Adaptive Thinking` 제어기(`512 ~ 4,096 tok` 예산 조절)**를 도입하고, 마스터 금고 자산들에서 자동 컴파일한 **`139종` 순수 파이썬 검증 함수(`vault_stdlib.py`, `:8081/api/l3-tools` & `/api/l3-run`)**를 연결했습니다. LLM은 **1차 패스에서 자연어 질문 속 변수 JSON만 추출(Autoformalization)**하고, **실제 산술·법령 경계 계산은 파이썬 엔진이 오차 `0.0%`로 수행**한 뒤, **2차 패스에서 해설만 합성**합니다.
> - **검증된 정량 지표**: 내장 결정론적 검증 함수 **`29종 → 139종`** 확장 · `@dataclass` 스키마 자동 타입 보정(문자열 숫자·불리언 강제 변환) 적용 · 다단계 법령/재무 산술 오차 **`0.0%`**.

---

## 1. 2-Pass 신경-기호(Neuro-Symbolic) 자율 정형화 파이프라인

```text
[사용자 질문] "납품 완료 통지 후 15일이 지났는데 원청이 서면 이의제기 없이 검수를 미루고 수령일로부터 90일째 대금 1억 원을 안 줍니다. 지연이자가 얼마인가요?"
     |
     v
[Pass 1 — Autoformalization] 로컬 Qwen3-14B가 매칭된 L3 함수(evaluate_subcontract_claim_and_defenses) 스키마에 맞춰 변수만 JSON 추출
     -> {"contract_amount_krw": 100000000, "days_since_delivery_notice": 15, "written_rejection_within_10_days": false, "days_since_receipt": 90}
     |
     v
[Deterministic Execution] :8081 브리지가 vault_stdlib.py 내 순수 파이썬 함수 실행 (@dataclass 기본값 병합 & 타입 자동 보정)
     -> 간주검수 성립(하도급법 제9조 10일 경과) + 60일 초과분(30일) 연 15.5% 법정 지연이자 1,273,973원 확정 산출 (오차 0.0%)
     |
     v
[Pass 2 — Grounded Synthesis] 확정된 파이썬 결과 블록 + FTS5 법령 원문을 결합해 실무 대응 답변 합성 (임의 재계산 금지)
```

---

## 2. `@dataclass` 스키마 자동 보정 기반 L3 기호 외골격 실행기 (Pure Python)

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
    delay_interest_krw = round(spec.contract_amount_krw * 0.155 * (overdue_days / 365.0))
    return {
        "deemed_inspection_passed": deemed_accepted,
        "overdue_days_beyond_60": overdue_days,
        "statutory_delay_interest_krw": delay_interest_krw,
    }


def coerce_llm_json_to_dataclass(dc_cls: Any, raw_args: Dict[str, Any]) -> Any:
    """로컬 LLM이 문자열 숫자나 느슨한 불리언을 추출하더라도 @dataclass 정적 타입으로 안전하게 보정한다."""
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
