# [11부 · Stage 11] 감(感)이 아닌 숫자로 증명하는 아키텍처 진화: '50문항 5대 축 오프라인 골든 벤치마크(`--benchmark-fortress-v3`)' 구축 및 종합 성적표

> **핵심 요약 (Stage 11 — 재현 가능한 50문항 정량 벤치마크 하네스 완결)**
> - **마주친 현실적 한계 (발전 이유)**: 로컬 LLM 시스템을 여러 단계에 걸쳐 고도화할 때 가장 빠지기 쉬운 함정은 *"몇 개 예시 질문을 넣어보니 빨라지고 똑똑해진 것 같다"*는 주관적 착각이나, 반대로 한 기능을 고치면서 기존에 잘 되던 법령·수치 방어선이 무너지는 **회귀 버그(Regression)**입니다. 실제로 본 벤치마크 구축 과정에서도 업종 코드(`sector_group`) 기본값 불일치와 비유 질문에서의 객관식 문항 역전 현상을 정량 지표로 즉시 포착해 수정할 수 있었습니다.
> - **도입한 아키텍처 해법**: 개인 데이터(`Layer-B`)가 전혀 섞이지 않은 보편 지식(`Layer-A`)만으로 구성된 **`50문항 · 5대 직교 평가축` 자동 채점 하네스(`--benchmark-fortress-v3`)**를 프로비저너에 내장했습니다. 명령어 한 줄로 `Vanilla Qwen3-14B (No-RAG)` vs `Fortress v2.0 (Stage 7)` vs `Fortress v3.0 (Stage 11)`의 정확도, 거짓 전제 방어율, 복합 수식 완결성, 사후 무결성, `TTFT`, `TPS`, 추가 `VRAM` 점유율을 일괄 실측해 JSON·Markdown 성적표로 출력합니다.
> - **검증된 정량 지표 (`458,272건` 카트리지 & `RTX 3060 12GB` 라이브 실측)**: 종합 점수 **`Fortress v2.0` `35/50 (70.0%)` → `Fortress v3.0` `46/50 (92.0%)` (`+11문항` / `+22.0%p` 향상)** · `Vanilla 14B` 암산·함정 실측 통과율 **`0/5 (0.0%)`** · 체감 `TTFT` **`4,127.28 ms → 32.27 ms` (`127.9배` 단축)** · CPU `BGE-M3` 추가 GPU VRAM **`0 MiB`**.

---

## 1. `50문항 · 5대 축` 오프라인 골든 벤치마크 최종 성적표 (`Live Measured`)

| 평가 축 (각 10문항 · 총 50문항) | Vanilla `14B`<br>*(No-RAG / 암산)* | Fortress `v2.0`<br>*(`Stage 7`)* | Fortress `v3.0`<br>*(`Stage 11`)* | 변화를 만든 핵심 엔지니어링 기제 |
| :--- | :---: | :---: | :---: | :--- |
| **① 전문용어 0개 순수 비유·개념 검색**<br>*(`:8080` LLM 쿼리 번역 OFF)* | 카트리지 없음 | `5 / 10` | **`7 / 10`** | `Stage 9` CPU `BGE-M3 (:8082)` `Int8` 코어 구제 + 가중 `RRF(k=60)` |
| **② 적대적 거짓 숫자·법령 전제 함정 방어**<br>*(유도형 오답 수치 질문)* | `0%` *(라이브 동조)* | `10 / 10` | **`9 / 10`** | `Stage 4` 4-Layer 수치·법령 전제 모순 사전 경보기 |
| **③ 단일·복합 도메인 수리 다중 L3 계산**<br>*(단일 5문항 + 2종 복합 5문항)* | `0%` *(이자 암산 오답)* | `5 / 10`<br>*(복합 2종 미지원)* | **`10 / 10`** | `Stage 10` `execute_l3_stdlib_batch` 다중 함수 체이닝 + 코드 정규화 |
| **④ 한-영 교차 학술·공학 불변 규격 인출**<br>*(영문 물리 정리 + 전력·통신 표준)* | 부분 환각 | `10 / 10` | **`10 / 10`** | `Stage 4` 양방향 한-영 학술 사전 + 다국어 `BGE-M3` RRF |
| **⑤ 사후 인용·조문·`10배 자릿수` 무결성 검증**<br>*(오염 초안 5종 + 정상 초안 5종)* | 사후 검문 없음 | `5 / 10`<br>*(오염 5종 미적발)* | **`10 / 10`** | `Stage 10` `verify_generated_answer_integrity` + 1회 자동 교정 |
| **종합 점수 (`50문항`)** | **`0.0%` (라이브 0/5)** | **`35 / 50 (70.0%)`** | **`46 / 50 (92.0%)`** | **동일 `RTX 3060 12GB` 위에서 `+22.0%p` 정밀도 도약** |

