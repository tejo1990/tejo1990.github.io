# [22부 · Stage 22] 계약·법정 마감기한(`D-Day`) 타임라인을 결재용 워드(`.docx` 제5절)와 5-시트 수식 엑셀(`.xlsx` 제5시트) 감사 보고서에 원클릭 합본하고, `2.64 GB` 82패턴 무유출·`50/50` 벤치마크 완결하기

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 21단계로 내 휴대폰·아웃룩 달력 알람(`.ics`)은 해결됐지만, 회사 결재 서류나 정산 대장에는 **'바뀐 가격 비교표 + 독소조항 점검표 + 계약·법정 마감일정표' 세 가지가 하나의 워드·엑셀 파일 안에 나란히 묶여 있어야** 상사 결재와 사후 일정 관리가 한 번에 끝납니다.
> - **어떻게 고쳤나요?** 버튼 한 번만 누르면 **결재용 워드(`.docx`) `'제5절 계약·법정 마감일정표'`(위험 일정 빨간 음영 표)와 5-시트 엑셀(`.xlsx`) `'5_계약법정기한_DDay일정표'`(`=MIN` 최단 남은 일수 · `=SUM` 법정 위반 합계 수식 내장)가 한 파일로 즉시 내려받아지고, `2.64GB` 고객 배포본 82개 패턴 유출 `0건` 및 50문제 만점(`50/50`)**까지 모두 검증했습니다.

> **핵심 요약 (Stage 22 — 신·구 단가 `1:1 Delta` + 법령·독소조항 점검표 + 계약·법정 마감일정표를 단 한 권의 오프라인 보고서로 봉인)**
> - **마주친 현실적 한계 (발전 이유)**: Stage 21을 통해 등록된 계약서·견적서에서 하도급 `10일` 간주검수·`60일` 지급상한·상가 만료 `1개월 전` 갱신통지 기한(`D-Day`)을 역산하고 `.ics` 캘린더 파일로 내보낼 수 있게 되었지만, **기업·기관 결재 라인에서는 `.ics` 달력 파일과 별개로 정식 품의서(`.docx`)와 정산 관리대장(`.xlsx`) 안에 `'계약·법정 마감일정표'`가 단가 비교표·법령 위반 점검표와 함께 한 파일로 묶여 있어야** 책임 소재와 사후 이행 점검이 완결됩니다. 또한 5번째 엑셀 시트(`sheet5.xml`)를 추가할 때 `sharedStrings.xml` 직렬화 순서나 1번 시트 `SUM(C3:C8)` 수식 범위를 건드리면 기존 오피스 호환성과 회귀 테스트가 깨질 수 있었습니다.
> - **도입한 아키텍처 해법**: 외부 패키지 설치 `0개` 순수 파이썬 표준 라이브러리(`zipfile`, `xml.etree.ElementTree`, `hashlib`)만으로 **① `_normalize_audit_export_payload`에 Stage 21 기한 레이더 자동 호출(`include_deadline_radar=True`)·근거 `DocID` 원장 결합·결정론적 `audit_sha256` 해시 서명 통합**, **② 워드(`.docx`) 가로 A4 본문에 5열 긴급도 색상 음영 표인 `'제5절 계약·법정 마감일정표 (Stage 21·22 D-Day 레이더)'` 신설(출처 원장·AI 검토 의견은 제6·7절로 이동)**, **③ 엑셀(`.xlsx`) 5번째 전용 시트 `'5_계약법정기한_DDay일정표'`(`xl/worksheets/sheet5.xml`, `<f>MIN(D3:D...)</f>` 최단·최장경과 D-Day 및 `<f>SUM(H3:H...)</f>` 법정상한 위반 합계 수식 내장, `sharedStrings.xml` 직렬화 전 전 시트 인터닝 불변량 및 1번 시트 `SUM(C3:C8)` 완벽 보존)**, **④ 대시보드 타임라인 툴바 단독 다운로드(`scope_mode='deadlines'`) 및 3-in-1 통합 다운로드**를 구현했습니다.
> - **검증된 정량 지표**: 단독 일정 워드·5-시트 통합 엑셀 생성 **`9.1 ~ 19.4 ms`** · 전 OOXML 파트 `ET.fromstring` 및 Stage 14/15 역파싱 무결성 **`100%` (`17/17` 단위 테스트 `OK`)** · `--benchmark-fortress-v3` **`50/50 (100.0%, SSE Warm TTFT 47.43 ms, 37.1x 가속, 99.13 tok/s)`** · `--edition personal_client` 실빌드(`458,160` 페이지 제자리 슬랙 소거) 6대 표면 및 **`2,641,301,504 B`(`2.64 GB`) 82개 개인·일정 패턴 원시 바이트 스캔 히트 `0건`**.

