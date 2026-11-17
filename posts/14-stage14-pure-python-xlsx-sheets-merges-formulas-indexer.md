# [14부 · Stage 14] `openpyxl`·`pandas` 없이 엑셀 견적서·내역서(`.xlsx`) 다중 시트·병합 셀·수식 값을 0.03초 만에 뚫다: 순수 파이썬 SpreadsheetML 파서와 품목코드 행 교차 방어

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 회사 견적서나 원가내역서는 거의 다 **엑셀(`.xlsx`)**로 되어 있는데, 맨 윗줄 합쳐진 제목 때문에 열 이름이 다 깨지거나 세로로 묶인 품목 분류가 빈칸이 되고, 수식(`=수량×단가`, `=SUM(...)`)이나 퍼센트·날짜가 이상한 숫자로 읽히는 문제가 있었습니다.
> - **어떻게 고쳤나요?** `openpyxl`이나 엑셀 프로그램 없이 파이썬 기본 내장 도구만 써서 **여러 시트 순서·합쳐진 제목과 2단 머리글·세로 병합 분류·수식 계산값까지 `0.031초` 만에 정확히 복원해 한글(`.hwp/.hwpx`)·제안서(`.pptx`)·견적서(`.xlsx`) 실무 3대장을 완성**했습니다.

> **핵심 요약 (Stage 14 — 한국 실무 문서 3대장 `.hwpx/.hwp` · `.pptx` · `.xlsx` 무의존성 오프라인 완전 정복)**
> - **마주친 현실적 한계 (발전 이유)**: 건설·제조·IT 구축 사업의 견적서, 원가산출내역서, BOM 단가표는 거의 100% **엑셀(`.xlsx`)**로 작성됩니다. 그러나 인터넷이 차단된 폐쇄망 PC에서는 `openpyxl`이나 `pandas` 설치가 불가능하며, 단순하게 XML 태그만 순서대로 읽으면 **① `sharedStrings.xml` 속 동아시아 후리가나(`<rPh>`) 루비 텍스트가 섞여 품목명이 오염되고**, **② `sheet10.xml`이 `sheet2.xml`보다 먼저 정렬되어 탭 이름과 데이터가 뒤바뀌며**, **③ 상단 전체 병합 제목(`A1:G1`), 2단 복합 헤더(`A2:A3` + `B2:C2`), 세로 병합 분류 셀(`A4:A18`) 때문에 컬럼명이 날아가거나 하위 행 분류가 빈칸이 되고**, **④ 수식 셀(`<f>`)에 캐시 값(`<v>`)이 생략되었거나 퍼센트(`0.155`)·날짜 일련번호(`46343`)가 원시 숫자로 노출되어 수치 검증이 깨지는 한계**가 있었습니다.
> - **도입한 아키텍처 해법**: 외부 패키지 의존성 `0개`, 오직 파이썬 표준 라이브러리(`zipfile`, `xml.etree.ElementTree`, `posixpath`, `ast`, `datetime`)만으로 **① `<rPh>` 후리가나 완전 제거 공유 문자열 추출기**, **② `xl/workbook.xml` + `.rels` 기반 실제 탭 순서 복원 및 `veryHidden` 내부 시트 격리기**, **③ `<mergeCells>` 전폭 배너(`[시트개요]`) 분리 · 2단 복합 헤더(`품목식별(품목명)`) 합성 · 세로 병합 분류 자동 전파기**, **④ 수식 캐시 `<v>` 정제 + `<v>` 미저장 수식의 `eval()` 없는 순수 `ast` 안전 계산 폴백(`SUM`/`AVERAGE`/`MIN`/`MAX`/`ROUND`/`+-*/`) + `styles.xml` 퍼센트·날짜 변환기**, **⑤ `EncryptedPackage` 암호 문서 및 구형 `BIFF8 .xls` 진단 가드**를 구현해 `SQLite FTS5`와 품목명·품목코드(`XLSX-INV-400`) 양방향 모순 감지기에 직결했습니다.
> - **검증된 정량 지표**: `10시트 × 15행`(2단 병합 헤더 + 세로 병합 분류 + 수식 합계 포함) `.xlsx` 파싱 **`31.51 ms`** · 후리가나 오염·시트 순서 역전 **`0건`** · `eval()` 호출 **`0회`** · 한글 품목명 및 영문 품목코드(`XLSX-INV-400`) 양방향 `NUMERIC_VALUE_CONTRADICTION` & `ENTITY_ATTRIBUTE_ROW_SWAP` 적발(`29/29` 전체 테스트 `OK`, `50/50` 골든 벤치마크 `100.0%` 유지).