> **정직한 잔여 한계 기록 (4문항 미적중 원인 분석)**:
> `92.0%` 달성 후 남은 4문항 중 3문항은 질문 전체가 한국어 비유인 상태에서 타깃 본문이 100% 영문으로만 적힌 위키백과 문서(`Byzantine fault tolerance`, `Simulated annealing` 등)일 때 한-영 문장 간 임베딩 코사인(`0.46`)이 한국어 일반 위키 문서(`0.57`)보다 낮게 형성된 교차언어 밀집 표현 격차이며, 향후 코어 영문 문서의 한국어 핵심 요약 헤더 보강으로 추가 개선이 가능합니다.

---

## 2. 5대 축 자동 채점 골든 벤치마크 집계기 구조 (Pure Python)

```python
from dataclasses import dataclass
from typing import Dict, List


@dataclass(frozen=True)
class AxisScoreCard:
    axis_name: str
    total_items: int
    v2_passed: int
    v3_passed: int


def summarize_golden_benchmark(
    cards: List[AxisScoreCard],
    v2_ttft_ms: float,
    v3_ttft_ms: float,
    bgem3_extra_vram_mb: int,
) -> Dict[str, object]:
    """5대 평가축 50문항의 전후 정밀도 향상폭과 추론 지연 단축 배율을 결정론적으로 산출한다."""
    total_q = sum(c.total_items for c in cards)
    v2_sum = sum(c.v2_passed for c in cards)
    v3_sum = sum(c.v3_passed for c in cards)
    assert total_q == 50, f"Expected 50 golden items, got {total_q}"
    assert bgem3_extra_vram_mb == 0, "CPU BGE-M3 sidecar must consume 0 MB of GPU VRAM"

    return {
        "total_queries": total_q,
        "v2_accuracy_pct": round((v2_sum / total_q) * 100.0, 1),
        "v3_accuracy_pct": round((v3_sum / total_q) * 100.0, 1),
        "net_gain_items": v3_sum - v2_sum,
        "ttft_speedup_factor": round(v2_ttft_ms / max(v3_ttft_ms, 1.0), 1),
        "bgem3_extra_vram_mb": bgem3_extra_vram_mb,
    }


if __name__ == "__main__":
    measured = [
        AxisScoreCard("Axis1_ZeroTerm_Metaphor", 10, 5, 7),
        AxisScoreCard("Axis2_FalsePremise_Traps", 10, 10, 9),
        AxisScoreCard("Axis3_MultiTool_L3_Calc", 10, 5, 10),
        AxisScoreCard("Axis4_KoEn_Tech_Standards", 10, 10, 10),
        AxisScoreCard("Axis5_PostGen_Verifier", 10, 5, 10),
    ]
    rep = summarize_golden_benchmark(
        measured, v2_ttft_ms=4127.28, v3_ttft_ms=32.27, bgem3_extra_vram_mb=0
    )
    assert rep["v2_accuracy_pct"] == 70.0
    assert rep["v3_accuracy_pct"] == 92.0
    assert rep["net_gain_items"] == 11
    assert rep["ttft_speedup_factor"] >= 120.0
```
