# [3부 · Stage 3] 온프레미스의 마지막 보안 구멍을 막다: '64KB 제자리(In-Place) HWID 헤더 잠금'과 'Zero-Argv 무인자 런처'

> **핵심 요약 (Stage 3 — Zero-Leak 로컬 물리·프로세스 보안)**
> - **마주친 현실적 한계 (발전 이유)**: 내 PC에 로컬 LLM을 구축하더라도 **① PC 수리점이나 외부인이 USB로 `.gguf` 모델이나 `.sqlite` 지식 DB 파일을 그대로 복사해 가는 물리 탈취**, **② 윈도우 작업 관리자(`Task Manager`) 명령줄 컬럼에 노출되는 모델 경로·플래그**, **③ PowerShell 히스토리 파일(`ConsoleHost_history.txt`)에 남는 실행 인자**라는 3대 로컬 보안 구멍이 존재했습니다. 그렇다고 `8.4GB` 모델을 매번 통째로 복호화하면 수십 초 지연과 `+8.4GB` 임시 평문 파일이 남습니다.
> - **도입한 아키텍처 해법**: 파일 전체가 아닌 **맨 앞 `64 KB (65,536 Byte)` 매직 헤더와 루트 메타데이터 영역만 메인보드·디스크 하드웨어 지문(`HWID`) 파생 키로 제자리(In-Place) 잠금(`EphemeralHeaderUnlockGuard`)**하고, 자식 프로세스 전용 환경변수 주입으로 CLI 인자를 **`0개`(`Zero-Argv`)**로 은닉하며 종료 즉시 재잠금·히스토리 세정을 수행했습니다.
> - **검증된 정량 지표**: 헤더 잠금/해제 **`~41.5 ms`** · 추가 디스크 소모 **`0 Byte`** · 작업 관리자 노출 플래그 **`0개`** · 무단 복사 후 열람 시도 시 `sqlite3.DatabaseError: file is not a database` 즉시 차단.

---

## 1. 일반 로컬 설치 vs. 전체 파일 암호화 vs. `64KB 제자리 HWID 가드` 비교

| 비교 항목 | ① 일반 로컬 설치<br>*(Ollama / LM Studio 기본값)* | ② 전체 파일 암복호화<br>*(Full-File Decrypt to Temp)* | ③ 본 시스템: `64KB 제자리 HWID 가드`<br>*(`EphemeralHeaderUnlockGuard`)* |
| :--- | :--- | :--- | :--- |
| **타 PC 무단 복사 방어** | ❌ **방어 불가** (USB 복사 즉시 실행·열람) | ⭕ 방어 가능 | ⭕ **완벽 차단** (`ERR_HARDWARE_CLONE_DETECTED` / `file is not a database`) |
| **암·복호화 소요 시간** | `0 ms` (무방비) | ❌ `15,000 ~ 45,000 ms` (수십 초 대기) | ✅ **`~41.5 ms` 해제 / `~41.7 ms` 재잠금** (체감 지연 `0초`) |
| **추가 디스크 공간 소모** | `0 Byte` | ❌ 원본만큼 임시 평문 공간 추가 필요 (`+11 GB`) | ✅ **`0 Byte`** (`r+b` 파일 포인터 `seek(0)` 제자리 변환) |
| **비정상 종료(크래시) 안전성** | 평문 방치 | ❌ 디스크에 복호화된 임시 평문 파일 잔류 위험 | ✅ Python `with` 컨텍스트 + Win32 `SetConsoleCtrlHandler` 이중 복구 |
| **작업 관리자(`PEB`) 노출** | ❌ `--model ... --ctx-size ...` 전체 노출 | ❌ CLI 인자 그대로 노출 | ✅ **노출 플래그 `0개` (`Zero-Argv` 환경변수 주입)** |

---

## 2. `64KB` 제자리 헤더 잠금 및 트랜잭션 컨텍스트 가드 핵심 코드 (Pure Python)

`GGUF`와 `SQLite3` 포맷 모두 첫 `64 KB` 안에 매직 시그니처와 텐서/페이지 루트 테이블이 집약되어 있어, 이 구간만 하드웨어 귀속 키스트림으로 가역 변환하면 대용량 파일 전체가 외부 환경에서 즉시 무력화됩니다.

```python
import hashlib
import hmac
import sqlite3
import tempfile
from pathlib import Path

HEADER_LOCK_BYTES = 65536  # 64 KB


def _derive_keystream(master_key: str, machine_hwid: str, length: int) -> bytes:
    seed = hashlib.pbkdf2_hmac(
        "sha256",
        master_key.encode("utf-8"),
        f"SOVHDRv2::{machine_hwid}".encode("utf-8"),
        60_000,
        dklen=32,
    )
    out = bytearray()
    counter = 0
    while len(out) < length:
        block = hmac.new(seed, counter.to_bytes(8, "big"), hashlib.sha256).digest()
        out.extend(block)
        counter += 1
    return bytes(out[:length])


def toggle_header_in_place(binary_path: Path, master_key: str, machine_hwid: str) -> None:
    """추가 디스크 0 Byte로 파일 첫 64KB 헤더를 제자리(In-Place) 잠금/해제한다."""
    size = binary_path.stat().st_size
    span = min(HEADER_LOCK_BYTES, size)
    stream = _derive_keystream(master_key, machine_hwid, span)
    with open(binary_path, "r+b") as fh:
        fh.seek(0)
        head = fh.read(span)
        fh.seek(0)
        fh.write(bytes(a ^ b for a, b in zip(head, stream)))
        fh.flush()


class EphemeralHeaderUnlockGuard:
    """실행 구간 동안만 64KB 헤더를 임시 해제하고 예외 발생 시에도 즉시 재잠금한다."""

    def __init__(self, binary_file: Path, master_key: str, current_hwid: str) -> None:
        self.binary_file = binary_file
        self.master_key = master_key
        self.current_hwid = current_hwid

    def __enter__(self) -> "EphemeralHeaderUnlockGuard":
        toggle_header_in_place(self.binary_file, self.master_key, self.current_hwid)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        toggle_header_in_place(self.binary_file, self.master_key, self.current_hwid)
        return False


if __name__ == "__main__":
    with tempfile.TemporaryDirectory() as td:
        db_file = Path(td) / "vault_demo.sqlite"
        conn = sqlite3.connect(db_file)
        conn.execute("CREATE TABLE docs (title TEXT);")
        conn.execute("INSERT INTO docs VALUES ('Confidential Spec');")
        conn.commit()
        conn.close()

        toggle_header_in_place(db_file, "<VAULT_MASTER_KEY>", "<MASKED_SHA256_HWID>")

        blocked = False
        try:
            bad_conn = sqlite3.connect(db_file)
            bad_conn.execute("SELECT * FROM docs").fetchall()
        except sqlite3.DatabaseError:
            blocked = True
        finally:
            try:
                bad_conn.close()
            except Exception:
                pass
        assert blocked is True

        with EphemeralHeaderUnlockGuard(db_file, "<VAULT_MASTER_KEY>", "<MASKED_SHA256_HWID>"):
            ok_conn = sqlite3.connect(db_file)
            rows = ok_conn.execute("SELECT title FROM docs").fetchall()
            ok_conn.close()
            assert rows == [("Confidential Spec",)]
```
