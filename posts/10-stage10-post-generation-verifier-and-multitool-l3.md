# [10부 · Stage 10] LLM이 마지막 문장에서 숫자와 조문을 바꾸는 것까지 잡는다: 다중 L3 함수 체이닝과 사후 답변 역검증기(Post-Verifier)

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 한 질문에 두 가지 계산(미수금 이자 + 상가 보증금)이 섞이면 하나만 계산하거나, 파이썬 계산기가 `9억(900,000,000)`을 정확히 줬는데도 AI가 문장으로 옮겨 적다가 `0` 하나를 빼먹어 `90,000,000원`이라고 오타를 내는 일이 있었습니다.
> - **어떻게 고쳤나요?** 한 번에 여러 계산기를 연쇄 실행하게 만들고, **AI가 답변을 다 쓴 직후 '빨간펜 교정 선생님(사후 역검증기)'이 숫자·쉼표 자릿수·법 조문 번호를 원본과 대조해 틀리면 즉시 다시 쓰게** 만들었습니다.

> **핵심 요약 (Stage 10 — 사전·실행·사후 3중 결정론적 폐루프 완성)**
> - **마주친 현실적 한계 (발전 이유)**: `Stage 6`의 단일 L3 파이썬 연동 이후에도 실무 환경에서는 두 가지 빈틈이 발견되었습니다: **① 복합 사건 질문**(예: *"거래처 미수금 1억 원 90일 지연이자랑 지금 계약하려는 서울 상가 보증금 5천만·월세 300만 원 환산보증금을 같이 봐줘"*)이 들어오면 `topTool` 단 1개만 실행되어 두 번째 도메인 수식이 누락되었고, **② 생성 단계(`Pass 2`)에서 로컬 LLM이 파이썬이 계산해 준 정확한 금액(`900,000,000원` = 9억 원)을 한국어 쉼표 표기로 옮기다가 `0`을 하나 빼먹거나(`90,000,000원`), 임의로 반올림하거나, 원문에 없는 조문(`제999조`)·자산 번호(`#5`)를 덧붙이는 사후 변조**가 발생할 수 있었습니다.
> - **도입한 아키텍처 해법**: **① 직교 도메인 계산기를 최대 3개까지 한 번의 `json_object` 추출로 묶어 실행하는 `Multi-Tool L3 Batch Chaining`(`POST /api/l3-run-batch`)**과 한국어 권역/업종 코드 자동 정규화(`"서울" → "SEOUL"`, `"서비스" → "REALESTATE_PRO_TECH_EDU_MED"`), **② 생성 직후 답변에서 인용 범위·법령 조문·L3 수치·10배 자릿수 오류(`ZERO_DIGIT_SHIFT_10X`)를 전수 대조하는 `verify_generated_answer_integrity`**, **③ 위반 적발 시 위반 내역을 역주입해 1회 자동 재작성하는 폐루프 자기교정(`Closed-Loop Self-Repair`)**을 완성했습니다.
> - **검증된 정량 지표**: 2개 도메인 복합 L3 배치 파이썬 실행 **`4.09 ~ 5.94 ms` (`10/10` 100% 통과)** · 중첩 스키마 단순화로 다중 인자 추출 지연 **`8,888 ms → 3,383 ms` (`2.6배` 단축)** · 적대적 오염 답안 5종 + 정상 답안 5종 사후 검증 정확도 **`10/10 (오탐 0% · 미탐 0%)`** 및 라이브 `Qwen3-14B` 1회 자동 교정 통과 실증.

---

## 1. 실제 로컬 `14B` 테스트에서 포착한 '한국어 억↔쉼표 10배 자릿수 누락(`ZERO_DIGIT_SHIFT_10X`)' 현상

실제 `Qwen3-14B` 라이브 테스트 중 매우 흥미로운 한국어 특유 숫자 오류가 관찰되었습니다. 파이썬 L3 엔진이 서울 상가 환산보증금 기준선을 `900000000`(`9억 원`)으로 정확히 넘겨주었는데도, 한국어(`억` = $10^8$)와 서구식 천 단위 쉼표($10^3, 10^6, 10^9$) 사이를 변환하며 문장을 쓰던 LLM이 **`90,000,000원`(`9천만 원`)**으로 `0` 하나를 빠뜨리는 실수를 범할 수 있다는 점입니다. 사후 역검증기는 단순 반올림 오차(`0.015% ~ 6%`)뿐 아니라 **`aval * 10 == auth_v` 또는 `aval / 10 == auth_v` 관계인 10배 자릿수 이동 오류(`ZERO_DIGIT_SHIFT_10X`)**를 정밀 타격하여 즉시 재작성을 강제합니다.

