# [8부] "두꺼비집이 내려가요"를 '배선용차단기 단락전류'로: 3-Tier 구어체 쿼리 번역기와 타이핑 디바운스 큐 병목 해결기

> **핵심 요약 (Executive Summary)**
> - **발전시켜 나간 이유 1 (일상 구어체와 전문 용어의 단절)**: 45.8만 건의 백과사전·법령·의학 DB를 구축해도, 실제 사용자는 *"공장 기계 돌리니까 두꺼비집이 자꾸 내려가"*, *"타이레놀 먹고 술 마셔도 돼?"*, *"비 오니까 상가 천장 누수 법적 청구"*처럼 일상 언어로 질문합니다. 반면 전문 문서는 `배선용차단기 단락전류 접지`, `아세트아미노펜 NAPQI 간독성`, `민법 제667조 도급 하자담보책임`으로 적혀 있어 단순 키워드 검색만으로는 적중률이 떨어집니다.
> - **발전시켜 나간 이유 2 (실시간 타이핑 디바운스가 부른 `89.7초` GPU 큐 정체 사고)**: 이 간극을 메우려고 로컬 LLM이 구어체를 전문 검색어로 바꿔주는 `Neural Query Rewriter`를 붙였는데, 검색창 실시간 미리보기(`90ms` 디바운스)마다 번역기가 호출되면서 **단일 추론 슬롯(`N_PARALLEL=1`) GPU에 `'내'`, `'내 ㄱ'`, `'내 개...'` 같은 타이핑 도중의 미완성 글자 조각 40여 개가 줄지어 쌓여 정작 본 답변 생성이 `89.70초`까지 밀리고 캐시가 오타로 오염되는 실제 병목**을 발견했습니다.
> - **3-Tier 게이트 & 타이핑/실행 완전 분리 해법**: **① 40선 상황별 구어체 시드 앵커(`<0.05ms`) → ② `mtime` 자동 핫리로드 영구 보안 캐시(미완성 한글 자모 기록 차단) → ③ 실행 클릭 시에만 단 1회 작동하는 로컬 LLM 번역기**로 계층화하고, 타이핑 중(`250ms` + `AbortController`)에는 `?neural=0` 플래그로 GPU 호출을 **`0회`**로 격리하여 체감 응답 시간을 **`89.70초 → 22.66초`(`~4배` 가속)**로 단축했습니다.

---

## 1. 일상 구어체 → 표준 검색어 변환 예시 (4대 도메인 40선 프리웜 시드)

콜드 스타트 지연조차 없애기 위해 현장에서 가장 빈번한 40가지 상황별 구어체 질문 패턴과 핵심 상황 앵커 쌍(Anchor Pair)을 기본 내장했습니다.

| 도메인 | 사용자의 실제 일상 구어체 질문 | 변환되는 표준 FTS5 검색 키워드 (`<0.05 ms` 즉시 매칭) |
| :--- | :--- | :--- |
| **법령 (`L01`)** | *"일하다가 다쳐서 병원 입원했는데 회사에서 치료비랑 쉬는 동안 월급 얼마나 줘?"* | `근로기준법 업무상 부상 요양보상 휴업보상 평균임금 100분의 60 제78조 제79조` |
| **의학 (`M01`)** | *"어제 술 마시고 머리가 깨질 것 같은데 타이레놀 먹어도 괜찮아?"* | `아세트아미노펜 알코올 음주 병용금기 간독성 NAPQI 글루타치온 급성간부전` |
| **공학 (`E01`)** | *"공장에서 기계 돌리니까 두꺼비집이 자꾸 퍽 소리 나면서 내려가는데 쇼트난 걸까?"* | `두꺼비집 단락전류 과전류 접지 배선용차단기 누전차단기 전기설비` |
| **전산 (`P01`)** | *"파이썬으로 웹페이지 긁어오는데 상대 서버가 멈춰있으면 내 프로그램도 영원히 멈춰버려"* | `웹크롤링 Python timeout 소켓 블로킹 asyncio 비동기 교착상태` |

---

## 2. 타이핑 미리보기(`?neural=0`)와 본 질의 실행(`?neural=1`)의 물리적 격리 구조

```text
[사용자 키보드 타이핑 중 ("두꺼비집이 자꾸...")]
   |-- 250ms 디바운스 + 이전 요청 AbortController 즉시 취소
   v
GET :8081/api/fts5-search?q=...&neural=0
   |-- Step 0: 로컬 프리웜 시드 / 앵커 캐시 조회 (< 0.05 ms)
   |-- Step 1: 정적 동의어 사전 + 2-Gram FTS5 실시간 검색 (~ 5 ms)
   |-- [중요] :8080 GPU LLM 호출 = 0회! (큐 점유 0%, 미완성 자모 캐싱 차단)

[사용자 Enter / '직접 질의 실행' 버튼 클릭!]
   v
GET :8081/api/fts5-search?q=...&neural=1
   |-- 캐시 적중 시: 0.05 ms 만에 표준 키워드 확장 완료
   |-- 캐시 미적중 시에만 :8080 Qwen3-14B (/no_think) 단 1회 호출 (~ 1.3s) 후 영구 캐싱
   v
POST :8080/v1/chat/completions (검증된 원문 근거 기반 최종 답변 즉시 생성: 22.6s)
```

---

## 3. 프리웜 우선순위 보장 및 미완성 한글 자모 오염 방지 캐시 엔진 (Pure Python)

운영 과정에서 **구버전 임시(Ad-hoc) 캐시 항목이 정교하게 큐레이션된 프리웜 앵커(`prewarmed=True`)보다 먼저 매칭되어 엉뚱한 법령으로 유도하는 현상**까지 잡아내어, ① 프리웜 완전일치 → ② 프리웜 앵커 매칭 → ③ 일반 사용자 캐시 순으로 우선순위를 고정하고 조합 중인 한글 자모(`[ㄱ-ㅎㅏ-ㅣ]$`) 저장을 원천 차단했습니다.

```python
import re
from typing import Dict, List, Optional, Tuple

INCOMPLETE_HANGUL_JAMO_RE = re.compile(r"[ㄱ-ㅎㅏ-ㅣ]$")


def is_cacheable_complete_query(query_text: str) -> bool:
    """타이핑 도중의 짧은 글자 조각(<3자)이나 조합 중인 한글 자모로 끝나는 문자열의 캐시 오염을 차단한다."""
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
    2순위: 큐레이션된 프리웜 상황 앵커 쌍(Anchor-Pair) 포함 일치
    3순위: 일반 런타임 번역 캐시 완전 일치
    """
    q_clean = query_text.strip()
    q_low = q_clean.lower()

    # 1. Prewarmed exact match
    exact_entry = cache_store.get(q_clean)
    if exact_entry and exact_entry.get("prewarmed") is True:
        return {"status": "CACHE_HIT_PREWARMED_EXACT", **exact_entry}

    # 2. Prewarmed anchor-pair match (beats stale ad-hoc entries)
    for entry in cache_store.values():
        if entry.get("prewarmed") is not True:
            continue
        anchors: List[Tuple[str, str]] = entry.get("anchors", [])  # type: ignore[assignment]
        for a1, a2 in anchors:
            if a1.lower() in q_low and a2.lower() in q_low:
                return {"status": "CACHE_HIT_PREWARMED_ANCHOR", **entry}

    # 3. Ad-hoc exact match
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
    assert "배선용차단기" in str(hit["rewritten_text"])
```
