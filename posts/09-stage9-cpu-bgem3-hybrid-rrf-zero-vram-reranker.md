# [9부 · Stage 9] 외부 벡터 DB 없이 45.8만 건 의미 검색 완성하기: GPU VRAM을 0바이트 쓰는 CPU `BGE-M3` 사이드카와 `SQLite FTS5`의 2단계 가중 RRF 결합

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 전문 용어를 몰라서 *"비밀은 안 알려주고 정답을 안다는 사실만 증명하는 법"*처럼 순수 비유로만 물어보면 단어 찾기 검색기가 문서를 놓쳤습니다.
> - **어떻게 고쳤나요?** 귀한 그래픽카드 메모리는 건드리지 않고 **남아 있는 일반 CPU에서 가벼운 '뜻(의미) 판독기(`BGE-M3`)'를 함께 돌려, 단어 찾기 점수와 뜻 찾기 점수를 합산**함으로써 비유 질문도 `100%` 찾아내게 했습니다.

> **핵심 요약 (Stage 9 — Zero-VRAM 시맨틱 리랭커 & Int8 코어 앵커 캐시)**
> - **마주친 현실적 한계 (발전 이유)**: `SQLite FTS5 + 한국어 2-Gram`은 법령명·조문번호·영문 약어가 하나라도 겹치면 `~5ms` 만에 정확한 원문을 찾지만, 사용자가 전문 용어를 단 한 글자도 쓰지 않고 순수 비유나 풀어쓴 설명(예: *"비밀 내용은 한 글자도 안 알려주고 내가 정답을 안다는 사실만 확신시키는 검증법"* → `영지식 증명`)으로 질문하면 `:8080` LLM 쿼리 번역기를 거치지 않고서는 어휘 매칭이 실패했습니다. 반면 `Milvus`·`Qdrant` 같은 외부 벡터 DB 데몬은 상시 `2~4 GB` 가상메모리를 점유하고 단일 파일 `64KB HWID` 잠금 체계를 깨뜨립니다.
> - **도입한 아키텍처 해법**: 디스크에 잠자고 있던 `SLOT_0` 챔피언 **`bge-m3-q8_0.gguf`(`0.59 GB`)를 `LLAMA_ARG_DEVICE=none` · `LLAMA_ARG_N_GPU_LAYERS=0`(순수 CPU, 추가 GPU VRAM `0 MiB`) 사이드카(`127.0.0.1:8082`)로 점화**했습니다. 45.8만 건 전체를 무식하게 벡터화하는 대신 **① `207종` 핵심 마스터·불변 표준 코어 앵커의 `Int8` 양자화 벡터 캐시(`1 KB/문서`, 평문 저장 `0 Byte`)를 통한 어휘 0% 매칭 구제(`Dense Rescue`)**와 **② `FTS5 Top-25` 후보군에 대한 2단계 레짐 적응형 `Reciprocal Rank Fusion (RRF, k=60)` 재순위화**를 결합했습니다.
> - **검증된 정량 지표**: `BGE-M3` 사이드카 기동 전후 **추가 GPU VRAM `0 MiB`** (`1,742 ms` 기동, 단일 질의 임베딩 `73.0 ms`) · `Int8` 양자화 코사인 복원 충실도 **`> 0.9995`** (`704 KB` 캐시 DB) · `:8080` LLM 쿼리 번역기 OFF 상태에서 순수 비유 질문 Top-3 적중률 **`50% (5/10) → 100% (10/10)` 완결** (영지식 증명·비잔틴 장군·모의 담금질·가상 메모리·양자 키 분배 등 어휘 0% 일치 문서 전수 구제 성공).

---

## 1. 왜 첫 실험에서 단순 RRF가 `7.7초` 지연과 오탐을 냈고, 어떻게 해결했는가?

실제 45.8만 건 카트리지 위에서 초기 프로토타입을 돌렸을 때 두 가지 생생한 공학적 난관이 드러났습니다:

1. **콜드 후보 실시간 임베딩 병목 (`7.7초` → `0.25초` 단축)**: `FTS5`가 추린 미캐싱 긴 본문 10개를 매번 CPU에서 실시간 임베딩하자 검색 1회에 `7.7초`가 걸렸습니다. 이를 해결하기 위해 `207종` 핵심 코어 앵커는 백그라운드에서 1회 사전 임베딩(`Int8` 영속화)하고, 런타임 미캐싱 후보는 상위 핵심 요약 `200자` 이내 최대 `4개`로 캡핑하여 지연을 대폭 줄였습니다.
2. **단일 비유 단어('열쇠', '노드')에 의한 객관식 기출 문항 역전 현상 해결**: 비유 질문 속 *"열쇠"*라는 한 단어만 우연히 포함된 일반 문서나 객관식 기출(`KMMLU`)이 어휘 순위 `r_lex = 1`을 차지하면서 정작 의미 유사도 1위(`r_sem = 1`)인 핵심 개념 문서를 밀어내는 현상이 관찰되었습니다. 이에 **어휘 신뢰도가 높은 법령·마스터 자산(`lex_score >= 28.0`)에는 `w_lex = 1.38`을 부여해 기존 `100%` 정확도를 보호하고, 우연한 1단어 일치(`lex_score < 28.0`)에는 `w_lex = 0.45~0.65`, 고유사도 시맨틱 일치(`cos >= 0.50`)에는 `w_sem = 1.18`을 부여하는 레짐 분리형 가중 RRF**를 완성했습니다.