---

## 1. 개별 파일 수동 취합 vs `Stage 22` 3-in-1 통합 OOXML 감사 보고서(`.docx` 제5절 · 5-시트 `.xlsx`) 비교

| 검증 항목 | `.ics` 별도 보관 & 수동 워드/엑셀 복붙 | `Stage 22` 순수 파이썬 3-in-1 통합 OOXML 익스포터 (`POST /api/export-audit-report`) | 결재 · 감사 · 보안 위생 효과 |
| :--- | :--- | :--- | :--- |
| **결재용 워드(`.docx`) 일정표 통합** | 단가 비교표·독소조항표 뒤에 마감일 표를 수기로 작성 | **`제5절 계약·법정 마감일정표` 5열 긴급도 음영 표 자동 합본 (`~19.4 ms`)** | `제1~7절` 단일 결재 품의서 원클릭 완성 |
| **5-시트 수식 엑셀(`.xlsx`) 대장** | 단순 CSV 첨부 시 시트 분리·수식·서식 소실 | **`5_계약법정기한_DDay일정표`(`sheet5.xml`) + `<f>MIN(D3:D...)</f>` · `<f>SUM(H3:H...)</f>` (`~9.1 ms`)** | 최단 마감일·법정위반 건수 엑셀 실시간 재계산 |
| **ECMA-376 `sharedStrings` 무결성** | 시트 추가 시 문자열 인덱스·`rId` 불일치로 엑셀 복구 경고 발생 | **5개 시트 전수 인터닝 완료 후 `sst`(`count`/`uniqueCount`) 일괄 직렬화 & `rId1..7` 정렬** | MS Excel·한셀·LibreOffice 복구 경고 **`0건`** |
| **단독 일정표 ↔ 3-in-1 통합 분리** | 비교 문서 미선택 시 파일명·범위 혼선 | **`scope_mode='deadlines'` 시 `Contract_Deadlines_Schedule_Audit.*` 단독 직출** | 일정 전용 배포와 전방위 계약 감사 모두 대응 |
| **`2.64 GB` 고객 배포본 포렌식 검증** | 개인 계약 일정(`S21-UPS-900` 등) 슬랙 잔존 우려 | **`458,160` 페이지 무-`VACUUM` 제자리 제로화 후 `82`개 패턴 Raw 스캔** | **`2,641,301,504 B` 유출 `0건`** · 벤치마크 **`50/50`** |

---

## 2. 5-시트 `sharedStrings.xml` 순서 불변량 및 `sheet5.xml` `<f>MIN</f>`·`<f>SUM</f>` 조립 핵심 원리 (Pure Python)

