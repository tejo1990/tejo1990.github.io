# 🛡️ Sovereign Local LLM & Deterministic Python Agent Architecture (`tejo1990.github.io`)

> **왜 '그냥 쓰는 클라우드 LLM'이나 '원터치 로컬 LLM(Ollama 기본 설치)'과 결과물의 차원이 다른가?**
> 소비자용 GPU(`RTX 3060 12GB`) 한 장의 물리적 한계와 확률적 언어 모델(LLM)의 환각·아부 편향을 극복하기 위해, **초기 5부작 검증 하네스에서 출발하여 `13.90 GB` 100% VRAM 4-슬롯 금고 · `64KB HWID` 제자리 헤더 잠금 · `458,270건` 5-스코프 오프라인 지식 카트리지 · `3-Tier` 구어체 쿼리 번역기 · `139종` 순수 파이썬 기호 외골격(`vault_stdlib.py`)**까지 단계별로 진화시킨 실전 AI 엔지니어링 기술 블로그입니다.

🌐 **웹사이트 바로가기 (인터랙티브 7단계 진화 타임라인 탑재)**: **[https://tejo1990.github.io](https://tejo1990.github.io)**

---

## 🧭 한눈에 보는 로컬 LLM 7단계 고도화 타임라인 (`Why -> How -> Metric`)

| 진화 단계 | 마주친 현실 병목 (발전 이유) | 핵심 엔지니어링 해법 | 검증된 정량 지표 |
| :--- | :--- | :--- | :--- |
| **Stage 1** | 클라우드/원터치 LLM의 아부 편향·암산 환각·기억 증발 | `De-anchoring` + 순수 파이썬 AST 게이트 + `Dual-SSOT MVCC` | 토큰 **`99.76%` 절감** · 토너먼트 **`99.88/100`** |
| **Stage 2** | `44GB` 스와핑 속도 급락 & 윈도우 가상메모리(Commit) `Vulkan OOM` 크래시 | **`13.90 GB` 100% VRAM 4-슬롯**(`14B` 주력 / `4B` 폴백 / `VL-3B` OCR) + Commit 사전 차단 가드 | 주력 **`31~33 TPS`** · 폴백 **`71+ TPS`** · OOM **`0건`** |
| **Stage 3** | 로컬 `.gguf`/`.sqlite` 무단 복사 및 작업 관리자 CLI 인자·히스토리 노출 | **`64KB` 제자리(In-Place) `HWID` 헤더 잠금** + `Zero-Argv` 런처 + 히스토리 소거 | 잠금/해제 **`~41.5 ms`** · 추가 디스크 **`0 Byte`** |
| **Stage 4** | `14B` 가중치 암기 한계 및 질문 속 틀린 숫자 전제 동조 | **`458,270건` 5-스코프 `SQLite FTS5` 카트리지**(한·영 위키+법령+공학+의학+Vision OCR) & 숫자 모순 가드 | 브리지 검색 **`30,000x` 가속** · 도면 OCR **`~3.8초`** |
| **Stage 5** | 일상 구어체 검색 미스 & 실시간 타이핑 디바운스 GPU 큐 정체(`89.7초`) | **40선 구어체 시드(`<0.05ms`) + 프리웜 우선 캐시 + 타이핑 `?neural=0` 분리(`AbortController`)** | 구어체 적중 **`10/10`** · 응답 **`89.7s → 22.6s`** |
| **Stage 6** | 로컬 모델의 복합 수리 추론 한계 및 고정 `<think>` 지연 | **`Adaptive Thinking` 제어 + `139종` L3 파이썬 외골격(`vault_stdlib.py`) 2-Pass 자율 계산** | 검증 함수 **`29 → 139종`** · 계산 오차 **`0.0%`** |
| **Stage 7** | 타 PC/고객 배포 시 개인 문서(`Layer-B`) 혼입 위험 | **`--edition` 빌드 물리 격리(`Layer-A` 코어 전용 추출 + 25종 표준 입력 템플릿 생성)** | 개인 문서 유출 **`0건`** 보장 |

---

## 📚 마스터 시리즈 전체 목차 (`posts/` — 총 9부작)

1. **[1부: 왜 '그냥 쓰는 클라우드 LLM'과 '원터치 로컬 LLM'은 실무 의사결정에서 실패하는가?](posts/01-why-vanilla-llm-and-oneclick-local-fail.md)**
2. **[2부: 하드웨어 물리 한계 직시: '13.9GB 100% VRAM 직결 4-슬롯 금고'와 '가상메모리(Commit) OOM 가드'](posts/02-rtx3060-12gb-4slot-moe-hybrid-rag.md)**
3. **[3부: 확률적 언어 모델을 '결정론적 엔지니어링 도구'로 바꾸는 순수 파이썬(Pure Python) AST 검증 게이트](posts/03-deterministic-python-ast-verification-gate.md)**
4. **[4부: 단일 프롬프트의 한계를 넘는 '3-Way 서브에이전트 토너먼트': 250문항 AI 안전·편향 벤치마크 실증](posts/04-subagent-tournament-250-benchmark.md)**
5. **[5부: 기억을 잃지 않는 에이전트: `Dual-SSOT MVCC` 상태 원장과 멀티스레드 일일 자동화 레이더 구축기](posts/05-dual-ssot-mvcc-and-parallel-automation.md)**
6. **[6부: 온프레미스의 마지막 보안 구멍을 막다: '64KB 제자리(In-Place) HWID 헤더 잠금'과 'Zero-Argv 무인자 런처'](posts/06-zero-leak-64kb-header-lock-and-zero-argv.md)**
7. **[7부: 두뇌(추론)와 기억(지식)의 완전 분리: '45.8만 건 5-스코프 불변 지식 카트리지'와 오프라인 스캔도면 Vision OCR](posts/07-458k-5scope-cartridge-and-vision-ocr.md)**
8. **[8부: "두꺼비집이 내려가요"를 '배선용차단기 단락전류'로: 3-Tier 구어체 쿼리 번역기와 타이핑 디바운스 큐 병목 해결기](posts/08-3tier-neural-query-rewriter-and-debounce-isolation.md)**
9. **[9부: 로컬 LLM의 추론 한계를 넘는 법: 'Adaptive Thinking' 제어, '139종 L3 파이썬 기호 외골격(2-Pass 자율 계산)', 그리고 배포 에디션 위생 분리](posts/09-adaptive-thinking-l3-exoskeleton-and-edition-hygiene.md)**
