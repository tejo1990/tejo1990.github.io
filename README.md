# 🛡️ Sovereign Offline AI Fortress (`tejo1990.github.io`)

> **소비자용 GPU(`RTX 3060 12GB`) 로컬 LLM은 어디까지 진화할 수 있는가?**
> 단순 원터치 로컬 설치(`Ollama` 기본값)의 메모리 초과·법령 환각·구어체 검색 실패·추론 병목을 하나씩 해결하며 고도화한 **로컬 LLM 7단계 직렬 진화 기록(`Stage 1 ~ 7` === `1부 ~ 7부` 완전 1:1 매칭)** 및 에이전트 하네스 별첨 연구 포트폴리오입니다.

🌐 **웹사이트 바로가기**: **[https://tejo1990.github.io](https://tejo1990.github.io)**

---

## 🧭 로컬 LLM 고도화 본편 7부작 (`Stage 1 ~ Stage 7` 1:1 직렬 매칭)

| 단계 | 포스트 (1:1 대응) | 마주친 현실 병목 (발전 이유) | 핵심 엔지니어링 해법 및 정량 성과 |
| :---: | :--- | :--- | :--- |
| **Stage 1** | **[1부: 질문 탈앵커링 & 순수 파이썬 AST 검증 골격](posts/01-stage1-deterministic-python-verification-backbone.md)** | 아부 편향·암산 환각·패키지 의존성 부패 | `De-anchoring` + `Pure Python AST` 게이트 · 토큰 **`99.76%` 절감** |
| **Stage 2** | **[2부: 13.9GB 100% VRAM 4-슬롯 금고 & 가상메모리 OOM 가드](posts/02-stage2-13gb-vram-vault-and-virtual-commit-guard.md)** | `44GB` 스와핑 속도 급락 & 윈도우 Commit `Vulkan OOM` | **`13.90 GB` 4-슬롯**(`14B` `31~33 TPS` / `4B` `71+ TPS`) + 상위 점유 앱 사전 차단 |
| **Stage 3** | **[3부: 64KB 제자리 HWID 헤더 잠금 & Zero-Argv 런처](posts/03-stage3-zero-leak-64kb-hwid-header-lock.md)** | `.gguf`/`.sqlite` 무단 USB 복사 및 작업 관리자 CLI 노출 | **`64KB` 제자리 헤더 잠금(`~41.5 ms`, 추가 용량 `0B`)** + 노출 인자 `0개` |
| **Stage 4** | **[4부: 45.8만 건 5-스코프 불변 지식 카트리지 & Vision OCR](posts/04-stage4-458k-5scope-cartridge-and-vision-ocr.md)** | `14B` 암기 한계 및 틀린 숫자 전제 동조 | **`458,270건` `SQLite FTS5`**(한·영 위키+법령+공학+의학+OCR) · 검색 **`30,000x` 가속** |
| **Stage 5** | **[5부: 3-Tier 구어체 쿼리 번역기 & 타이핑 디바운스 격리](posts/05-stage5-3tier-neural-rewriter-and-debounce-isolation.md)** | 일상 구어체 미스 & 타이핑 디바운스 GPU 큐 정체(`89.7초`) | 40선 시드(`<0.05ms`) + 자모 오염 차단 + `?neural=0` 분리(**`89.7s → 22.6s`**) |
| **Stage 6** | **[6부: Adaptive Thinking 제어 & 139종 L3 파이썬 외골격](posts/06-stage6-adaptive-thinking-and-139-l3-exoskeleton.md)** | 로컬 `14B`의 다단계 수리 추론 한계 및 고정 `<think>` 지연 | 난이도별 추론 점화 + **`139종 vault_stdlib.py` 2-Pass 자율 계산(오차 `0.0%`)** |
| **Stage 7** | **[7부: Edition Isolation 배포 위생 분리 & 25종 표준 템플릿](posts/07-stage7-edition-isolation-and-client-hygiene.md)** | 타 PC 이식·납품 시 개인 문서(`Layer-B`) 혼입 위험 | 빌드 시점 개인 레이어 **물리적 `100%` 소거(`0건` 유출)** + 25종 표준 템플릿 생성 |

---

## 📎 별첨 연구 노트: 에이전트 하네스 & 벤치마크 실험 (외전 2편)

* **[별첨 A: 단일 프롬프트의 한계를 넘는 '3-Way 서브에이전트 토너먼트' — 250문항 안전·편향 벤치마크 실증](posts/appendix-a-subagent-tournament-250-benchmark.md)**
* **[별첨 B: 멀티세션 Split-Brain을 막는 `Dual-SSOT MVCC` 상태 원장과 `ThreadPoolExecutor` 병렬 수집기](posts/appendix-b-dual-ssot-mvcc-and-parallel-radar.md)**
