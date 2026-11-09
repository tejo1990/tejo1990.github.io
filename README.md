# 🛡️ Sovereign Local LLM & Deterministic Python Agent Architecture (`tejo1990.github.io`)

> **왜 '그냥 쓰는 클라우드 LLM'이나 '원터치 로컬 LLM(Ollama 기본 설치)'과 결과물의 차원이 다른가?**
> 확률적 언어 모델(LLM)의 아부 편향(Sycophancy)과 계산 환각(Hallucination)을 극복하기 위해, **`RTX 3060 12GB` 4-슬롯 MoE 로컬 최적화 + 외부 의존성 0개의 순수 파이썬(`Pure Python`) AST 검증 게이트 + 3-Way 서브에이전트 토너먼트 + `Dual-SSOT MVCC` 상태 원장**으로 진화시킨 실전 AI 엔지니어링 기술 블로그입니다.

🌐 **웹사이트 바로가기**: **[https://tejo1990.github.io](https://tejo1990.github.io)**

---

## 📊 핵심 실측 벤치마크 요약 (Vanilla LLM vs. 본 시스템)

| 검증 지표 | 일반 클라우드 LLM / 원터치 로컬 LLM | 본 아키텍처 (`Sovereign Agentic Harness`) | 실증 근거 |
| :--- | :--- | :--- | :--- |
| **컨텍스트 토큰 소모량** | `739,875 tokens` (전체 문서 투입 시) | **`11,518 tokens` (`99.76%` 절감)** | `query_vault.py` 2-Hop 태그 그래프 JIT 라우팅 |
| **법령·재무·물리 계산 오차** | 확률적 암산으로 인한 빈번한 환각 | **`0.0%` (순수 파이썬 시뮬레이터 29종 강제 실행)** | `verify_asset.py` AST + 런타임 `Exit Code 0` 게이트 |
| **250문항 적대적 벤치마크** | `1,533자` 장황한 프롬프트 (토큰 효율 `10.04`) | **`538자` 탈앵커링 필터로 방어율 `100%` (효율 `28.60`, 종합 `99.88점`)** | 3-Way 서브에이전트 격리 토너먼트 실측 |
| **소비자용 GPU(`12GB`) 최적화** | 30B 모델 스와핑 멈춤(`2.4 tok/s`) 또는 3B 요약기 전락 | **`Qwen3-30B-A3B` (MoE) + `SQLite FTS5 + BGE-M3` (`총 44.0GB` 고정)** | 4-슬롯 `1-In, 1-Out` 롤링 볼트 |
| **기각된 오답 재발률** | 세션 종료 시 기억 증발로 동일 실수 반복 | **`0.0%` (기각된 가설 `125건` 영구 박제)** | `Dual-SSOT` SQLite MVCC 원장 (`v257`) |

---

## 📚 시리즈 전체 목차 (`posts/`)

1. **[1부: 왜 '그냥 쓰는 클라우드 LLM'과 '원터치 로컬 LLM'은 실무 의사결정에서 실패하는가?](posts/01-why-vanilla-llm-and-oneclick-local-fail.md)**
2. **[2부: 소비자용 GPU(`RTX 3060 12GB`) 한 장으로 끝내는 '4-슬롯 MoE 롤링 볼트' & 오프라인 하이브리드 RAG 설계](posts/02-rtx3060-12gb-4slot-moe-hybrid-rag.md)**
3. **[3부: 확률적 언어 모델을 '결정론적 엔지니어링 도구'로 바꾸는 순수 파이썬(Pure Python) AST 검증 게이트](posts/03-deterministic-python-ast-verification-gate.md)**
4. **[4부: 단일 프롬프트의 한계를 넘는 '3-Way 서브에이전트 토너먼트': 250문항 AI 안전·편향 벤치마크 실증](posts/04-subagent-tournament-250-benchmark.md)**
5. **[5부: 기억을 잃지 않는 에이전트: `Dual-SSOT MVCC` 상태 원장과 멀티스레드 일일 자동화 레이더 구축기](posts/05-dual-ssot-mvcc-and-parallel-automation.md)**
