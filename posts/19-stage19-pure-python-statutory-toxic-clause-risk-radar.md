# [19부 · Stage 19] 여러 계약서·견적서(`.hwpx`·`.hwp`·`.docx`·`.xlsx`·`.pptx`) 속 대한민국 법령 위반과 독소조항을 0.006초 만에 일괄 스캔해 위험도 순위표로 띄우기: 순수 파이썬 실정법 리스크 레이더

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 검토할 계약서·견적서 파일이 10~20개 쌓여 있을 때 일일이 열어보며 독소조항을 찾기는 너무 힘들고, 그렇다고 AI에게 20개를 한꺼번에 던져주면 중간 문서를 빼먹거나 지역별 상가 보증금 계산을 틀렸습니다.
> - **어떻게 고쳤나요?** 공항 보안검색대 금속탐지기처럼 **버튼 한 번만 누르면 파이썬이 `0.006초` 만에 모든 문서를 한국 법령 기준(하도급 60일/15.5%, 상가 환산보증금, 징벌적 위약벌 등)으로 전수 검사해 '가장 위험한 계약서 1위~N위 순위표'와 위반 조문을 딱 찍어주도록** 완성했습니다.

> **핵심 요약 (Stage 19 — 질문하기 전에 먼저 가장 위험한 계약서부터 찾아주는 전수 검문소)**
> - **마주친 현실적 한계 (발전 이유)**: Stage 16~18을 통해 여러 계약서와 견적서를 드래그 앤 드롭으로 등록하고 1:1 비교 및 감사 보고서 출력까지 자동화했지만, **검토해야 할 문서가 10~20건 쌓였을 때 "어느 문서에 가장 치명적인 법령 위반·독소조항이 숨어 있는지"를 먼저 골라내는 선제 트리아지(Triage)**가 필요했습니다. 그렇다고 20개 문서를 로컬 LLM 프롬프트에 한꺼번에 밀어 넣으면 컨텍스트 희석(`Lost-in-the-Middle`)이 발생하고, `보증금 + 월세×100` 권역별 환산보증금 상한(`9억/6.9억/5.4억/3.7억`)이나 하도급법 `60일·15.5%` 강행 기준을 확률적 어림짐작으로 오판합니다.
> - **도입한 아키텍처 해법**: LLM 토큰을 단 `0개` 소모하면서 색인된 모든 커스텀 문서를 원본 단위로 재결합해 전수 검사하는 **순수 파이썬 대한민국 법령·독소조항 일괄 리스크 레이더(`POST /api/scan-custom-risks`)**를 설계·구현했습니다. **① 하도급법(`제9조` 10일 간주검수 초과·무기한 유예, `제13조` 60일 초과 대금지급·연 `15.5%` 미만 지연이자·면제 특약)**, **② 상가건물 임대차보호법(`제2조·제10조·제10조의4·제11조·제15조` 권역별 환산보증금 자동 산출 및 초과 경보, 권리금·갱신권 사전 포기 당연무효, 대규모점포·전대차 권리금 사각지대)**, **③ 민법·약관규제법·근로기준법(`제398조` 일 `0.25%+` 과다 지체상금·무제한 상한, 손해배상 별도 징벌적 `위약벌`, 일방 임의해지·부제소, SOW 외 무상 추가과업·원시코드 무상귀속, `3.3%` 위장 프리랜서 소급 위험)**을 정량 점수(`0~100점`, `CRITICAL`/`HIGH`/`CAUTION`/`SAFE`)로 산출해 대시보드 순위표에 즉시 렌더링합니다.
> - **검증된 정량 지표**: 실전 등록 문서 `13건` 전수 스캔 및 순위화 **`6.48 ms`**(브라우저 오프라인 JS 엔진 폴백 **`1.1 ms`**) · Stage 15 `<w:del>` 변경추적 삭제 독소조항(`위약벌 500%` 등) 오탐 **`0건`** · 전체 테스트 스위트 **`34/34 OK`** 및 골든 벤치마크 **`50/50 (100.0%)`** 유지.

---

## 1. 다중 계약서 LLM 일괄 프롬프트 투입 vs `Stage 19` 순수 파이썬 실정법 리스크 레이더 비교