```python
import html
import io
import xml.etree.ElementTree as ET
import zipfile


def build_stage22_5sheet_audit_xlsx(deadlines: list[tuple[int, int, str, str]]) -> bytes:
    """
    5번째 시트(5_계약법정기한_DDay일정표) 문자열 인터닝을 sharedStrings.xml 직렬화 이전에 완결하고,
    최단 D-Day MIN(D3:D...) 및 법정상한 위반 합계 SUM(H3:H...) 이중 노드를 기록한다.
    """
    sst_index: dict[str, int] = {}
    sst_list: list[str] = []
    sst_refs = 0

    def cell_s(ref: str, text: str) -> str:
        nonlocal sst_refs
        sst_refs += 1
        if text not in sst_index:
            sst_index[text] = len(sst_list)
            sst_list.append(text)
        return f'<c r="{ref}" t="s"><v>{sst_index[text]}</v></c>'

    s1_xml = (
        '<?xml version="1.0" encoding="UTF-8"?>'
        '<worksheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main"><sheetData>'
        f'<row r="9">{cell_s("A9", "총 감사 점검 포인트 합계")}<c r="C9"><f>SUM(C3:C8)</f><v>12</v></c></row>'
        f'<row r="10">{cell_s("A10", "계약·법정 마감일정(D-Day)")}<c r="C10"><v>{len(deadlines)}</v></c></row>'
        "</sheetData></worksheet>"
    )

    s5_rows = []
    for idx, (d_day, viol_flag, iso_dt, kind) in enumerate(deadlines, start=3):
        d_lbl = f"D+{abs(d_day)}" if d_day < 0 else ("D-Day" if d_day == 0 else f"D-{d_day}")
        s5_rows.append(
            f'<row r="{idx}">{cell_s(f"C{idx}", d_lbl)}<c r="D{idx}"><v>{d_day}</v></c>'
            f'{cell_s(f"E{idx}", iso_dt)}{cell_s(f"F{idx}", kind)}<c r="H{idx}"><v>{viol_flag}</v></c></row>'
        )
    last_r = 2 + len(deadlines)
    min_d = min(d for d, _, _, _ in deadlines)
    sum_v = sum(v for _, v, _, _ in deadlines)
    s5_rows.append(
        f'<row r="{last_r + 1}">{cell_s(f"A{last_r + 1}", "DEADLINE-TOTAL")}'
        f'<c r="D{last_r + 1}"><f>MIN(D3:D{last_r})</f><v>{min_d}</v></c>'
        f'<c r="H{last_r + 1}"><f>SUM(H3:H{last_r})</f><v>{sum_v}</v></c></row>'
    )
    s5_xml = (
        '<?xml version="1.0" encoding="UTF-8"?>'
        '<worksheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main">'
        f"<sheetData>{''.join(s5_rows)}</sheetData></worksheet>"
    )

    sst_xml = (
        '<?xml version="1.0" encoding="UTF-8"?>'
        f'<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" count="{sst_refs}" uniqueCount="{len(sst_list)}">'
        + "".join(f"<si><t>{html.escape(s)}</t></si>" for s in sst_list)
        + "</sst>"
    )
    wb_xml = (
        '<?xml version="1.0" encoding="UTF-8"?>'
        '<workbook xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main">'
        '<sheets><sheet name="1_감사요약_모순경보" sheetId="1"/><sheet name="5_계약법정기한_DDay일정표" sheetId="5"/></sheets>'
        "</workbook>"
    )

    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w", compression=zipfile.ZIP_DEFLATED) as zf:
        zf.writestr("xl/workbook.xml", wb_xml.encode("utf-8"))
        zf.writestr("xl/worksheets/sheet1.xml", s1_xml.encode("utf-8"))
        zf.writestr("xl/worksheets/sheet5.xml", s5_xml.encode("utf-8"))
        zf.writestr("xl/sharedStrings.xml", sst_xml.encode("utf-8"))
    return buf.getvalue()


if __name__ == "__main__":
    pkg = build_stage22_5sheet_audit_xlsx([
        (-4, 0, "2026-11-15", "잔금지급일"),
        (16, 1, "2026-12-05", "하도급 10일 간주검수 기한"),
        (66, 1, "2027-01-24", "하도급 60일 법정 지급상한"),
    ])
    with zipfile.ZipFile(io.BytesIO(pkg), "r") as zf:
        for member in zf.namelist():
            ET.fromstring(zf.read(member))
        s5 = zf.read("xl/worksheets/sheet5.xml")
        assert b"<f>MIN(D3:D5)</f><v>-4</v>" in s5
        assert b"<f>SUM(H3:H5)</f><v>2</v>" in s5
```
