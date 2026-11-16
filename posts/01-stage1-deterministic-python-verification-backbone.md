# [1부 · Stage 1] 왜 일반·원터치 로컬 LLM은 실패하는가: 질문 탈앵커링과 순수 파이썬(Pure Python) AST 검증 골격

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 일반 AI는 내가 틀린 생각을 말해도 *"좋은 생각입니다!"*라며 맞장구를 치고(아부), 세금·이자 암산을 자주 틀립니다.
> - **어떻게 고쳤나요?** 비유하자면 **'예스맨 비서'를 '깐깐한 팩트체크 비서 + 공학용 전자계산기'로 바꾼 첫걸음**입니다. 편향된 질문을 중립 질문으로 먼저 고쳐 잡고, 계산은 외부 패키지 없이 영구 작동하는 순수 파이썬 코드가 직접 수행합니다.

> **핵심 요약 (Stage 1 — 기초 검증 골격)**
> - **마주친 현실적 한계 (발전 이유)**: 기본 클라우드 LLM 웹채팅(ChatGPT/Claude/Gemini)은 사용자의 잘못된 가설에 동조하는 **아부 편향(Sycophancy)**과 확률적 토큰 생성으로 인한 **법령·재무·물리 암산 환각(Hallucination)**을 일으키며, 원터치 로컬 LLM 기본 설치(`ollama run`)는 무거운 모델에서의 스와핑 멈춤(`2.4 tok/s`) 또는 소형 모델에서의 추론 붕괴로 인해 실무 의사결정에 쓸 수 없었습니다. 또한 외부 서드파티 패키지에 의존하는 코드는 수개월 내 버전 충돌(Dependency Rot)로 깨집니다.
> - **도입한 아키텍처 해법**: LLM을 '정답 생성기'로 믿는 대신 **① 편향된 전제를 백지 비교 질문으로 바꾸는 질문 탈앵커링(Auto De-anchoring) 게이트**, **② 외부 라이브러리 의존성 0개의 순수 파이썬(`Pure Python Stdlib`) 결정론적 계산기**, **③ `ast.parse()` 구문 분석 및 서브프로세스 실행(`Exit Code 0`)을 통과한 코드만 금고에 적재하는 `verify_asset.py` 검증 게이트**를 기초 골격으로 세웠습니다.
> - **검증된 정량 지표**: 2-Hop 직교 태그 그래프 라우팅(`query_vault.py`)을 통해 전체 문서를 프롬프트에 넣는 방식 대비 **컨텍스트 토큰 `99.76%` 절감**(`739,875 → 11,518 tok`), 도메인 수리 계산 오차 **`0.0%`** 달성.

---

## 1. 프런티어 LLM의 '영리한 변호사 역설'과 일반 벡터 RAG의 한계

모델의 추론 능력이 올라갈수록 사용자가 잘못된 전제를 던졌을 때 그 전제를 정당화하는 논리를 더 정교하게 지어내는 **'영리한 변호사 역설(Clever-Lawyer Paradox)'**이 발생합니다. 또한 문서를 500자 단위로 잘라 넣는 일반 벡터 RAG는 표·법령 조항·수식이 절단되고 마지막 산술 계산을 여전히 LLM의 확률적 암산에 맡긴다는 치명적 한계가 있습니다.

| 비교 축 | 일반 클라우드 / 원터치 로컬 LLM + 기본 벡터 RAG | 본 시스템 (`Stage 1` 결정론적 지식·코드 검증 골격) |
| :--- | :--- | :--- |
| **전제 검증 태도** | *"좋은 접근입니다!"*라며 적자 계약·오류 가설에 동조 | 첫 줄에서 **`🔄 [질문 재구성]`** 선언 후 전제 오류 및 반례 먼저 제시 |
| **저장 단위** | 500자 단위로 쪼개진 비정형 텍스트 조각 (수식·조문 절단) | **[핵심 명제 + 수리 메커니즘 + 기각 조건 + 순수 파이썬 실행 함수]** 완결형 자산 |
| **의존성 수명** | `npm` / 무거운 서드파티 패키지 버전 충돌 시 파이프라인 파손 | **100% Python Standard Library (`math`, `dataclasses`, `sqlite3`)**만 허용 |
| **그래프 연결성 ($\kappa$)** | 단편적 유사도 검색 | **Molloy-Reed 기준 $\kappa > 2.0$ (`69.14`)** 달성으로 이종 도메인 간 2-Hop 브릿징 |

