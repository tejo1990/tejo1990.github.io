# [2부] 소비자용 GPU(`RTX 3060 12GB`) 한 장으로 끝내는 '4-슬롯 MoE 롤링 볼트' & 오프라인 하이브리드 RAG 설계

> **핵심 요약 (Executive Summary)**
> - **원터치 설치(Vanilla Ollama)의 함정**: 하드웨어 메모리 대역폭(Roofline) 계산 없이 무작정 큰 모델을 받으면 PCIe 버스 스와핑으로 속도가 1/10로 추락하고, 작은 모델을 받으면 추론 품질이 붕괴합니다. 또한 수백 GB의 모델을 무분별하게 받아 디스크만 채우게 됩니다.
> - **4-슬롯 하드웨어 체급 매트릭스 (`총 44.0 GB` 고정)**: 모든 환경(무외장 그래픽 노트북 ~ `RTX 3060 12GB` 주력 데스크톱 ~ 고사양 워크스테이션)을 단 **4개의 슬롯(`SLOT_0` ~ `SLOT_3`)**으로 100% 커버하고, 상위 호환 모델이 나오면 구형 모델을 즉시 삭제하는 **`1-In, 1-Out` 롤링 볼트 규율**로 유지보수 비용을 `0`으로 수렴시켰습니다.
> - **하이브리드 로컬 검색 (`SQLite FTS5` + `BGE-M3`)**: 무거운 외부 벡터 DB 서버(Docker/Milvus)를 띄우는 대신, 파이썬 내장 `SQLite FTS5` 형태소/키워드 색인과 `bge-m3-q8_0.gguf`(`1.2GB`) 임베딩을 결합해 **오프라인 1초 내 정밀 검색**을 구현했습니다.

---

## 1. 왜 `RTX 3060 12GB VRAM`에서 `30B-A3B` MoE 구조가 챔피언인가?

일반적인 Dense 모델(예: `32B Q4_K_M`, 약 `19.8 GB`)은 `12 GB VRAM`에 다 들어가지 않아 나머지 `8 GB`를 시스템 DDR4 RAM에 올려야 합니다. 이 순간 매 토큰을 생성할 때마다 느린 시스템 메모리 대역폭(`~40 GB/s`)이 전체 속도를 지배하여 초당 2~3토큰 수준으로 느려집니다.

반면 본 아키텍처의 주력 슬롯(`SLOT_2`)에 배치한 **MoE(Mixture-of-Experts) 아키텍처(`Qwen3-30B-A3B-Q4_K_M`, `17.8 GB`)**는 다음과 같은 물리적 이점을 가집니다:
- **전체 지식 용량(`Total Parameters`)은 `30B`**로 대형 모델의 지식을 그대로 유지합니다.
- **토큰당 실제 활성화되는 연산 파라미터(`Active Parameters`)는 단 `3B`**뿐이므로, GPU VRAM에 핫 레이어와 KV 캐시를 고정하고 희소 전문가(Routed Experts)만 부분 호출하여 **`RTX 3060 12GB + 16GB RAM` 환경에서도 초고속 추론 속도**를 뽑아냅니다.

---

## 2. 4-슬롯 롤링 볼트 (`4-Slot Rolling Vault` — 총 `44.0 GB` 고정)

디스크 여유 공간을 낭비하지 않고 전 환경을 커버하는 4-슬롯 표준 구성표입니다.

| 슬롯 ID | 타겟 하드웨어 사양 | 챔피언 가중치 및 비중국계 오픈웨이트 대안 | 용량 | 핵심 아키텍처 역할 |
| :--- | :--- | :--- | :---: | :--- |
| **`SLOT_0`**<br>*(하이브리드 RAG)* | **전 사양 공통 필수**<br>*(CPU/GPU 무관)* | `bge-m3-q8_0.gguf` + `SQLite FTS5` 인덱스 | **`1.2 GB`** | 로컬 마크다운/PDF/코드베이스를 1초 내 키워드+의미 하이브리드 검색 |
| **`SLOT_1`**<br>*(경량 노트북)* | **일반 사무용 노트북**<br>*(내장그래픽 + RAM 12~16GB)* | `Qwen3-8B-Q4_K_M.gguf`<br>*(대안: `Gemma-3-12B` / `EXAONE-3.5-7.8B`)* | **`5.2 GB`** | 외장 GPU가 없는 출강용/사무용 노트북에서 초당 12~18글자 구동 보장 |
| **`SLOT_2`**<br>*(주력 데스크톱)* | **주력 환경 (`RTX 3060 12GB`)**<br>*(12GB VRAM + 16~32GB RAM)* | `Qwen3-30B-A3B-Q4_K_M.gguf`<br>*(대안: `Mistral-Small-24B`)* | **`17.8 GB`** | **핵심 주력 엔진**: 30B 지식과 3B 연산 속도를 결합한 가성비 극대화 슬롯 |
| **`SLOT_3`**<br>*(워크스테이션)* | **고사양 GPU / 통합메모리**<br>*(RTX 4090/5090 또는 36GB+)* | `Qwen3-32B-Q4_K_M.gguf`<br>*(대안: `Gemma-3-27B` / `EXAONE-32B`)* | **`19.8 GB`** | 고난도 수학·아키텍처 심층 추론용 Dense 풀파워 슬롯 |
| **합계** | **4개 슬롯 정원 고정** | **`1-In, 1-Out` 교체 규율 적용** | **`44.0 GB`** | 256GB 외장 SSD 1개 또는 로컬 디스크의 10% 미만으로 완벽 수납 |

---

## 3. 결정론적 `1-In, 1-Out` 슬롯 교체 판정 코드 (Pure Python)

새 모델이 나왔을 때 무조건 다운로드하는 대신, **기존 슬롯의 용량 상한(`slot_max_gb`)을 넘지 않으면서 벤치마크 점수가 엄격히 상승($S_{\text{new}} > S_{\text{curr}}$)할 때만 구형 가중치를 삭제하고 교체**하는 순수 파이썬 검증 코드입니다.

```python
from dataclasses import dataclass
from typing import Dict, List


@dataclass(frozen=True)
class RollingSlotSpec:
    slot_id: str
    current_model: str
    current_size_gb: float
    current_score: float
    candidate_model: str
    candidate_size_gb: float
    candidate_score: float
    slot_max_gb: float


def evaluate_rolling_vault_swap(
    slots: List[RollingSlotSpec],
    total_storage_cap_gb: float = 75.0,
) -> Dict[str, object]:
    """4-슬롯 롤링 볼트의 1-In, 1-Out 교체 조건 및 용량 상한 불변량을 검증한다."""
    swap_decisions: Dict[str, str] = {}
    final_total_gb = 0.0

    for s in slots:
        fits_slot = s.candidate_size_gb <= s.slot_max_gb
        is_strictly_better = s.candidate_score > s.current_score
        if fits_slot and is_strictly_better:
            swap_decisions[s.slot_id] = f"SWAP: Delete {s.current_model} -> Keep {s.candidate_model}"
            final_total_gb += s.candidate_size_gb
        else:
            swap_decisions[s.slot_id] = f"KEEP: Retain {s.current_model} (Reject {s.candidate_model})"
            final_total_gb += s.current_size_gb

    return {
        "total_footprint_gb": round(final_total_gb, 2),
        "within_storage_cap": final_total_gb <= total_storage_cap_gb,
        "decisions": swap_decisions,
    }
```
