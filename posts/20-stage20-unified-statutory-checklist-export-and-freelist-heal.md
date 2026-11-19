# [20부 · Stage 20] 다중 문서 법령·독소조항 전수 점검표를 워드(`.docx` 제4절)·4-시트 수식 엑셀(`.xlsx`) 감사 보고서에 원클릭 통합하고, `2.6 GB` SQLite 프리리스트 헤더를 `<0.5 ms`에 자가 복구하기

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 19단계에서 위험한 계약서 순위표를 화면에 띄웠지만, 상사에게 보고서를 내려면 신·구 가격 비교표와 다중 계약서 독소조항 점검표가 **하나의 워드·엑셀 파일로 합쳐져야** 했고, 수십만 건 도서관 파일을 여러 창에서 동시에 다룰 때 목차 빈칸 숫자가 어긋나면 업데이트가 멈출 수 있었습니다.
> - **어떻게 고쳤나요?** 버튼 한 번에 **워드(`.docx`) 4장 '법령 위반 전수 점검표'와 엑셀(`.xlsx`) 4번 전용 시트(`=MAX` 최고 위험점수 · `=SUM` 총 배점 수식)까지 한 파일로 묶어 `0.008~0.018초` 만에 내려주고, 도서관 빈칸 목차가 어긋나도 `0.0005초` 만에 스스로 고치는 자가 치유 장치**를 달아 20단계를 완성했습니다.

> **핵심 요약 (Stage 20 — 사전 리스크 트리아지부터 결재 완제품 파일·DB 무결성 자가 치유까지 폐루프 완성)**
> - **마주친 현실적 한계 (발전 이유)**: Stage 19에서 등록된 계약서·견적서 13건을 `6.94 ms` 만에 대한민국 법령 기준으로 전수 스캔해 위험도 순위표로 띄웠지만, **현업 법무·구매 결재에서는 화면 순위표가 신·구 단가 비교표(Stage 17·18)와 별개로 놀지 않고 하나의 정식 감사 보고서(`.docx`/`.xlsx`) 안에 `'법령 위반·독소조항 전수 점검표'`로 통합 수록**되어야 했습니다. 또한 `2.6 GB`(`458,286건`) 카트리지에서 백그라운드 브리지와 CLI 관리 명령의 `64KB` 헤더 잠금 창이 교차할 때 SQLite 헤더 `36..40B`의 프리리스트 페이지 카운트가 실제 트렁크 체인 길이와 1~2페이지 어긋나면 `DELETE` 시 `database disk image is malformed` 예외가 발생할 수 있었습니다.
> - **도입한 아키텍처 해법**: **① `_normalize_audit_export_payload`에 Stage 19 리스크 레이더 자동 병합·전수 스캔(`include_risk_radar=True`) 및 `audit_sha256` 서명 봉인**, **② 워드(`.docx`)에 5열 리스크 색상 음영 표로 구성된 `제4절 법령 위반·독소조항 전수 점검표` 섹션 신설**, **③ 엑셀(`.xlsx`)에 최고 위험점수 `<f>MAX(E3:E...)</f>`와 개별 위반 배점 총합 `<f>SUM(F3:F...)</f>` 수식이 내장된 4번 전용 시트(`4_법령위반_독소조항점검표`) 및 1번 시트 `<f>SUM(C3:C8)</f>` 확장**, **④ `2.6 GB` 전체 `VACUUM` 없이 프리리스트 트렁크 페이지 `8B` 헤더 체인만 순회해 오프셋 `36..40B` 카운트를 즉시 정합화하는 `_sync_sqlite_freelist_header_count`(`< 0.5 ms`)**를 순수 파이썬 표준 라이브러리만으로 구현했습니다.
> - **검증된 정량 지표**: 통합 워드(`.docx`, `16.6 KB`) 생성 **`18.34 ms`** · 4-시트 수식 엑셀(`.xlsx`, `16.4 KB`) 생성 **`8.19 ms`** · 프리리스트 헤더 자가 치유 **`< 0.5 ms` (`PRAGMA quick_check == 'ok'`, `fts5_integrity: OK`)** · 전체 단위·E2E 테스트 **`35/35 OK`** 및 골든 벤치마크 **`50/50 (100.0%)`** 유지.

---

## 1. 개별 화면 수동 취합 & 전체 `VACUUM` 복구 vs `Stage 20` 통합 OOXML 컴파일러 & 프리리스트 자가 치유 비교