---

## 2. 실제 질문 대조 실증: 상가 임대차 계약 및 매장 수익성(DSCR) 감사

> **질문**: *"보증금 3,000만 원, 월세 150만 원, 인테리어 및 초도비 9,000만 원(대출 7,000만 원, 금리 6.5%) 조건으로 매장에 입점해 월매출 1,800만 원을 올리면 수익성이 괜찮을까요? 그리고 10년 채우면 권리금은 못 받게 되나요?"*

* ❌ **일반 LLM의 답변**: *"월매출 1,800만 원이면 안정적인 순수익이 기대됩니다! 다만 계약갱신요구권이 10년이므로 10년 후에는 권리금 회수가 어려울 수 있습니다."*
  - **법령 환각**: 개정 《상가건물 임대차보호법》 제10조의4에 의해 **10년 만료 후에도 권리금 회수기회 보호 의무는 유지**됩니다(대법원 판례 확립).
  - **재무 환각**: 인테리어 36개월 감가상각(월 250만 원)과 창업자 기회비용(월 300만 원), 원리금 상환액을 누락해 **실질 적자(`DSCR < 1.0`)** 계약을 흑자로 오판했습니다.

* ✅ **본 시스템의 결정론적 파이썬 검증 출력**:
  확률적 추측을 멈추고 검증된 순수 파이썬 함수를 실행하여 법정 보호 여부와 실질 현금흐름을 오차 없이 분리 판정합니다.

---

## 3. 자동 무결성 검증 게이트 (`verify_asset.py`) 핵심 코드 (Pure Python)

지식 금고에 자산이 추가·수정될 때마다 외부 패키지 임포트를 차단하고 실제 서브프로세스 런타임 실행(`Exit Code 0`)을 강제하는 검증기입니다.

```python
import ast
import os
import subprocess
import sys
import tempfile
from typing import List, Tuple

ALLOWED_STDLIB_MODULES = {
    "math", "dataclasses", "typing", "collections", "itertools",
    "functools", "statistics", "json", "re", "datetime", "enum",
    "hashlib", "sqlite3", "pathlib", "os", "sys", "vault_stdlib",
}


def verify_pure_python_block(code_str: str, timeout_sec: int = 10) -> Tuple[bool, List[str]]:
    """
    1) AST 문법 유효성 검사,
    2) 외부 서드파티 패키지 의존성 0개(표준 라이브러리 전용) 검사,
    3) 격리 서브프로세스 실제 실행(Exit Code 0) 통과 여부를 검증한다.
    """
    errors: List[str] = []
    try:
        tree = ast.parse(code_str)
    except SyntaxError as exc:
        return False, [f"AST SyntaxError at line {exc.lineno}: {exc.msg}"]

    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                root_pkg = alias.name.split(".")[0]
                if root_pkg not in ALLOWED_STDLIB_MODULES:
                    errors.append(f"Forbidden external dependency: import {root_pkg}")
        elif isinstance(node, ast.ImportFrom) and node.module:
            root_pkg = node.module.split(".")[0]
            if root_pkg not in ALLOWED_STDLIB_MODULES:
                errors.append(f"Forbidden external dependency: from {root_pkg} import ...")

    if errors:
        return False, errors

    with tempfile.NamedTemporaryFile("w", suffix=".py", delete=False, encoding="utf-8") as tmp:
        tmp.write(code_str)
        tmp_path = tmp.name

    try:
        proc = subprocess.run(
            [sys.executable, "-X", "utf8", tmp_path],
            capture_output=True,
            text=True,
            encoding="utf-8",
            timeout=timeout_sec,
        )
        if proc.returncode != 0:
            errors.append(f"Runtime execution failed (exit={proc.returncode}): {proc.stderr.strip()}")
    finally:
        if os.path.exists(tmp_path):
            os.remove(tmp_path)

    return len(errors) == 0, errors


if __name__ == "__main__":
    ok, errs = verify_pure_python_block("import math\nassert math.isclose(math.sqrt(9), 3.0)\n")
    assert ok and not errs
```