| 검문 계층 | 검사 대상 | 적발하는 대표적 LLM 오류 유형 | 적발 후 처리 |
| :--- | :--- | :--- | :--- |
| **검문 ① 인용 경계** | `[근거 자산 #k]` | 검색된 근거가 `2건`인데 `[근거 자산 #5]`를 지어내어 인용 | 위반 명세 생성 → 1회 자동 재작성 |
| **검문 ② 법령 조문** | `제N조`, `제N조의M` | 제공된 법령 원문·L3 결과에 없는 `제999조` 환각 생성 | 허용 원문 조문 목록 제시 후 즉시 교정 |
| **검문 ③ 수치 보존** | 금액·이율·일수 앵커 | `1,273,973원`을 임의로 `1,318,562원`으로 왜곡하거나 수치 전면 누락 | L3 확정 원수치 재주입 후 무결성 배지 갱신 |
| **검문 ④ 자릿수 가드** | `억/만` ↔ 쉼표 변환 | `9억 원(900,000,000)`을 `90,000,000원`으로 표기 (`ZERO_DIGIT_SHIFT_10X`) | 정확한 자릿수 명시 후 즉시 재작성 |

---

## 2. 사후 생성 답변 결정론적 역검증기 핵심 구현 (Pure Python)

```python
import re
from typing import Dict, List, Set, Tuple

CITATION_RE = re.compile(r"\[?\s*근거\s*자산\s*#\s*(\d+)\s*\]?")
ARTICLE_RE = re.compile(r"제\s*(\d+)\s*조(?:\s*의\s*(\d+))?")
NUMBER_RE = re.compile(r"(?<![A-Za-z0-9_#])(\d{1,3}(?:,\d{3})+|\d+(?:\.\d+)?)\s*(억\s*원|억|만\s*원|만|원|%)?")


def extract_articles(text: str) -> Set[str]:
    return {
        f"제{int(m)}조" + (f"의{int(s)}" if s else "")
        for m, s in ARTICLE_RE.findall(text or "")
    }


def extract_scaled_numbers(text: str) -> List[Tuple[str, float]]:
    cleaned = ARTICLE_RE.sub(" ", CITATION_RE.sub(" ", text or ""))
    out: List[Tuple[str, float]] = []
    for raw_n, unit in NUMBER_RE.findall(cleaned):
        val = float(raw_n.replace(",", ""))
        u = (unit or "").replace(" ", "")
        if u in ("억", "억원"):
            val *= 100_000_000.0
        elif u in ("만", "만원"):
            val *= 10_000.0
        out.append((f"{raw_n}{u}", val))
    return out


def verify_post_generation_integrity(
    answer_text: str,
    retrieved_doc_texts: List[str],
    l3_authoritative_numbers: Dict[str, float],
) -> Dict[str, object]:
    """생성된 답변의 인용 번호, 법령 조문, L3 확정 수치 및 10배 자릿수 오류를 결정론적으로 검문한다."""
    violations: List[str] = []

    cited_ids = [int(x) for x in CITATION_RE.findall(answer_text)]
    bad_ids = [i for i in cited_ids if i < 1 or i > len(retrieved_doc_texts)]
    if bad_ids:
        violations.append(f"OUT_OF_BOUNDS_CITATION: {bad_ids}")

    allowed_articles = extract_articles("\n".join(retrieved_doc_texts))
    hallucinated_articles = sorted(extract_articles(answer_text) - allowed_articles)
    if allowed_articles and hallucinated_articles:
        violations.append(f"UNGROUNDED_STATUTE_ARTICLE: {hallucinated_articles}")

    ans_nums = extract_scaled_numbers(answer_text)
    for field_name, auth_val in l3_authoritative_numbers.items():
        if any(abs(v - auth_val) / max(abs(auth_val), 1.0) <= 1.5e-4 for _, v in ans_nums):
            continue
        for surface, v in ans_nums:
            is_ten_shift = abs(v * 10.0 - auth_val) <= 1.0 or abs(v / 10.0 - auth_val) <= 1.0
            rel_err = abs(v - auth_val) / max(abs(auth_val), 1.0)
            if is_ten_shift:
                violations.append(f"ZERO_DIGIT_SHIFT_10X on {field_name}: '{surface}' != {auth_val:,.0f}")
            elif 1.5e-4 < rel_err <= 0.06:
                violations.append(f"ROUNDING_TAMPER on {field_name}: '{surface}' != {auth_val:,.0f}")

    return {"passed": len(violations) == 0, "violations": violations}


if __name__ == "__main__":
    docs = ["하도급법 제13조 지연이자 및 상가임대차법 제2조 서울 환산보증금 9억 원"]
    l3_truth = {"delay_interest_krw": 1_273_973.0, "seoul_cap_krw": 900_000_000.0}

    ok_res = verify_post_generation_integrity(
        "[근거 자산 #1] 제13조 지연이자는 1,273,973원이며 제2조 서울 기준은 9억 원입니다.", docs, l3_truth
    )
    assert ok_res["passed"] is True

    bad_res = verify_post_generation_integrity(
        "[근거 자산 #5] 제999조에 따라 지연이자는 1,273,973원이며 기준은 90,000,000원입니다.", docs, l3_truth
    )
    assert bad_res["passed"] is False
    assert len(bad_res["violations"]) == 3
```