---

## 1. 단순 XML 셀 읽기 vs 순수 파이썬 `.xlsx` 구조·수식 보존 파서(`Stage 14`) 비교

| 검증 항목 | 단순 XML 셀 순회 (기존 실패 패턴) | 순수 파이썬 `.xlsx` SpreadsheetML 파서 (`Stage 14`) | `SQLite FTS5` & 모순 감지기 연동 효과 |
| :--- | :--- | :--- | :--- |
| **`sharedStrings.xml` 추출** | `<si>` 하위 모든 `<t>` 결합으로 `<rPh>` 후리가나 노이즈가 품목명에 혼입 | `<t>` 및 `<r>/<t>`만 추출하고 `<rPh>`·`<phoneticPr>` 원천 배제 | 한자·외래어 포함 설비 품목명 오염 `0건` |
| **다중 시트 탭 순서 & 숨김** | `sheet10.xml < sheet2.xml` 사전순 정렬로 시트명↔본문 오매칭 및 숨김 시트 유출 | `xl/workbook.xml` + `workbook.xml.rels` 시각 탭 순서 매핑 & `veryHidden` 격리 | `10시트` 내역서 **`31.51 ms`** 해독 · 탭 불일치 `0건` |
| **병합 셀(`mergeCells`) & 헤더** | `A1:G1` 제목이 헤더를 덮어써 `열2, 열3`으로 깨지고 `A4:A18` 하위 분류 증발 | 전폭 배너(`[시트개요]`) 분리 + 2단 병합 헤더(`상위(하위)`) 합성 + 세로 병합 값 전파 | 모든 행 카드가 정확한 `분류·품목코드·품목명·단가` 보존 |
| **수식(`<f>`/`<v>`) & 서식(`styles.xml`)** | `42000000.00000001` 오차, `<v>` 생략 수식 공란, `0.155`·`46343` 원시 숫자 노출 | 부동소수점 정제 + 순수 `ast` 2-Pass 수식 폴백(`SUM` 등) + `15.5%`·`2026년 11월 17일` 변환 | 날짜 숫자의 금액 오인 차단 & 수식 합계 `100%` 검색 |
| **암호 파일 & 품목코드 경보** | 비밀번호 `.xlsx`에서 `BadZipFile` 크래시 · 품목코드 단독 질문 시 행 비교 누락 | `[MS-CFB]` `EncryptedPackage` / `BIFF8 .xls` 분기 안내 + 품목코드(`XLSX-INV-400`) 주어 인식 | 품목명·품목코드 양방향 **`ENTITY_ATTRIBUTE_ROW_SWAP`** 적발 |

---

## 2. 엑셀 병합 시트(`mergeCells`) 전폭 배너 분리 · 2단 복합 헤더 · 세로 분류 전파 직렬화기 (Pure Python)

