# [5부 · Stage 5] "두꺼비집이 내려가요"를 '배선용차단기 단락전류'로: 3-Tier 구어체 쿼리 번역기와 타이핑 디바운스(`?neural=0`) 큐 병목 해결기

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 일반인은 *"두꺼비집이 내려가요"*라고 묻는데 도서관 책에는 `'배선용차단기 단락전류'`라고 적혀 있어 못 찾았고, 반대로 글자를 칠 때마다 AI 통역사를 불렀더니 미완성 오타 40개가 줄을 서서 `90초` 동안 먹통이 됐습니다.
> - **어떻게 고쳤나요?** 자주 쓰는 일상 말투 사전(`0.00005초`)을 먼저 두고, **타이핑하는 도중에는 AI 두뇌를 절대 깨우지 않다가 엔터를 눌렀을 때만 딱 1번 통역**하게 길을 분리했습니다.

> **핵심 요약 (Stage 5 — 구어체 시맨틱 브리지 & 단일 GPU 큐 제어)**
> - **마주친 현실적 한계 (발전 이유)**: 45.8만 건의 전문 카트리지를 갖추더라도 일반인은 *"두꺼비집이 자꾸 내려가"*, *"타이레놀 먹고 맥주 마셔도 돼?"*처럼 일상 구어체로 질문해 한자어/학술어 색인을 비껴갔습니다. 그런데 이를 해결하려고 타이핑 디바운스(`90ms`)마다 로컬 LLM 번역기를 호출했더니, **단일 슬롯(`N_PARALLEL=1`) GPU 서버에 타이핑 도중의 미완성 글자 조각 40여 개가 쌓여 본 답변이 `89.70초`까지 지연되고 오타 접두어가 캐시를 오염시키는 실제 장애**가 발생했습니다.
> - **도입한 아키텍처 해법**: **① 40선 상황별 구어체 시드 앵커(`<0.05ms`) → ② `mtime` 자동 핫리로드 영구 캐시(프리웜 우선순위 고정 및 미완성 한글 자모 기록 차단) → ③ 실행 시에만 단 1회 호출되는 로컬 LLM 번역기** 3계층 게이트를 구축하고, 타이핑 중(`250ms` + `AbortController`)에는 `?neural=0` 플래그로 GPU 호출을 **`0회`**로 물리 격리했습니다.
> - **검증된 정량 지표**: 타이핑 중 `:8080` LLM 호출 **`0회`** · 4대 도메인 구어체 검색 적중 **`10/10`** · 실측 질의응답 소요 시간 **`89.70초 → 22.66초`**(`~4배` 단축).

---

## 1. 일상 구어체 → 표준 검색어 변환 예시 (40선 프리웜 시드 발췌)

| 도메인 | 사용자의 일상 구어체 질문 | `<0.05 ms` 만에 변환되는 표준 FTS5 키워드 |
| :--- | :--- | :--- |
| **법령 (`L01`)** | *"일하다가 다쳐서 병원 입원했는데 치료비랑 쉬는 동안 월급 얼마나 줘?"* | `근로기준법 업무상 부상 요양보상 휴업보상 평균임금 100분의 60 제78조 제79조` |
| **의학 (`M01`)** | *"어제 술 마시고 머리가 깨질 것 같은데 타이레놀 먹어도 괜찮아?"* | `아세트아미노펜 알코올 음주 병용금기 간독성 NAPQI 글루타치온 급성간부전` |
| **공학 (`E01`)** | *"공장에서 기계 돌리니까 두꺼비집이 퍽 소리 나면서 내려가는데 쇼트난 걸까?"* | `두꺼비집 단락전류 과전류 접지 배선용차단기 누전차단기 전기설비` |
| **전산 (`P01`)** | *"파이썬으로 웹페이지 긁어오는데 상대 서버가 멈추면 내 프로그램도 영원히 멈춰"* | `웹크롤링 Python timeout 소켓 블로킹 asyncio 비동기 교착상태` |

---

## 2. 프리웜 우선순위 보장 및 미완성 한글 자모 오염 방지 캐시 엔진 (Pure Python)

```python
import re
from typing import Dict, List, Optional, Tuple

INCOMPLETE_HANGUL_JAMO_RE = re.compile(r"[ㄱ-ㅎㅏ-ㅣ]$")


def is_cacheable_complete_query(query_text: str) -> bool:
    """짧은 글자 조각(<3자)이나 조합 중인 한글 자모로 끝나는 타이핑 문자열의 캐시 저장을 차단한다."""
    q = query_text.strip()
    if len(q) < 3:
        return False
    if INCOMPLETE_HANGUL_JAMO_RE.search(q):
        return False
    return True


def lookup_priority_rewrite_cache(
    query_text: str,
    cache_store: Dict[str, Dict[str, object]],
) -> Optional[Dict[str, object]]:
    """
    1순위: 큐레이션된 프리웜 시드 완전 일치
    2순위: 큐레이션된 프리웜 상황 앵커 쌍(Anchor-Pair) 포함 일치 (오래된 임시 캐시보다 우선)
    3순위: 일반 런타임 번역 캐시 완전 일치
    """
    q_clean = query_text.strip()
    q_low = q_clean.lower()

    exact_entry = cache_store.get(q_clean)
    if exact_entry and exact_entry.get("prewarmed") is True:
        return {"status": "CACHE_HIT_PREWARMED_EXACT", **exact_entry}

    for entry in cache_store.values():
        if entry.get("prewarmed") is not True:
            continue
        anchors: List[Tuple[str, str]] = entry.get("anchors", [])  # type: ignore[assignment]
        for a1, a2 in anchors:
            if a1.lower() in q_low and a2.lower() in q_low:
                return {"status": "CACHE_HIT_PREWARMED_ANCHOR", **entry}

    if exact_entry:
        return {"status": "CACHE_HIT_ADHOC", **exact_entry}
    return None


if __name__ == "__main__":
    assert is_cacheable_complete_query("두꺼비집 ㄱ") is False
    assert is_cacheable_complete_query("두꺼비집이 내려가요") is True

    demo_cache = {
        "두꺼비집 쇼트 원인": {
            "prewarmed": True,
            "rewritten_text": "배선용차단기 누전차단기 단락전류 과전류 접지",
            "anchors": [("두꺼비집", "내려가"), ("두꺼비집", "쇼트")],
        },
        "공장 두꺼비집이 자꾸 내려가요": {
            "prewarmed": False,
            "rewritten_text": "오래된 부정확 임시 캐시",
        },
    }
    hit = lookup_priority_rewrite_cache("공장 두꺼비집이 자꾸 내려가요", demo_cache)
    assert hit is not None and hit["status"] == "CACHE_HIT_PREWARMED_ANCHOR"
```
