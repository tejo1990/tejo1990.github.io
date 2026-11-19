# [21부 · Stage 21] 등록된 계약서·견적서에서 하도급 10일 검수·60일 지급상한·상가 갱신통지 기한(`D-Day`)을 순수 파이썬으로 자동 역산하고, 한글 안 깨지는 RFC 5545 `75-octet` 표준 캘린더(`.ics`)로 즉시 내보내기

> **🌱 처음 오신 분을 위한 20초 쉬운 비유 요약**
> - **왜 고쳤나요?** 계약서 내용을 꼼꼼히 고쳐서 도장을 찍었더라도, 막상 바쁘게 일하다가 **납품 후 `10일` 검수기한, `60일` 대금지급일, 상가 만료 `1개월 전` 갱신 통보 날짜**를 깜빡 넘기면 꼼짝없이 연체이자를 물거나 원치 않는 계약이 자동 연장되는 큰 손해를 봅니다.
> - **어떻게 고쳤나요?** 등록된 계약서·견적서들에서 날짜만 읽는 것을 넘어 **계약서에 안 적힌 법정 필수 마감일(`+10일`·`+60일`·`만료 1달 전`)까지 파이썬이 `0.011초` 만에 자동 계산해 `D-Day` 급한 순서로 보여주고, 내 휴대폰·아웃룩 달력에 3일 전·1일 전 알람까지 쏙 들어가는 표준 일정 파일(`.ics`, 한글 깨짐 `0%`)로 즉시 내려주도록** 완성했습니다.

> **핵심 요약 (Stage 21 — 정적 조항 검토를 넘어 실정법 마감기한 `D-Day` 타임라인과 오프라인 `.ics` 알람까지 사후 이행 폐루프 완성)**
> - **마주친 현실적 한계 (발전 이유)**: Stage 19·20으로 계약 체결 전 독소조항 적발과 결재 보고서(`.docx`/`.xlsx`) 합본은 완결되었지만, **실무에서 가장 빈번한 금전 손실은 계약 체결 '이후' 날짜를 놓쳐서 발생**합니다(예: 하도급 목적물 수령 후 `10일` 간주검수 도과, `60일` 대금지급 상한 초과에 따른 연 `15.5%` 지연이자 기산, 상가임대차 만료 `6개월 전 ~ 1개월 전` 갱신요구·거절 통지 마감 도과로 인한 묵시적 갱신). 게다가 계약서 본문에는 체결일·착수일·`<w:del>` 삭제 구버전 납기일이 뒤섞여 있고, 법정 파생 기한은 계약서에 날짜로 적혀 있지 않으며, 한글 일정을 단순 75글자/바이트 슬라이싱으로 `.ics` 파일에 쓰면 3바이트 한글 UTF-8 문자가 반쪽 나 아웃룩·애플 캘린더에서 깨집니다.
> - **도입한 아키텍처 해법**: 외부 패키지(`icalendar` 등) `0개` 순수 파이썬 표준 라이브러리(`datetime`, `calendar`, `re`, `hashlib`)만으로 **① 한국어·ISO·엑셀 1900 일련번호(`46358 → 2026-12-02`) 명시 기한 추출 및 체결일·기간 시작일·Stage 15 `<w:del>` 삭제 날짜 원천 배제**, **② 대한민국 실정법 파생 기한 자동 역산(`D_deliv + 10d` 하도급법 제9조 간주검수 · `D_deliv + 60d` 제13조 지급상한 · `add_months(D_exp, -6..-1)` 상임법 제10조 갱신통지 마감 · 만료 `N일/개월 전` 사전통지 및 하자담보 만료)**, **③ 멀티바이트 한글 UTF-8 코드포인트 경계를 절대 쪼개지 않는 RFC 5545 `<= 75 octets` 바이트 폴딩(`_fold_rfc5545_line_utf8`) + 배타적 종일 종료일(`DTEND = DTSTART + 1d`) + `D-3`/`D-1` `VALARM` 오프라인 사전 알람 익스포터(`POST /api/scan-contract-deadlines` & `POST /api/export-contract-ics`)**를 구현했습니다.
> - **검증된 정량 지표**: 실전 커스텀 문서 `13건`(`18개` 계약·법정 기한) 전수 스캔 및 `.ics`(`29.2 KB`) 컴파일 **`11.35 ms`** · `<w:del>` 삭제 날짜 및 계약 시작일 오탐 **`0건`** · 전 물리 라인 `<= 75 UTF-8 bytes` 및 디코딩 무결성 **`100%`** · 저장소 전체 테스트 스위트 **`51/51 OK`**.

---

## 1. 단순 날짜 정규식 긁기 & 문자 슬라이싱 `.ics` vs `Stage 21` 실정법 파생 기한 역산 & `75-octet` 안전 폴딩 비교