---

## 2. 평문 무저장 `Int8` 벡터 양자화기 & 레짐 분리형 가중 `RRF(k=60)` (Pure Python)

서드파티 `numpy`나 외부 벡터 DB 없이 파이썬 표준 `array('b')`만으로 `1024`차원 실수 벡터를 `1,024 Byte`로 압축(`코사인 오차 < 0.0005`)하고 어휘·의미 순위를 결합하는 핵심 코드입니다.

```python
import array
import math
from typing import Dict, List, Tuple


def normalize_l2(vec: List[float]) -> Tuple[float, ...]:
    norm = math.sqrt(sum(x * x for x in vec))
    if norm <= 1e-12:
        return tuple(0.0 for _ in vec)
    return tuple(x / norm for x in vec)


def quantize_unit_vec_int8(unit_vec: Tuple[float, ...]) -> Tuple[float, bytes]:
    """1024차원 float 벡터(8KB)를 1KB int8 BLOB + scale 1개로 무손실급 압축한다."""
    max_abs = max((abs(v) for v in unit_vec), default=1e-8)
    scale = max(max_abs, 1e-8) / 127.0
    q_arr = array.array("b", [max(-127, min(127, int(round(v / scale)))) for v in unit_vec])
    return scale, q_arr.tobytes()


def dequantize_int8_to_unit_vec(scale: float, blob: bytes) -> Tuple[float, ...]:
    q_arr = array.array("b")
    q_arr.frombytes(blob)
    return normalize_l2([float(v) * scale for v in q_arr])


def fuse_lexical_and_bgem3_rrf(
    candidates: List[Tuple[str, float, float]],
    rrf_k: int = 60,
) -> List[Tuple[str, float]]:
    """
    candidates: [(doc_id, lexical_score, bge_m3_cosine), ...]
    어휘 점수가 0점인 순수 비유 질문(Dense Rescue)에서도 고유사도 코어 문서를 Top-1으로 구제하며,
    정확한 조문·타이틀 일치(lexical_score >= 28.0)는 흔들림 없이 보호한다.
    """
    lex_sorted = [ c[0] for c in sorted(candidates, key=lambda x: x[1], reverse=True) if c[1] > 0 ]
    sem_sorted = [ c[0] for c in sorted(candidates, key=lambda x: x[2], reverse=True) ]
    lex_rank: Dict[str, int] = {did: i + 1 for i, did in enumerate(lex_sorted)}
    sem_rank: Dict[str, int] = {did: i + 1 for i, did in enumerate(sem_sorted)}

    fused: List[Tuple[str, float]] = []
    for doc_id, lex_sc, cos_val in candidates:
        r_l = lex_rank.get(doc_id, 22)
        r_s = sem_rank.get(doc_id, 30)
        w_lex = 1.38 if lex_sc >= 28.0 else 0.65
        w_sem = 1.18 if cos_val >= 0.50 else (1.02 if cos_val >= 0.43 else 0.0)
        score = (w_lex / (rrf_k + r_l)) + (w_sem / (rrf_k + r_s)) + min(0.0040, lex_sc / 25000.0)
        fused.append((doc_id, round(score, 6)))

    fused.sort(key=lambda item: item[1], reverse=True)
    return fused


if __name__ == "__main__":
    sample_vec = normalize_l2([math.sin(i * 0.11) for i in range(1024)])
    sc, raw_b = quantize_unit_vec_int8(sample_vec)
    restored = dequantize_int8_to_unit_vec(sc, raw_b)
    assert len(raw_b) == 1024
    assert sum(a * b for a, b in zip(sample_vec, restored)) > 0.9995

    ranked = fuse_lexical_and_bgem3_rrf([
        ("NOISY-LITERAL-KEY-DOC", 14.0, 0.41),   # '열쇠' 1단어 우연 일치
        ("ZERO-KNOWLEDGE-PROOF", 0.0, 0.61),     # 어휘 0개 일치 · BGE-M3 의미 적중
    ])
    assert ranked[0][0] == "ZERO-KNOWLEDGE-PROOF"
```
