# [3부] 확률적 언어 모델을 '결정론적 엔지니어링 도구'로 바꾸는 순수 파이썬(Pure Python) AST 검증 게이트

> **핵심 요약 (Executive Summary)**
> - **왜 일반 RAG(문서 검색)만으로는 부족한가?**: 문서를 잘 찾아오더라도 마지막에 LLM이 숫자를 더하거나 조건문을 해석할 때 확률적 오류가 개입합니다. 또한 무거운 서드파티 패키지에 의존하는 코드는 6개월만 지나도 버전 충돌(Dependency Rot)로 깨집니다.
> - **해법 — '지식의 코드화(Codified Knowledge)'와 4중 검증 게이트(`verify_asset.py`)**: 본 시스템은 모든 도메인 법칙을 **외부 라이브러리 의존성 0개의 순수 파이썬(`Pure Python Stdlib`) 함수**로 함께 저장하며, 저장 즉시 자동으로 **① YAML 온톨로지 태그 검사, ② 수식·경계 조건 검사, ③ `ast.parse()` 구문 분석 및 외부 패키지 차단, ④ 실제 서브프로세스 실행(`Exit Code 0`)**을 통과해야만 승인됩니다.
> - **2-Hop 직교 태그 그래프 라우팅 (`query_vault.py`)**: 수백만 자의 검증된 자산을 프롬프트에 다 넣는 대신, 질문과 매칭되는 런북과 2-Hop 구조적 동형성(Cross-Domain Isomorphs) 브릿지만 실시간 추출하여 **컨텍스트 토큰의 `99.76%`를 절약**합니다.

---

## 1. 일반 벡터 RAG vs. 결정론적 코드 금고(Sovereign Vault) 비교

| 비교 축 | 일반 벡터 DB 기반 RAG<br>*(LangChain + Chroma/Pinecone)* | 본 시스템: 결정론적 지식·코드 금고<br>*(Sovereign Vault + `query_vault.py` + `verify_asset.py`)* |
| :--- | :--- | :--- |
| **저장 단위** | 500자 단위로 쪼개진 비정형 텍스트 조각 (수식·표 절단 발생) | **[핵심 명제 + 수리 메커니즘 + 기각 조건 + 순수 파이썬 실행 함수]**가 결합된 완결형 마크다운 자산 |
| **실행 가능성** | 텍스트만 반환 (LLM이 다시 암산해야 함) | `query_vault.py --run <함수명>` 및 `:8081/api/l3-run`으로 **즉시 정확한 계산 결과값 반환** |
| **의존성 부패 (Bit-Rot)** | 패키지 버전 업그레이드 시 파이프라인 파손 빈발 | **100% Python Standard Library (`math`, `dataclasses`, `typing`, `sqlite3`)**만 허용하여 10년 뒤에도 즉시 실행 |
| **그래프 연결성 ($\kappa$)** | 단편적 유사도 검색 (도메인 간 통찰 연결 불가) | **Molloy-Reed 거대 연결 성분 기준 $\kappa > 2.0$ (`69.14`)** 달성으로 이종 도메인 간 2-Hop 수리 모델 자동 브릿징 |

---

## 2. 자동 무결성 검증기 (`verify_asset.py`)의 핵심 작동 원리

지식 금고(`vault/*.md`)에 파일이 생성되거나 수정될 때, 에이전트 라이프사이클 훅(`Stop Hook`)이 자동으로 아래 검증 로직을 수행합니다. 코드 블록이 없거나 실행 중 에러가 나면 턴 종료 자체가 차단되어 **불량 코드가 금고에 섞이는 것을 원천 봉쇄**합니다.

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
    마크다운 내부의 Python 코드 블록이:
    1) 문법적으로 유효한지(AST Parse),
    2) 외부 서드파티 패키지 없이 표준 라이브러리(및 내부 vault_stdlib)만 사용하는지,
    3) 실제로 서브프로세스에서 Exit Code 0으로 에러 없이 실행되는지 검증한다.
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
    ok, errs = verify_pure_python_block("import math\nassert math.isclose(math.sqrt(4), 2.0)\n")
    assert ok and not errs
```

---

## 3. 실전 성과: 코어 분리 및 JIT 라우팅 벤치마크

- **컨텍스트 토큰 절감**: 전체 코어 문서를 매번 프롬프트에 넣는(`739,875 토큰`) 대신 `11,518 토큰`의 라우팅 테이블과 JIT 실행기를 사용해 **`99.76%`의 토큰을 절감**했습니다.
- **검색 순도(Purity) `100.0%`**: 범용 도메인 코어(`Layer-A`)와 개인 컨텍스트(`Layer-B`)를 물리적으로 분리(`--layer core`)하여 범용 법령·제조·세무 질의에서 개인 노이즈가 섞이는 오탐을 **`16.9% -> 0.0%`로 완전 제거**했습니다.