| 검증 항목 | 화면 순위표 수동 캡처 & 오류 시 `2.6 GB VACUUM` | `Stage 20` 통합 OOXML 익스포터 & 트렁크 체인 자가 복구 | 실무 결재 · 시스템 안정성 효과 |
| :--- | :--- | :--- | :--- |
| **워드(`.docx`) 법무 점검표 수록** | 신·구 단가표와 다중 계약 독소조항 표를 별도 수작업 편집 | **`제4절 법령 위반·독소조항 전수 점검표` 5열 리스크 음영 표 자동 합본 (`18.34 ms`)** | 단가 변동 + 실정법 위반 원클릭 품의 완결 |
| **엑셀(`.xlsx`) 전용 시트·수식** | 텍스트 나열 시 최고 위험점수·총 배점 집계 불가 | **`4_법령위반_독소조항점검표` 시트 + `<f>MAX(E3:E...)</f>` · `<f>SUM(F3:F...)</f>` (`8.19 ms`)** | Stage 14 역파싱 `100%` 무손실 및 엑셀 수식 즉시 연동 |
| **단독 점검표 다운로드 분리** | 기본 선택된 두 문서 비교가 섞여 파일명 혼선 발생 | **`scopeMode='radar'` 분리로 `Sovereign_Audit_Report_Statutory_Risk_Radar_Audit.*` 직출** | 리스크 레이더 단독 보고서와 1:1 비교 통합 보고서 완벽 분리 |
| **SQLite 프리리스트 헤더 정합성** | 헤더 `36..40B` 카운트 오차 시 `2.6 GB` 전체 `VACUUM`(`수십 초`) 필요 | **트렁크 체인 `8B` 포인터 순회 후 `4 Byte` 빅엔디안 정수 즉시 보정 (`< 0.5 ms`)** | `458,286건` 대용량 카트리지 무중단 업서트 보장 |

---

## 2. SQLite 프리리스트 트렁크 체인 `<0.5 ms` 자가 복구 & 4-시트 `MAX`/`SUM` 수식 생성 핵심 원리 (Pure Python)

```python
import io
import pathlib
import sqlite3
import struct
import tempfile
import xml.etree.ElementTree as ET
import zipfile


def heal_sqlite_freelist_header_count(db_path: pathlib.Path) -> int:
    """SQLite 헤더(오프셋 36..40B) 프리리스트 페이지 수를 실제 트렁크 체인 길이와 <0.5ms 내에 일치시킨다."""
    with open(db_path, "r+b") as f:
        hdr = f.read(100)
        if hdr[:16] != b"SQLite format 3\x00":
            return 0
        page_size = struct.unpack(">H", hdr[16:18])[0] or 65536
        if page_size == 1:
            page_size = 65536
        first_trunk, hdr_cnt = struct.unpack(">II", hdr[32:40])
        actual_cnt, trunk, seen = 0, first_trunk, set()
        while trunk != 0 and trunk not in seen:
            seen.add(trunk)
            f.seek((trunk - 1) * page_size)
            next_trunk, leaf_cnt = struct.unpack(">II", f.read(8))
            actual_cnt += 1 + leaf_cnt
            trunk = next_trunk
        if hdr_cnt != actual_cnt:
            f.seek(36)
            f.write(struct.pack(">I", actual_cnt))
            f.flush()
        return actual_cnt


def build_stage20_radar_sheet4_xml(scores_and_points: list[tuple[int, int]]) -> bytes:
    """최고 위험점수 MAX(E3:E...)와 개별 위반 배점 합계 SUM(F3:F...)이 동시 기록된 4번 시트를 생성한다."""
    rows = [
        f'<row r="{idx}"><c r="E{idx}"><v>{sc}</v></c><c r="F{idx}"><v>{pt}</v></c></row>'
        for idx, (sc, pt) in enumerate(scores_and_points, start=3)
    ]
    last_r = 2 + len(scores_and_points)
    max_sc = max(sc for sc, _ in scores_and_points)
    sum_pt = sum(pt for _, pt in scores_and_points)
    rows.append(
        f'<row r="{last_r + 1}">'
        f'<c r="E{last_r + 1}"><f>MAX(E3:E{last_r})</f><v>{max_sc}</v></c>'
        f'<c r="F{last_r + 1}"><f>SUM(F3:F{last_r})</f><v>{sum_pt}</v></c>'
        f"</row>"
    )
    xml_bytes = (
        '<?xml version="1.0" encoding="UTF-8"?>'
        '<worksheet xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main">'
        f"<sheetData>{''.join(rows)}</sheetData></worksheet>"
    ).encode("utf-8")
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w", compression=zipfile.ZIP_DEFLATED) as zf:
        zf.writestr("xl/worksheets/sheet4.xml", xml_bytes)
    return buf.getvalue()


if __name__ == "__main__":
    with tempfile.TemporaryDirectory() as td:
        p = pathlib.Path(td) / "sample.sqlite"
        c = sqlite3.connect(str(p))
        c.execute("CREATE TABLE demo (id INTEGER PRIMARY KEY, body TEXT);")
        c.executemany("INSERT INTO demo (body) VALUES (?);", [("A" * 900,) for _ in range(30)])
        c.commit()
        c.execute("DELETE FROM demo WHERE id <= 20;")
        c.commit()
        c.close()
        with open(p, "r+b") as fh:
            fh.seek(36)
            fh.write(struct.pack(">I", 999))
        heal_sqlite_freelist_header_count(p)
        c2 = sqlite3.connect(str(p))
        assert c2.execute("PRAGMA quick_check;").fetchone()[0] == "ok"
        c2.close()

    blob = build_stage20_radar_sheet4_xml([(100, 32), (58, 26)])
    with zipfile.ZipFile(io.BytesIO(blob), "r") as zf:
        s4 = zf.read("xl/worksheets/sheet4.xml")
        ET.fromstring(s4)
        assert b"<f>MAX(E3:E4)</f><v>100</v>" in s4 and b"<f>SUM(F3:F4)</f><v>58</v>" in s4
```