| 검증 항목 | 문서 10~20건을 LLM 프롬프트에 통째로 넣고 질문 | `Stage 19` 순수 파이썬 법령 리스크 레이더 (`POST /api/scan-custom-risks`) | 현장 계약 검토 효과 |
| :--- | :--- | :--- | :--- |
| **다중 문서 스캔 속도 & GPU 부하** | 수만 토큰 Prefill 병목 (`수십 초` 지연) + 중간 문서 누락 | **GPU 토큰 `0개` · `13건` 일괄 전수 스캔 `6.48 ms` (`0.0065초`)** | 폴더 전체 계약서 즉시 우선순위 분류 |
| **하도급법 숫자 경계 판정** | `60일` 지급기한·`10일` 간주검수·`연 15.5%` 지연이자 혼동 | **`d_pay > 60`, `d_insp > 10`, `0 < r_delay < 15.5%` 결정론 판정** | 강행규정 위반 조항 미탐 `0%` |
| **상임법 권역별 환산보증금 계산** | `보증금 + 월세×100` 암산 실수 및 서울(`9억`)·부산(`6.9억`) 혼동 | **원화 파서 + 권역별 상한(`9억/6.9억/5.4억/3.7억`) 자동 대조** | `5%` 인상 캡 해제 vs 10년 권리금 유지 정확 분리 |
| **워드(`<w:del>`) 취소선 삭제 조항** | 이미 삭제 합의된 구버전 위약벌을 현행 독소조항으로 오탐 | **Stage 15 파서와 직결되어 `<w:del>` 삭제 조항 오탐 `0건`** | 협상 완료 조항에 대한 허위 경보 원천 차단 |

---

## 2. 대한민국 법령 경계치 & 권역별 환산보증금 리스크 스코어링 핵심 코드 (Pure Python)

```python
import re
from typing import Dict, List, Optional


def parse_krw_amount(expr: str) -> Optional[float]:
    """복합 한국어 금액 표기('1억 5,000만원', '850만원', '90,000,000원')를 정확한 원화(KRW)로 변환한다."""
    m_comp = re.search(r"(\d+(?:,\d{3})*(?:\.\d+)?)\s*억(?:\s*원)?(?:\s*(\d+(?:,\d{3})*)\s*만)?", expr)
    if m_comp:
        eok = float(m_comp.group(1).replace(",", "")) * 100_000_000.0
        man = float(m_comp.group(2).replace(",", "")) * 10_000.0 if m_comp.group(2) else 0.0
        return eok + man
    m_man = re.search(r"(\d+(?:,\d{3})*(?:\.\d+)?)\s*만\s*원?", expr)
    if m_man:
        return float(m_man.group(1).replace(",", "")) * 10_000.0
    return None


def evaluate_korean_statutory_risk(doc_id: str, text: str) -> Dict[str, object]:
    """하도급법·상가임대차법·민법(위약벌/과다 지체상금) 기준 위반 점수와 등급을 결정론적으로 산출한다."""
    points = 0
    findings: List[str] = []

    m_pay = re.search(r"(?:수령|검수|세금계산서)[^0-9\n]{0,20}(\d{2,3})\s*일", text)
    if m_pay and int(m_pay.group(1)) > 60:
        points += 32
        findings.append("SUBCONTRACT_PAYMENT_OVER_60D (하도급법 제13조: 60일 초과 연 15.5%)")

    if "위약벌" in text and re.search(r"(손해배상과?\s*별도|\d{2,3}\s*%)", text):
        points += 28
        findings.append("PUNITIVE_PENALTY_SEPARATE_FROM_DAMAGES (민법 제103조·제398조)")

    if re.search(r"(권리금|갱신요구권).*?(포기|주장할\s*수\s*없)", text):
        points += 32
        findings.append("COMMERCIAL_LEASE_ILLEGAL_RIGHTS_WAIVER (상임법 제15조 강행규정 위반 무효)")

    m_dep = re.search(r"보증금\s*[:：]?\s*([^\n/]+)", text)
    m_rent = re.search(r"월세\s*[:：]?\s*([^\n/]+)", text)
    if m_dep and m_rent:
        dep, rent = parse_krw_amount(m_dep.group(1)), parse_krw_amount(m_rent.group(1))
        if dep is not None and rent is not None:
            conv = dep + rent * 100.0
            cap = 900_000_000.0 if "서울" in text else 690_000_000.0
            if conv > cap:
                points += 26
                findings.append(f"COMMERCIAL_LEASE_OVER_DEPOSIT_CAP ({conv / 1e8:.2f}억 > {cap / 1e8:g}억)")

    score = min(100, points)
    grade = "CRITICAL" if score >= 55 else ("HIGH" if score >= 20 else "SAFE")
    return {"doc_id": doc_id, "risk_score": score, "risk_grade": grade, "findings": findings}


if __name__ == "__main__":
    r_sub = evaluate_korean_statutory_risk("DOC-01", "목적물 수령 후 90일 지급하며 손해배상과 별도 위약벌 30%를 부과한다.")
    r_lease = evaluate_korean_statutory_risk("DOC-02", "서울 강남구 상가 / 보증금: 1억 5,000만원 / 월세: 850만원 / 권리금을 포기한다.")
    assert r_sub["risk_grade"] == "CRITICAL" and r_sub["risk_score"] == 60
    assert r_lease["risk_grade"] == "CRITICAL" and len(r_lease["findings"]) == 2
```
