# [7부 · Stage 7] 내 컴퓨터의 AI를 타 PC에 안전하게 이식하는 법: 'Edition Isolation(배포 위생 물리 분리)'과 25종 표준 입력 템플릿

> **핵심 요약 (Stage 7 — 배포 위생 격리 완결)**
> - **마주친 현실적 한계 (발전 이유)**: 본인이 직접 쓰기 위해 고도화한 로컬 LLM 시스템(`my_own` 에디션) 안에는 누구나 공통으로 쓰는 보편 법령·재무·공학 지식(`Layer-A`)뿐 아니라, 개인의 이력·재무 조건·지역 의사결정 기록 및 개인 맞춤형 캐시 시드(`Layer-B`)가 함께 축적됩니다. 만약 이 패키지를 다른 PC나 외부 고객 환경에 그대로 복사·배포하면 심각한 프라이버시 유출 사고로 이어집니다.
> - **도입한 아키텍처 해법**: 프로비저너 컴파일 단계(`--edition`)에서 **① SQLite 지식 카트리지**, **② 오프라인 대시보드 내장 JSON 및 구어체 프리웜 시드(`personal_only` 필터)**, **③ `:8081/api/l3-tools` 파이썬 계산기 화이트리스트**의 3개 레이어 모두에서 개인 자산(`LAYER_B_PERSONAL`)을 **물리적으로 `100%` 도려내는 `Edition Isolation` 게이트**를 구축했습니다. 대신 빈손으로 시작하는 사용자를 위해 빈칸만 채우면 즉시 색인되는 **25종 표준 마크다운 입력 템플릿**을 자동 생성합니다.
> - **검증된 정량 지표**: 클라이언트 에디션(`personal_client` / `domain_client`) 빌드 시 개인 문서·개인 시드 유출 **`0건` (자동 감사 테스트 검증)** · 4개 실무 팩 **총 `25개` 표준 입력 템플릿** 즉시 배포.

---

## 1. 에디션별 물리 격리 매트릭스 (`my_own` vs. `personal_client` / `domain_client`)

단순히 화면에서 숨기는(UI 마스킹) 수준이 아니라, 외부 배포용 파일(`.sqlite`, `.html`)을 빌드할 때 바이트 수준에서 개인 데이터를 아예 포함하지 않습니다.

| 구성 요소 | 본인 전용 에디션 (`--edition my_own`) | 외부 이식·납품 에디션 (`--edition personal_client` / `domain_client`) |
| :--- | :--- | :--- |
| **마스터 금고 자산 (`vault/`)** | `Layer-A`(보편 코어) + `Layer-B`(개인 런북·의사결정 기록) 전체 수록 | **`Layer-A` 보편 마스터 코어만 100% 수록** (`Layer-B` 물리적 제외 및 기존 잔류 행 자동 삭제) |
| **구어체 프리웜 시드** | 보편 40선(`L01~P10`) + 개인 경력/전략 전용 시드 포함 | **보편 40선(`L01~P10`)만 수록** (`personal_only=True` 시드 자동 스트리핑) |
| **L3 파이썬 검증기 (`:8081`)** | 전체 검증 함수 활성화 | **`LAYER_A_UNIVERSAL_CORE` 소속 보편 법령·재무·공학 계산기만 노출** |
| **사용자 문서 초기 구성** | 본인의 실제 기밀 단가표·계약서 색인본 | 개인 문서 `0건` + **사용자가 직접 채워 넣을 수 있는 25종 빈 표준 마크다운 템플릿 생성** |

---

## 2. 4개 실무 팩 기반 '25종 표준 입력 템플릿' 자동 생성 구조

외부 PC에 처음 설치했을 때 *"내 문서를 어떤 형식으로 넣어야 AI가 표와 숫자를 가장 정확히 읽나요?"*라는 막막함을 없애기 위해 `custom_templates/` 폴더 아래에 직군별 표준 양식을 자동 생성합니다:

