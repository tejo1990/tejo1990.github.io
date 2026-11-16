# [4부 · Stage 4] 두뇌(추론)와 기억(지식)의 완전 분리: '45.8만 건 5-스코프 불변 지식 카트리지'와 오프라인 스캔도면 Vision OCR

> **핵심 요약 (Stage 4 — 오픈북 지식 외장화 & 멀티모달 OCR)**
> - **마주친 현실적 한계 (발전 이유)**: `14B` 모델은 논리 독해력은 우수하지만 수십만 개의 법령 조문 번호·상병코드(`KCD-8`)·약물 병용금기·전력공학 절연 규격을 가중치 안에 무결점으로 암기하지 못합니다. 더구나 사용자가 *"하도급 대금은 90일 안에만 주면 이자 안 붙지?"*처럼 **틀린 숫자가 섞인 유도 질문**을 던지면 그대로 맞장구를 치는 위험이 있었습니다.
> - **도입한 아키텍처 해법**: 로컬 GPU(`Qwen3-14B`)는 오직 **'주어진 원문을 독해·추론하는 두뇌'**로만 한정하고, 정전 지식과 사용자 문서를 **총 `458,270건` 단일 `SQLite FTS5` 불변 지식 카트리지(`:8081` 로컬 브리지)**로 외장화했습니다. 스캔 도면과 시험성적서 이미지는 `Qwen2.5-VL-3B` 오프라인 Vision OCR로 판독해 통합 색인하고, 질문 숫자와 법령 원문 숫자의 충돌을 잡는 **3-Layer 거짓 숫자 전제 모순 가드**를 세웠습니다.
> - **검증된 정량 지표**: `domain_docs_fts_docsize` 메타 캐싱으로 브리지 검색 **`30,000배` 가속** · `96,072개` 한-영 학술 사전 교차 인출 · 스캔 도면 판독 **`~3.8초`**(해시 캐시 재조회 `0.01초`).

---

## 1. `458,270건` 5-스코프 불변 지식 카트리지 아키텍처

| 논리 스코프 필터 | 수록 지식 및 출처 구성 | 문서 규모 | 해결하는 실무 문제 |
| :--- | :--- | :---: | :--- |
| **🏛️ 대한민국 법령**<br>(`STATUTE_ONLY`) | 민법·상법·근로기준법·하도급법·상가임대차법·개인정보보호법 등 핵심 조문 | 핵심 조문 전수 | 계약서 독소조항 대조, 소멸시효·간주검수·지연이자율 무환각 판정 |
| **⚙️ 공학·전산 표준**<br>(`TECH_STD_ONLY`) | 전력·변압기·제어계장(`4~20mA`, `RS-485`), 물리 상수, `CWE` 보안약점, `Python 3.12` 표준문서, Linux 매뉴얼 | **`9,913건`** | 폐쇄망 공장·연구소의 설비 규격 계산 및 오프라인 코딩 레퍼런스 |
| **🩺 의학·약리 표준**<br>(`MEDICAL_BIO_ONLY`) | `KCD-8` 한국표준질병사인분류, `WHO ATC` 약물분류, 병용금기·간/신독성 기준, 임상 정상수치표 | **`7,778건`** | 건강검진 수치 해석, 처방약 성분·병용금기 오프라인 즉시 대조 |
| **📚 한·영 학술 백과**<br>(`ENCYCLOPEDIA`) | 한국어 위키백과 실질문서 전수(`247,514건`) + 영문 위키 고순도 학술(`141,255건`) + **`96,072` 한-영 사전** | **`388,769건+`** | 한국어 미수록 글로벌 물리·수학·CS 이론까지 한국어 질문으로 영문 원문 교차 인출 |
| **📁 내 커스텀 문서**<br>(`CUSTOM_USER_DOCS`) | 순수 파이썬 `.xlsx`/`.docx`/`.pdf` 파서 + **`Qwen2.5-VL-3B` 스캔 도면·표 Vision OCR (`SHA-256` 캐시)** | 사용자 폴더 증분 색인 | 기밀 원가율 시트, 워드 내장 위약벌 표(`<w:tbl>`), 스캔 시험성적서 즉시 검색 |

---

## 2. 3-Layer 거짓 숫자 전제 모순 방어 게이트 (`False-Premise Numeric Guard`) (Pure Python)

```python
import re
from typing import Dict, List

UNIT_PATTERN = re.compile(r"(\d+(?:\.\d+)?)\s*(일|개월|년|%|프로|시간|원|만원|억원|도|V|A|옴)")


def detect_numeric_premise_contradiction(
    user_query: str,
    retrieved_statute_text: str,
) -> Dict[str, object]:
    """
    질문 속 숫자+단위 전제와 검색된 원문 법령/규격의 동일 단위 수치가 충돌할 때
    LLM이 질문자의 틀린 숫자에 동조하지 못하도록 사전 모순 경보를 생성한다.
    """
    q_nums = [(val, unit) for val, unit in UNIT_PATTERN.findall(user_query)]
    doc_by_unit: Dict[str, List[str]] = {}
    for val, unit in UNIT_PATTERN.findall(retrieved_statute_text):
        doc_by_unit.setdefault(unit, []).append(val)

    mismatches = []
    for q_val, q_unit in q_nums:
        if q_unit in doc_by_unit and q_val not in doc_by_unit[q_unit]:
            mismatches.append(
                {
                    "unit": q_unit,
                    "query_value": f"{q_val}{q_unit}",
                    "authoritative_values": [f"{v}{q_unit}" for v in doc_by_unit[q_unit]],
                }
            )

    return {
        "contradiction_detected": len(mismatches) > 0,
        "mismatches": mismatches,
    }


if __name__ == "__main__":
    guard = detect_numeric_premise_contradiction(
        user_query="하도급 대금은 수령 후 90일 안에만 주면 지연이자 안 내도 되지?",
        retrieved_statute_text="하도급법 제13조: 원사업자는 목적물 수령일로부터 60일 이내에 대금을 지급해야 하며 초과 시 연 15.5% 지연이자를 지급한다.",
    )
    assert guard["contradiction_detected"] is True
    assert guard["mismatches"][0]["authoritative_values"] == ["60일"]
```
