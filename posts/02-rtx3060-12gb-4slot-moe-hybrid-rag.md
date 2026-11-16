# [2부] 하드웨어 물리 한계 직시: 왜 44GB 스와핑 구성을 버리고 '13.9GB 100% VRAM 직결 4-슬롯 금고'와 '가상메모리(Commit) OOM 가드'로 진화했는가?

> **핵심 요약 (Executive Summary)**
> - **초기 설계(`44.0 GB` MoE 부분 오프로딩)에서 마주친 현실의 벽**: 이론상 `30B-A3B(17.8GB)` 모델을 VRAM과 시스템 DDR4 RAM에 걸쳐 올리면 잘 돌아갈 것 같지만, 실제 업무 환경(크롬 탭 25개, 문서 편집기, IDE 동시 실행)에서는 **① 시스템 RAM 대역폭 병목으로 인한 속도 급락(`~6.2 tok/s`)**과 **② 윈도우 가상메모리(Commit Charge) 한도 도달에 의한 `vk::Device::allocateMemory` Vulkan OOM 강제 종료**가 발생했습니다.
> - **아키텍처 결단 — '100% VRAM 상주 원칙'과 역할 분담 (`총 13.90 GB` 고정)**: 모델 크기에 대한 허영을 버리고, `RTX 3060 12GB GDDR6(360 GB/s)` 내부에 가중치 100%가 들어가는 **`Qwen3-14B-Q4_K_M`(`8.38 GB`, `~31~33 tok/s`)**을 주력 추론 두뇌로 고정했습니다. 대신 부족한 암기 지식은 SSD의 `45.8만 건 SQLite FTS5` 카트리지(7부)로 외장화하고, 스캔 도면 판독은 **`Qwen2.5-VL-3B`(`2.58 GB`)** 시각 전용 슬롯에 분리했습니다.
> - **실시간 VRAM & 윈도우 가상메모리(Commit) 2중 가드**: 서버 기동 전 남은 VRAM뿐 아니라 **윈도우 커밋 여유량**까지 실측하여, 여유가 충분하면 `14B`(`31+ TPS`), 백그라운드 앱 점유가 높으면 `4B`(`71+ TPS`)로 자동 폴백하고, 최소 하한선(`3.5 GB`) 미만이면 **가상메모리를 가장 많이 점유 중인 상위 프로세스(이름·PID·GB)를 정확히 지목해 안내한 뒤 라이선스 차감 없이 안전 중단**합니다.

---

## 1. 메모리 대역폭 루프라인(Roofline) 법칙: 왜 '어중간한 30B'보다 '100% VRAM 14B + 오픈북 DB'가 압도적인가?

로컬 LLM의 토큰 생성 속도($\text{TPS}$)는 연산 능력(FLOPS)이 아니라 **메모리 대역폭($\text{BW}_{\text{eff}}$)을 활성 가중치 크기($M_{\text{active}}$)로 나눈 값**에 지배됩니다.

- **가중치가 `12GB VRAM`을 단 `1~2 GB`라도 초과하는 순간**: PCIe 버스와 시스템 DDR4 RAM(`~40 GB/s`)이 병목이 되어 체감 속도가 `1/5` 토막(`6.2 tok/s`)으로 추락합니다.
- **가중치 100%를 `12GB GDDR6 VRAM`(`360 GB/s`) 안에 가두면**: `Qwen3-14B-Q4_K_M`(`8.38 GB`)이 KV 캐시(`q8_0`)까지 포함해 VRAM 내부에서만 완결되므로 **초당 `31~33 토큰`의 즉각적인 응답 속도**를 유지합니다.

---

## 2. 진화된 4-슬롯 롤링 스냅샷 금고 (`총 13.90 GB` 고정 · `1-In, 1-Out` 규율)

무분별하게 모델을 쌓아두지 않고, 직교하는 4개의 역할 슬롯만 유지하며 상위 모델 출시 시 자동 스모크 테스트 후 교체합니다.