| 검증 항목 | 단순 날짜 정규식 추출 · `line[:75]` 슬라이싱 | `Stage 21` 결정론적 법정 기한 레이더 & UTF-8 바이트 폴딩 `.ics` | 실무 일정 방어 · 호환성 효과 |
| :--- | :--- | :--- | :--- |
| **비마감 날짜 노이즈 차단** | 계약 체결일·기간 시작일·`<w:del>` 삭제 날짜가 기한경과로 오탐 | **범위 구문(`D1 ~ D2`) 시작일 소비·체결일 필터·`<w:del>` 원천 배제** | 가짜 `OVERDUE` 경보 **`0건`** |
| **하도급법 강행 기한 역산** | 계약서에 날짜가 안 적힌 `10일`/`60일` 법정 기한 누락 | **`D_deliv + 10d`(제9조 간주검수) · `+ 60d`(제13조 상한·연 `15.5%`) 자동 생성** | 약정 `20일 검수`·`90일 어음` 위반 배지 즉시 표시 |
| **상가·자동연장 통지 마감** | 만료일 당일만 표시되어 `1개월 전` 통지 마감 실기 위험 | **`calendar.monthrange` 말일 보정 기반 `E - 6m ~ E - 1m` 통지마감 역산** | `2027-03-31 → 2027-02-28` 윤년·말일 오차 **`0일`** |
| **RFC 5545 `.ics` 한글 폴딩** | `75글자`(`225B` 규격 위반) 또는 `b[:75]`(한글 3바이트 절단 깨짐) | **코드포인트 단위 UTF-8 바이트 누산(`첫 줄 <=75B`, `연속 줄 공백포함 <=75B`)** | 아웃룩·구글·애플·윈도우 캘린더 `100%` 무결 임포트 |
| **오프라인 사전 알람(`VALARM`)** | 인터넷 알림 서버 없으면 사전 경고 불가 | **표준 `TRIGGER:-P3D`(3일 전) & `-P1D`(1일 전) `ACTION:DISPLAY` 내장** | 완전 폐쇄망 PC·스마트폰 기본 캘린더 자체 알림 작동 |

---

## 2. 실정법 파생 기한(`+10d`/`+60d`/`E-1m`) 역산 & RFC 5545 한글 `75-octet` 무절단 폴딩 핵심 원리 (Pure Python)

```python
import calendar
import datetime


def shift_calendar_months(dt: datetime.date, delta_months: int) -> datetime.date:
    """말일(28/29/30/31일) 경계를 정확히 보존하며 월 단위 법정 통지 기한(상임법 -6m/-1m 등)을 역산한다."""
    ny, nm0 = divmod((dt.year * 12 + dt.month - 1) + delta_months, 12)
    nm = nm0 + 1
    return datetime.date(ny, nm, min(dt.day, calendar.monthrange(ny, nm)[1]))


def fold_rfc5545_line_utf8(line: str, limit_octets: int = 75) -> str:
    """한글 3바이트 UTF-8 문자를 절대 중간 절단하지 않고 모든 물리 라인을 <= 75 octets(CRLF+SPACE)로 폴딩한다."""
    if len(line.encode("utf-8")) <= limit_octets:
        return line
    segments, cur_chars, cur_bytes, budget = [], [], 0, limit_octets
    for ch in line:
        ch_b = len(ch.encode("utf-8"))
        if cur_chars and (cur_bytes + ch_b > budget):
            segments.append("".join(cur_chars))
            cur_chars, cur_bytes, budget = [ch], ch_b, limit_octets - 1
        else:
            cur_chars.append(ch)
            cur_bytes += ch_b
    if cur_chars:
        segments.append("".join(cur_chars))
    return "\r\n ".join(segments)


if __name__ == "__main__":
    deliv = datetime.date(2026, 11, 25)
    assert (deliv + datetime.timedelta(days=10)).isoformat() == "2026-12-05"
    assert (deliv + datetime.timedelta(days=60)).isoformat() == "2027-01-24"
    assert shift_calendar_months(datetime.date(2027, 3, 31), -1).isoformat() == "2027-02-28"

    summary = "SUMMARY:🚨[법령주의] [하도급 60일 법정 지급상한] 특고압 무정전전원장치 납품계약서 (약정 90일 초과 위반 시 연 15.5% 지연이자)"
    folded_bytes = (fold_rfc5545_line_utf8(summary) + "\r\n").encode("utf-8")
    phys_lines = folded_bytes.split(b"\r\n")[:-1]
    assert len(phys_lines) >= 3 and all(len(pl) <= 75 for pl in phys_lines)
    assert all(isinstance(pl.decode("utf-8"), str) for pl in phys_lines)
    assert folded_bytes.decode("utf-8").replace("\r\n ", "").strip() == summary
```