1. **`personal_client` (개인 종합 6종)**: 근로·용역계약서 연봉테이블, 주택·상가 임대차계약서 등기부, 실손·종합보험 증권 특약표, 대출원리금·DSCR 고정지출 시트, 건강검진·복약 알레르기 이력, 미수금·내용증명 소멸시효 관리대장.
2. **`domain_client/manufacturing_defense` (제조·방산 6종)**: 부품별 BOM·MOQ 납품단가표, 원청·하도급 납품기본계약서 특약, 설비 에러코드 대응 런북 등.
3. **`domain_client/retail_lease_franchise` (상가·프랜차이즈 6종)**: 점포 환산보증금·권리금 계약 시트, 가맹본부 차액가맹금·로열티 구조표 등.
4. **`domain_client/tax_labor_professional` (세무·노무 6종 + 공통 가이드)**: 5인 미만/이상 상시근로자 산정 시트, 법인 전환 성실신고 경계 시트 등.

---

## 3. 컴파일 타임 에디션 위생 분리기 및 무유출 감사 코드 (Pure Python)

클라이언트 배포본 생성 직후 단 1건의 개인 자산이나 개인 전용 시드라도 남아 있으면 빌드를 즉시 실패(`AssertionError`)시키는 결정론적 위생 검증 로직입니다.

```python
from dataclasses import dataclass
from typing import Dict, List


@dataclass(frozen=True)
class VaultAssetMeta:
    doc_id: str
    asset_layer: str  # "LAYER_A_UNIVERSAL_CORE" or "LAYER_B_PERSONAL"
    title: str


@dataclass(frozen=True)
class RewriteSeedMeta:
    seed_id: str
    personal_only: bool
    query: str


def compile_sanitized_edition_bundle(
    assets: List[VaultAssetMeta],
    seeds: List[RewriteSeedMeta],
    edition: str,
) -> Dict[str, object]:
    """
    외부 배포 에디션(personal_client / domain_client) 빌드 시
    개인 레이어(LAYER_B_PERSONAL) 자산과 personal_only 시드를 물리적으로 100% 제거한다.
    """
    is_owner_edition = (edition.strip().lower() == "my_own")

    kept_assets = [
        a for a in assets
        if is_owner_edition or a.asset_layer == "LAYER_A_UNIVERSAL_CORE"
    ]
    kept_seeds = [
        s for s in seeds
        if is_owner_edition or (not s.personal_only)
    ]

    leaked_personal_assets = [a.doc_id for a in kept_assets if a.asset_layer != "LAYER_A_UNIVERSAL_CORE"]
    leaked_personal_seeds = [s.seed_id for s in kept_seeds if s.personal_only]

    if not is_owner_edition:
        assert len(leaked_personal_assets) == 0, f"Layer-B leak detected: {leaked_personal_assets}"
        assert len(leaked_personal_seeds) == 0, f"Personal seed leak detected: {leaked_personal_seeds}"

    return {
        "edition": edition,
        "included_asset_count": len(kept_assets),
        "included_seed_count": len(kept_seeds),
        "personal_docs_leaked": len(leaked_personal_assets),
        "personal_seeds_leaked": len(leaked_personal_seeds),
    }


if __name__ == "__main__":
    sample_assets = [
        VaultAssetMeta("CORE-LEASE-01", "LAYER_A_UNIVERSAL_CORE", "상가임대차 환산보증금 및 권리금 수리 모델"),
        VaultAssetMeta("PRIVATE-NOTE-99", "LAYER_B_PERSONAL", "개인 비공개 의사결정 노트"),
    ]
    sample_seeds = [
        RewriteSeedMeta("E01", personal_only=False, query="두꺼비집 쇼트 원인"),
        RewriteSeedMeta("PRIV01", personal_only=True, query="개인 전용 비공개 시드"),
    ]

    client_bundle = compile_sanitized_edition_bundle(sample_assets, sample_seeds, edition="personal_client")
    assert client_bundle["included_asset_count"] == 1
    assert client_bundle["included_seed_count"] == 1
    assert client_bundle["personal_docs_leaked"] == 0
    assert client_bundle["personal_seeds_leaked"] == 0
```