| 슬롯 ID | 담당 역할 | 챔피언 모델 (`GGUF`) | 용량 | 최소 요구 사양 | 실측 성능 / 벤치마크 |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **`SLOT_0`**<br>`EMBED_RERANK` | 다국어 시맨틱 임베딩 & 하이브리드 색인 | `bge-m3-q8_0.gguf` | **`0.61 GB`** | RAM 4GB / VRAM 0GB | 종합 **`91.0점`** · 한·영 의미 교차 매칭 |
| **`SLOT_1`**<br>`LIGHT_FALLBACK` | 초고속 요약·분류 & 메모리 부족 시 스마트 폴백 | `Qwen3-4B-Q4_K_M.gguf` | **`2.33 GB`** | RAM 8GB / VRAM 4GB | **`71~83 tok/s`** · 종합 `78.5점` |
| **`SLOT_2`**<br>`MAIN_ENGINE` | **주력 심층 추론 · 법령/계약/기술 오픈북 두뇌** | `Qwen3-14B-Q4_K_M.gguf` | **`8.38 GB`** | RAM 16GB / VRAM 10GB | **`31~33 tok/s` (100% VRAM)** · 종합 **`89.2점`** |
| **`SLOT_3`**<br>`VISION_OCR` | 스캔 PDF·도면·시험성적서·표 오프라인 판독 | `Qwen2.5-VL-3B-Q4_K_M` + `mmproj` | **`2.58 GB`** | RAM 8GB / VRAM 4GB | 도면 1장 **`~3.8초`** · 종합 `86.4점` |
| **합계** | **4개 직교 슬롯 정원 고정** | **`1-In, 1-Out` 자동 교체 규율** | **`13.90 GB`** | 일반 `RTX 3060 12GB` PC | 디스크 점유 기존 대비 **`68.4%` 감량** |

---

## 3. 윈도우 가상메모리(Commit) 고갈에 의한 Vulkan OOM 사전 차단 가드 (Pure Python)

실제 운영 중 **"VRAM은 `10.6 GB`나 비어 있는데 왜 갑자기 서버가 죽거나 느려지는가?"**라는 문제를 추적한 결과, 원인은 GPU가 아니라 크롬 다중 탭 등이 윈도우 **가상메모리 커밋 한도(Pagefile Commit Limit)**를 소진해 드라이버가 주소 공간을 예약하지 못하는 현상이었습니다. 이를 해결한 3단계 프리플라이트(Pre-Flight) 메모리 가드 로직입니다.

```python
from dataclasses import dataclass
from typing import Dict, List


@dataclass(frozen=True)
class ProcessCommitUsage:
    name: str
    pid: int
    commit_gb: float


def decide_gpu_launch_tier(
    free_vram_gb: float,
    free_virtual_commit_gb: float,
    top_hogs: List[ProcessCommitUsage],
) -> Dict[str, object]:
    """
    VRAM 여유량과 윈도우 가상메모리(Commit) 여유량을 동시에 검문하여
    14B 주력 구동 / 4B 스마트 폴백 / OOM 사전 차단(쿼터 차감 0)을 결정론적으로 판정한다.
    """
    if free_vram_gb >= 10.0 and free_virtual_commit_gb >= 9.8:
        return {
            "guard_mode": "OPTIMAL_14B_100PCT_VRAM",
            "can_launch_server": True,
            "selected_model": "Qwen3-14B-Q4_K_M.gguf",
            "expected_tps": 31.8,
            "consume_license_quota": True,
        }

    if free_vram_gb >= 3.1 and free_virtual_commit_gb >= 3.5:
        return {
            "guard_mode": "SMART_FALLBACK_4B_FAST_PROTECT",
            "can_launch_server": True,
            "selected_model": "Qwen3-4B-Q4_K_M.gguf",
            "expected_tps": 71.2,
            "consume_license_quota": True,
        }

    culprits = [f"{p.name}(PID {p.pid}, {p.commit_gb:.2f}GB)" for p in top_hogs[:3]]
    return {
        "guard_mode": "ABORT_INSUFFICIENT_VIRTUAL_COMMIT",
        "can_launch_server": False,
        "selected_model": "NONE_ABORTED",
        "consume_license_quota": False,
        "action_guide": f"가상메모리 여유 부족({free_virtual_commit_gb:.1f}GB < 최소 3.5GB). 상위 점유 앱 종료 필요: {', '.join(culprits)}",
    }


if __name__ == "__main__":
    sample = decide_gpu_launch_tier(
        free_vram_gb=10.6,
        free_virtual_commit_gb=0.4,
        top_hogs=[ProcessCommitUsage("chrome.exe", 24436, 4.73)],
    )
    assert sample["can_launch_server"] is False
    assert sample["consume_license_quota"] is False
```