```python
from typing import Dict, List, Tuple


def serialize_xlsx_merged_sheet_cards(
    sheet_name: str,
    n_rows: int,
    n_cols: int,
    sparse_cells: Dict[Tuple[int, int], str],
    merge_ranges: List[Tuple[int, int, int, int]],
) -> List[str]:
    """외부 패키지 없이 .xlsx 시트의 전폭 배너 분리, 2단 병합 복합 헤더 합성, 세로 병합 분류 전파를 수행한다."""
    grid = [[sparse_cells.get((r, c), "").strip() for c in range(n_cols)] for r in range(n_rows)]
    banner_rows = set()
    vert_merges: List[Tuple[int, int, int, int]] = []

    for r1, c1, r2, c2 in merge_ranges:
        anchor = grid[r1][c1] if (0 <= r1 < n_rows and 0 <= c1 < n_cols) else ""
        if not anchor:
            continue
        outside = any(grid[r1][k] for k in range(n_cols) if k < c1 or k > c2)
        if r1 == r2 and (c2 - c1 + 1) >= max(3, n_cols - 1) and not outside:
            banner_rows.add(r1)
            continue
        if r2 > r1:
            vert_merges.append((r1, c1, r2, c2))
        for rr in range(r1, min(r2 + 1, n_rows)):
            for cc in range(c1, min(c2 + 1, n_cols)):
                if not grid[rr][cc]:
                    grid[rr][cc] = anchor

    active_rows = [r for r in range(n_rows) if any(grid[r])]
    if len(active_rows) < 2:
        return []

    out_lines: List[str] = []
    pos = 0
    while pos < len(active_rows) - 1:
        rc = active_rows[pos]
        uniq = list(dict.fromkeys(v for v in grid[rc] if v))
        if rc in banner_rows or len(uniq) == 1:
            out_lines.append(f"- [시트개요] {uniq[0]}")
            pos += 1
        else:
            break

    hdr_r = active_rows[pos]
    data_rows = active_rows[pos + 1 :]
    if data_rows and data_rows[0] == hdr_r + 1 and any(r1 == hdr_r and r2 == hdr_r + 1 for r1, _, r2, _ in vert_merges):
        sub_r = hdr_r + 1
        data_rows = data_rows[1:]
        headers = [
            f"{grid[hdr_r][c]}({grid[sub_r][c]})"
            if (grid[hdr_r][c] and grid[sub_r][c] and grid[hdr_r][c] != grid[sub_r][c])
            else (grid[sub_r][c] or grid[hdr_r][c] or f"열{c + 1}")
            for c in range(n_cols)
        ]
    else:
        headers = [grid[hdr_r][c] or f"열{c + 1}" for c in range(n_cols)]

    for idx_r, r_d in enumerate(data_rows, start=1):
        pairs: List[str] = []
        last_kv: Tuple[str, str] = ("", "")
        for c_i, val in enumerate(grid[r_d]):
            if val and (headers[c_i], val) != last_kv:
                pairs.append(f"{headers[c_i]}: {val}")
                last_kv = (headers[c_i], val)
        if pairs:
            out_lines.append(f"- [{sheet_name} 행 #{idx_r}] " + " | ".join(pairs))
    return out_lines


if __name__ == "__main__":
    cards = serialize_xlsx_merged_sheet_cards(
        "세부원가내역",
        5,
        4,
        {
            (0, 0): "2026년 태양광·ESS 변전설비 구축 견적내역서",
            (1, 0): "설비분류",
            (1, 1): "품목식별",
            (1, 3): "공급가액",
            (2, 1): "품목코드",
            (2, 2): "품목명",
            (3, 0): "고효율전력변환군",
            (3, 1): "XLSX-INV-400",
            (3, 2): "태양광인버터",
            (3, 3): "42000000",
            (4, 1): "XLSX-ESS-900",
            (4, 2): "리튬인산철배터리팩",
            (4, 3): "88000000",
        },
        [(0, 0, 0, 3), (1, 0, 2, 0), (1, 1, 1, 2), (1, 3, 2, 3), (3, 0, 4, 0)],
    )
    assert cards[0] == "- [시트개요] 2026년 태양광·ESS 변전설비 구축 견적내역서"
    assert "설비분류: 고효율전력변환군 | 품목식별(품목코드): XLSX-ESS-900 | 품목식별(품목명): 리튬인산철배터리팩 | 공급가액: 88000000" in cards[2]
```
