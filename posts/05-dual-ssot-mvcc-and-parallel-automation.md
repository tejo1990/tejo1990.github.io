# [5부] 기억을 잃지 않는 에이전트: `Dual-SSOT MVCC` 상태 원장과 멀티스레드 일일 자동화 레이더 구축기

> **핵심 요약 (Executive Summary)**
> - **에이전트 기억 부패(Context Rot)와 Split-Brain 문제**: 여러 세션에서 AI 에이전트를 쓰다 보면 이전 세션에서 이미 실패로 판명 난 아이디어를 다음 세션에서 다시 제안하거나, 마크다운 메모 파일과 실제 데이터베이스의 버전이 어긋나는 **Split-Brain 현상**이 발생합니다.
> - **해법 1 — `Dual-SSOT` (SQLite MVCC + L1 Hot / L2 Cold / Negative Knowledge 원장)**: 모든 의사결정을 트랜잭션(`tx_id`) 단위로 기록하고, 특히 **검증 끝에 기각된 가설(`Negative Knowledge`, 현재 125건)**을 별도 테이블로 박제하여 에이전트가 동일한 시행착오를 영원히 반복하지 않도록 설계했습니다.
> - **해법 2 — 라이프사이클 훅(`SessionStart` / `PreToolUse` / `Stop`)과 병렬 마스터 레이더(`run_daily_master_radar.py`)**: 세션 시작 시 토큰 소모 `0`으로 시스템 상태를 띄우고, 위험한 명령어를 사전에 교정하며, `ThreadPoolExecutor`로 4개의 데이터 수집 스캐너를 병렬 실행(`98.2초 -> 39.4초`, **`2.49배` 가속**)해 매일 아침 1페이지 마스터 브리핑을 자동 생성합니다.

---

## 1. `Dual-SSOT` 3계층 메모리 아키텍처

일반 챗봇처럼 대화 내역을 무작정 쌓아두는 대신, 컴퓨터 아키텍처의 **L1/L2 캐시 계층과 데이터베이스 MVCC(다중 버전 동시성 제어)**를 그대로 이식했습니다.

```
+-----------------------------------------------------------------------+
| [Session Start] -> init_session() : MVCC 스냅샷(v257) & L1 Hot 로드   |
+-----------------------------------------------------------------------+
          |
          v
+-----------------------------------------------------------------------+
| 1. L1 Hot Context (ACTIVE_CONTEXT.md)                                 |
|    - 현재 활성화된 최상위 아키텍처 불변량만 고정 (동일 키 In-Place 갱신) |
+-----------------------------------------------------------------------+
| 2. L2 Cold Ledger (SQLite domain_ssot.db)                             |
|    - 당장 쓰지 않는 43개 도메인 히스토리는 L2로 강등 (프롬프트 토큰 0)   |
|    - 필요 시 query_cold_ledger(topic_key)로 즉시 역소환                |
+-----------------------------------------------------------------------+
| 3. Negative Knowledge Registry (DO_NOT_REPEAT — 125건 누적)           |
|    - 반증·기각된 가설과 기각 사유(법령/수식 근거)를 영구 기록            |
|    - 에이전트가 과거의 오답을 다시 제안하는 것을 원천 차단               |
+-----------------------------------------------------------------------+
          |
          v
+-----------------------------------------------------------------------+
| [Turn End] -> promote_session() : 원자적 커밋 & 롤백 영수증(tx_id) 발급 |
+-----------------------------------------------------------------------+
```

또한 Git 병합 과정에서 바이너리 DB(`domain_ssot.db`)의 리비전이 마크다운(`ACTIVE_CONTEXT.md`)보다 낮아지는 사고를 막기 위해, 세션 시작·종료 시 자동으로 리비전 단조 증가성을 검사하고 백업 스냅샷으로 자동 복구하는 **`Monotonic Revision Guard`**를 구현했습니다.

---

## 2. 4대 스캐너 병렬 오케스트레이터 (`run_daily_master_radar.py`)

흩어져 있던 4개의 공공·채용·경진대회 데이터 수집 스크립트를 `concurrent.futures.ThreadPoolExecutor`로 병렬화하고, 이미 확인·처리한 항목(`applied_ledger.json`)을 자동 필터링하여 **`D-5 이내 마감임박 타겟`과 `오늘의 NEW 항목`만 1페이지 마크다운으로 컴파일**하는 자동화 파이프라인입니다.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import subprocess
import sys
import time
from typing import Dict, List, Tuple

SCANNERS: List[Tuple[str, str]] = [
    ("scan_it_genuine_jobs.py", "IT·인프라 진성 공고 스캐너"),
    ("scan_instructor_jobs.py", "AI·SW 교육 강사 공고 스캐너"),
    ("scan_remote_coach_jobs.py", "온라인 부트캠프 튜터·멘토 스캐너"),
    ("scan_step3_hackathons.py", "AI·데이터 경진대회 2단계 검증 스캐너"),
]


def run_single_scanner(script_name: str) -> Tuple[str, int, float]:
    t0 = time.perf_counter()
    proc = subprocess.run(
        [sys.executable, "-X", "utf8", f"scripts/{script_name}"],
        capture_output=True,
        text=True,
        encoding="utf-8",
        timeout=90,
    )
    elapsed = time.perf_counter() - t0
    return script_name, proc.returncode, round(elapsed, 2)


def run_parallel_master_radar() -> Dict[str, object]:
    """4대 스캐너를 병렬 실행하여 순차 실행(98.2s) 대비 2.49배 빠른 39.4s 내에 브리핑을 갱신한다."""
    t_start = time.perf_counter()
    results = {}
    with ThreadPoolExecutor(max_workers=4) as pool:
        futures = {pool.submit(run_single_scanner, s[0]): s[0] for s in SCANNERS}
        for fut in as_completed(futures):
            name, code, sec = fut.result()
            results[name] = {"exit_code": code, "elapsed_sec": sec}

    wall_time = round(time.perf_counter() - t_start, 2)
    seq_time = round(sum(v["elapsed_sec"] for v in results.values()), 2)
    return {
        "wall_time_sec": wall_time,
        "sequential_sum_sec": seq_time,
        "speedup_factor": round(seq_time / max(wall_time, 1e-6), 2),
        "scanners": results,
    }
```

---

## 3. 맺음말: AI 시대의 진짜 엔지니어링 역량이란 무엇인가?

누구나 프롬프트 한 줄로 그럴듯한 글과 코드를 뽑아낼 수 있는 시대에, 진짜 경쟁력은 **"AI를 썼느냐 안 썼느냐"**가 아니라 **"AI가 내놓는 확률적 환각과 아부를 어떤 결정론적 시스템(파이썬 검증기, 하드웨어 메모리 설계, MVCC 상태 원장, 멀티 에이전트 토너먼트)으로 통제하고 검증했는가"**에 있습니다.

이 5부작에 기록된 아키텍처와 검증 코드들은 실제 데스크톱(`RTX 3060 12GB`) 로컬 환경에서 매일 돌아가며 의사결정과 자동화를 수행하고 있는 살아있는 엔지니어링 자산입니다.
