# [7부] 두뇌(추론)와 기억(지식)의 완전 분리: '45.8만 건 5-스코프 불변 지식 카트리지'와 오프라인 스캔도면 Vision OCR

> **핵심 요약 (Executive Summary)**
> - **발전시켜 나간 이유**: `14B` 체급의 로컬 모델은 논리 독해력과 한국어 문장력은 뛰어나지만, 수십만 개의 법령 조항 번호·의학 상병코드(`KCD-8`)·약물 금기 조합·전력공학 절연 규격을 가중치 안에 정확히 암기하지 못합니다. 더 위험한 것은 사용자가 *"하도급 대금은 90일 안에만 주면 지연이자 안 붙지?"*처럼 **틀린 숫자가 포함된 질문(False Numeric Premise)**을 던지면 모델이 그 숫자를 그대로 받아들여 오답을 합리화한다는 점입니다.
> - **핵심 패러다임 전환 — '오픈북 외장 기억장치'**: 로컬 GPU(`Qwen3-14B`)에는 암기를 강요하지 않고 오직 **'주어진 원문 근거를 독해·대조·추론하는 두뇌'** 역할만 맡겼습니다. 대신 인류의 핵심 정전 지식과 내 문서를 **총 `458,270건` 규모의 단일 `SQLite FTS5` 불변 지식 카트리지(`:8081` 로컬 브리지)**로 외장화하여 매 질문마다 수십 밀리초 안에 정확한 원문 조항을 오픈북으로 펼쳐 줍니다.
> - **3대 엔지니어링 돌파**: **① `96,072개` 한-영 학술 표제어 양방향 교차 검색**, **② 외부 패키지 없는 `.xlsx`/`.docx`/`.pdf` 파서 + `Qwen2.5-VL-3B` 오프라인 스캔 도면 Vision OCR**, **③ `docsize` 메타 테이블 활용으로 검색 `30,000배` 가속 & 3-Layer 거짓 숫자 전제 모순 방어 게이트**를 완성했습니다.

---

## 1. `458,270건` 5-스코프 불변 지식 카트리지 구성표

단일 SQLite 파일 내부에서 도메인 오염 없이 원하는 영역만 즉시 필터링하거나 교차 검증(`내 계약서 + 대한민국 법령`)할 수 있도록 5개 논리 스코프로 설계했습니다.

| 논리 스코프 필터 | 수록 지식 및 출처 구성 | 문서 규모 | 해결하는 실무 문제 |
| :--- | :--- | :---: | :--- |
| **🏛️ 대한민국 법령**<br>(`STATUTE_ONLY`) | 민법·상법·근로기준법·하도급법·상가임대차법·개인정보보호법 등 핵심 조문 | 핵심 조문 전수 | 계약서 독소조항 대조, 소멸시효·간주검수·지연이자율 무환각 판정 |
| **⚙️ 공학·전산 표준**<br>(`TECH_STD_ONLY`) | 전력·변압기·제어계장(`4~20mA`, `RS-485`), 물리 상수, `CWE` 보안약점, `Python 3.12` 표준문서, Linux 매뉴얼 | **`9,913건`** | 폐쇄망 공장·연구소의 설비 규격 계산 및 오프라인 코딩 레퍼런스 |
| **🩺 의학·약리 표준**<br>(`MEDICAL_BIO_ONLY`) | `KCD-8` 한국표준질병사인분류, `WHO ATC` 약물분류, 병용금기·간/신독성 기준, 임상 정상수치표 | **`7,778건`** | 건강검진 수치 해석, 처방약 성분·병용금기 오프라인 즉시 대조 |
| **📚 한·영 학술 백과**<br>(`ENCYCLOPEDIA`) | 한국어 위키백과 실질문서 전수(`247,514건`) + 영문 위키 고순도 학술(`141,255건`) + **`96,072` 한-영 사전** | **`388,769건+`** | 한국어 미수록 최신 물리·수학·CS 이론까지 한국어 질문으로 영문 원문 교차 인출 |
| **📁 내 커스텀 문서**<br>(`CUSTOM_USER_DOCS`) | 순수 파이썬 `.xlsx`/`.docx`/`.pdf` 파서 + **`Qwen2.5-VL-3B` 스캔 도면·표 Vision OCR (`SHA-256` 캐시)** | 사용자 폴더 증분 색인 | 기밀 원가율 시트, 워드 내장 위약벌 표(`<w:tbl>`), 스캔 시험성적서 즉시 검색 |

---

## 2. 대용량 `SQLite FTS5`의 함정 해결: 왜 `30,000배` 속도 차이가 났는가?

문서 수가 3만 건에서 **45.8만 건**으로 늘어나자 초기 브리지 서버에서 두 가지 병목이 터졌습니다:
1. **`SELECT COUNT(*) FROM domain_docs_fts` 풀스캔 병목**: FTS5 가상 테이블에서 매 요청마다 `COUNT(*)`를 호출하면 수십만 행을 순회하느라 검색보다 카운트에 수 초가 허비되었습니다. 이를 내부 shadow 테이블(`domain_docs_fts_docsize`) `MAX(id)` 조회 + 메모리 캐싱으로 치환해 **상태 확인 지연을 `30,000배` 단축**했습니다.
2. **한국어 복합명사 미스 & 장문 질의 후미 절단**: 띄어쓰기가 제각각인 한국어 특성을 고려해 1차 형태소·동의어 매칭 실패 시 자동으로 **2단계 한국어 2-Gram(음절 바이그램) 폴백**이 작동하도록 했고, 복합 질문에서도 끝부분 조건이 잘리지 않도록 FTS5 쿼리 절 윈도우를 **48개 토큰**으로 확장했습니다.

---

## 3. 3-Layer 거짓 숫자 전제 모순 방어 게이트 (`False-Premise Numeric Guard`)

사용자가 *"하도급법상 목적물 수령 후 **90일** 이내에만 대금을 주면 지연이자가 안 붙죠?"*라고 잘못된 숫자를 전제로 물었을 때, 검색된 원문 조항(`60일 초과 시 연 15.5% 지연이자`)과의 숫자 불일치를 정규식·단위 기반으로 먼저 포착해 프롬프트 최상단에 **`[⚠️ 숫자 전제 모순 경보]`**를 강제 주입합니다.

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
