# [2부 · Stage 2] 하드웨어 물리 한계 직시: 44GB 스와핑 구성을 버리고 '13.9GB 100% VRAM 직결 4-슬롯 금고'와 '가상메모리(Commit) OOM 가드'로 진화하기

> **핵심 요약 (Stage 2 — 하드웨어 물리 최적화)**
> - **마주친 현실적 한계 (발전 이유)**: 초기에는 `44.0 GB` 규모의 대형 MoE(`17.8GB`) 부분 오프로딩 구성을 시험했으나, 크롬 다중 탭·문서 도구·IDE를 동시에 켜두는 실제 데스크톱 환경에서는 **① 시스템 DDR4 RAM 대역폭 병목으로 인한 속도 급락(`~6.2 tok/s`)**과 **② 윈도우 가상메모리(Commit Charge) 부족에 의한 `vk::Device::allocateMemory` Vulkan OOM 강제 종료**가 발생했습니다.
> - **도입한 아키텍처 해법**: 모델 크기 허영을 버리고 `RTX 3060 12GB GDDR6(360 GB/s)` 내부에 가중치 100%가 상주하는 **`Qwen3-14B-Q4_K_M`(`8.38 GB`, 주력) + `Qwen3-4B-Q4_K_M`(`2.33 GB`, 초고속 보호 폴백) + `Qwen2.5-VL-3B`(`2.58 GB`, 오프라인 도면 OCR) + `BGE-M3`(`0.61 GB`, 시맨틱 색인)** 총 **`13.90 GB` 4-슬롯 롤링 금고**로 재편했습니다. 동시에 기동 전 VRAM과 윈도우 가상메모리(Commit) 하한선을 모두 검문하는 **3단계 프리플라이트 메모리 가드**를 구축했습니다.
> - **검증된 정량 지표**: 디스크 점유 `44.0 GB → 13.90 GB`(`68.4%` 감량) 고정 · 주력 **`31~33 tok/s`** / 스마트 폴백 **`71~83 tok/s`** 무중단 구동 · 가상메모리 고갈 시 상위 점유 프로세스(PID·GB) 자동 지목 및 라이선스 차감 없는 안전 차단(`OOM 크래시 0건`).

---

## 1. 메모리 대역폭 루프라인(Roofline) 법칙: 왜 '어중간한 대형 모델'보다 '100% VRAM 14B'인가?

로컬 LLM의 토큰 생성 속도($\text{TPS}$)는 연산 능력(FLOPS)이 아니라 **실효 메모리 대역폭($\text{BW}_{\text{eff}}$)을 활성 가중치 크기($M_{\text{active}}$)로 나눈 값**에 지배됩니다.

- 가중치가 `12GB VRAM`을 단 `1~2 GB`라도 넘어 시스템 RAM(`~40 GB/s`)으로 흘러넘치는 순간 PCIe 버스 병목에 걸려 속도가 `1/5` 토막(`6.2 tok/s`)으로 추락합니다.
- 반면 `Qwen3-14B-Q4_K_M`(`8.38 GB`)을 KV 캐시(`q8_0`)와 함께 `12GB GDDR6 VRAM`(`360 GB/s`) 내부에 100% 가두면 **초당 `31~33 토큰`의 일관된 고성능**을 유지합니다.

---

## 2. 4-슬롯 롤링 스냅샷 금고 (`총 13.90 GB` 고정 · `1-In, 1-Out` 교체 규율)

| 슬롯 ID | 담당 역할 | 챔피언 모델 (`GGUF`) | 용량 | 최소 요구 사양 | 실측 성능 / 벤치마크 |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **`SLOT_0`**<br>`EMBED_RERANK` | 다국어 시맨틱 임베딩 & 하이브리드 색인 | `bge-m3-q8_0.gguf` | **`0.61 GB`** | RAM 4GB / VRAM 0GB | 종합 **`91.0점`** · 한·영 의미 교차 매칭 |
| **`SLOT_1`**<br>`LIGHT_FALLBACK` | 초고속 요약·분류 & 메모리 부족 시 스마트 폴백 | `Qwen3-4B-Q4_K_M.gguf` | **`2.33 GB`** | RAM 8GB / VRAM 4GB | **`71~83 tok/s`** · 종합 `78.5점` |
| **`SLOT_2`**<br>`MAIN_ENGINE` | **주력 심층 추론 · 법령/계약/기술 오픈북 두뇌** | `Qwen3-14B-Q4_K_M.gguf` | **`8.38 GB`** | RAM 16GB / VRAM 10GB | **`31~33 tok/s` (100% VRAM)** · 종합 **`89.2점`** |
| **`SLOT_3`**<br>`VISION_OCR` | 스캔 PDF·도면·시험성적서·표 오프라인 판독 | `Qwen2.5-VL-3B-Q4_K_M` + `mmproj` | **`2.58 GB`** | RAM 8GB / VRAM 4GB | 도면 1장 **`~3.8초`** · 종합 `86.4점` |
| **합계** | **4개 직교 슬롯 정원 고정** | **`1-In, 1-Out` 자동 교체 규율** | **`13.90 GB`** | 일반 `RTX 3060 12GB` PC | 무분별한 모델 누적 원천 방지 |

---

## 3. 윈도우 가상메모리(Commit) 고갈에 의한 Vulkan OOM 사전 차단 가드 (Pure Python)

*"VRAM은 `10.6 GB`나 비어 있는데 왜 서버가 죽는가?"* — 실제 운영에서 밝혀낸 원인은 크롬 다중 탭 등이 윈도우 **가상메모리 커밋 한도(Pagefile Commit Limit)**를 소진해 드라이버가 주소 공간을 예약하지 못하는 현상이었습니다. 이를 해결한 프리플라이트 가드입니다.

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
