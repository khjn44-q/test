# SSD 신뢰성 평가 플랫폼 — 대화 맥락 인수인계

작성일 2026-11-05. **이 문서는 사용자 요구와 설계 논의의 스냅샷이다. 실제 소스 감사 결과 또는 승인된 최종 설계가 아니다.** 다른 세션은 R2 ZIP, 최종 체크리스트, 실제 소스를 직접 읽고 재검토해야 한다.

## 1. 사용자가 설명한 현황

- SSD 신뢰성 평가 장비 A/B/C 3개 모델, 향후 증가 가능. C/GCC, Git 관리.
- 모델별 Vendor Original은 완전히 별도 소스. 비공개 구현 API, 전역변수·메모리·상수·구조체, API 조합 함수, 단순 Vendor Test Item이 있다. 같은 함수명도 인자·동작이 다를 수 있다.
- 별도로 Vendor를 직접 수정하고 자체 함수·Item·전역변수를 추가한 현재 Mixed Code가 있다. Vendor/Custom 경계가 붕괴하여 유지보수·전개가 어렵다.
- Vendor 코드는 빈번히 업데이트된다. 기존 코드 품질·가독성·결함 우려가 크지만 개별 결함은 실제 분석으로 확인해야 한다.
- 모델 세부타입(예: 92K-A/B), SSD 폼팩터별 `#define` 분기가 있다. 장비가 지원하는 NVMe/PCIe 규격 상한이 매크로에 존재하기도 한다.
- NVMe/PCIe 표준 외 SSD 개발사 내부 검증 Command Set, UART 명령/분석 로그, Chamber 온도, Voltage/Power, 장비 Utility API가 있다.
- 부적절한 변수 타입, overflow/underflow 위험이 존재한다. 정적분석과 향후 특화 분석 도구, LLM 기반 Item 생성·수정·재조합·평가 상태/로그 분석을 원한다.
- 원문 소스는 외부 ChatGPT에 제공할 수 없다. 사용자는 사내 LLM에 코드와 R2 ZIP, 최종 체크리스트를 첨부했다고 밝혔다.

## 2. 확정된 설계 원칙

**평가 정확성과 공통화는 둘 다 달성해야 한다.** 정확성을 희생한 공통화는 금지하지만, 어려운 부분도 공통화 대안을 충분히 검토한다. 양립 불가능한 실제 제약이 발견되면 **한계·원인·영향·대안·검증 근거를 문서화하고 작성자에게 명시적으로 보고한다.**

1. Vendor 원본은 모델·릴리스별 불변 보존. 원본 주석/포맷도 수정하지 않고 sidecar Docs 작성.
2. 현재 Mixed Code는 평가 의도·workaround·변경 계보·결함·회귀 비교의 참고 자료다. **현재 구조를 유지할 생각이 없으며 Custom 전체 Zero-base 재설계**가 목적.
3. Vendor 상위 조합 함수와 Item은 원칙적으로 참고만 한다. 변경 불가능한 최소 API 외에도 필수 ABI·전역·메모리·초기화·콜백·런타임을 실제 조사하여 좁은 실행 경계로 허용한다. 같은 Vendor 심볼 덮어쓰기나 1:1 복제 금지.
4. 공통 Item은 Vendor 헤더·심볼·모델 분기에 직접 의존하지 않는다. 장비별 차이는 Provider/Binding으로 격리.
5. 단일 Git 소스 트리에서 모델/세부타입/폼팩터/SDK의 유효 조합만 선택 빌드. 공통 코드에 `#ifdef` 확산 금지. ABI 차이는 빌드, 데이터 차이는 검증된 Profile, 동작 차이는 Provider, 평가 기준 차이는 Item Spec으로 분류하는 후보를 검토.
6. 모든 Custom 함수·구조체·변수·상수·파일에 전역 코딩/명명 규칙. 가독성·타입 안전·명시적 상태·소유권·단위·오류·Docs-First·LLM 개발 편의성 중시.
7. R2의 제안은 **정답이 아니라 소스 미열람 상태의 후보**다. 사내 LLM이 실제 원문·Git·빌드·규격을 전면 조사해 필요하면 폐기·대체한다. 중요한 미확정 사항은 담당자에게 질문하고 추측으로 확정하지 않는다.
8. 실장비 제어·파괴적 명령·온도/전압/Reset은 별도 승인 전 실행 금지. 사내 코드·인덱스·덤프·로그 외부 반출 금지.

## 3. 현재까지 제안한 구조 — 미승인 가설

```text
Test Items (C + metadata: 평가 목적/순서/판정)
  -> Evaluation Services (공통 조합/데이터 검증/상태)
  -> NVMe / PCIe / Internal Command / UART 순수 명세·코덱
  -> Narrow Capability Contracts + Runtime/Resource Management
  -> Model/SDK-specific Provider + Version Binding
  -> Immutable Vendor Primitive + Required ABI/Runtime

횡단: Docs/ADR, Source/Evidence Registry, Git provenance,
      Build Profiles, Static Analysis, Contract/HIL Tests, Structured Events.
```

아키텍처 후보:
- **A** 빌드별 공통 C 심볼을 구현하는 정적 Adapter. 단순할 때 우선.
- **B-lite** 빌드별 Provider 하나와 기능별 작은 `const ops` 테이블. Fake 주입·자원 조합 필요 시.
- **C** Vendor가 main/Item 콜백을 소유하면 Custom 라이브러리+Entry Shim.
- **D** SDK 전역 상태·ABI 충돌·장애 격리 근거가 있으면 Worker 프로세스.
- **E** Vendor의 하드웨어 시퀀스/트리거 엔진이 필요하면 활용.
- **F** xNVMe/libnvme/SPDK 등은 우선 구조 참고. 실제 SDK/큐/버퍼/장치 소유권·라이선스 검증 전 직접 도입 금지.
- R2의 **V1~V3**는 세부타입/폼팩터 변형 대안. 원문을 읽고 정의·선택.

빈번한 Vendor 업데이트는 모델 간 수평 차이와 SDK 릴리스 간 수직 차이를 모두 관리. **Vendor Release Registry + Version Binding + SDK Compatibility Matrix + Change Impact + Upgrade Runbook**을 핵심 운영 기능으로 권고. 함수 원형이 같아도 동작/초기화/메모리/timeout 의미가 바뀔 수 있다. 기존 Binding이 호환되면 재사용하고 실제 변화만 분리.

## 4. 반드시 분리할 도메인 계약

- **4종 데이터 표현**: Vendor ABI 타입(원형 보존), raw/wire 바이트·Dword, 규격 해석 모델, 평가 의미 모델. `packed struct`나 C 비트필드 캐스팅만으로 wire 해석 금지. 길이·endian·정렬·signedness·지원 개정·예약값·미확인 상태 명시.
- **NVMe**: SQE/CDW, CQE, Admin/I/O, Identify, Log Pages, Features, CID/queue, PRP/SGL, DMA/버퍼, timeout/cancel/reset, SDK 자동 재시도·분할·raw 필드 덮어쓰기. 세 모델이 CDW를 직접 구성 가능하다는 사용자 기억은 **실제 소스로 확인 필요**.
- **PCIe**: Config Space, Capabilities, MMIO, Link/LTSSM, AER, Reset, TLP/DLLP 관측·주입을 별도 Capability로 구분. Raw TLP/DLLP 제어 가능 여부 미확인.
- **내부 명령**: opcode 외 owner/dialect/revision/DUT FW 적용 범위·부작용·응답 구조·위험·검증 근거. 표준 NVMe 설명으로 내부 명령 의미 추측 금지.
- **UART**: 명령 응답과 비동기 로그 혼재, 프레임 분할·병합, Reset 후 stale response, 손실·시각 출처·세션 식별, 명령/로그 수신 경쟁.
- **Chamber/PSU**: setpoint ≠ readback ≠ stable, sensor freshness, rail/slot/fixture 공유, ramp, 보호, 물리 효과 시각. 안전 온도/전압을 임의 결정하지 않는다.
- **Utility**: 일반 문자열/수학/파일/시간은 표준 C 또는 공통 코드; DMA/장비 메모리·슬롯·하드웨어 시퀀스는 Provider에 격리.
- **상태/판정**: Execution State, DUT State, Evidence State, Evaluation Verdict 4축. 장비/전송 오류, DUT completion, decode validity, Item PASS/FAIL을 구별. 기대한 DUT 오류는 음성 평가 PASS 가능.
- **비동기 수명**: timeout 직후 DMA 버퍼 해제 금지. Reset 후 stale handle/callback, generation, Vendor global current_slot, shared chamber/rail, cleanup 실패를 검증.
- **재현성/로그**: Run/Item/Step/Request/Resource ID, Git commit, SDK/FW, DUT, spec profile, raw evidence, timestamp source·오차·손실 연결.

## 5. Custom C 코딩 규칙 제안

- 임시 prefix `se_`, lower_snake_case 함수·변수·파일, `SE_...` 상수/매크로. 프로젝트 접두사는 사내에서 확정. 파일명은 domain+responsibility (예: `se_nvme_health_decode.c`).
- 고정폭 wire 타입 `<stdint.h>`, 길이 `size_t` 등 목적에 맞게. overflow는 **계산 전에** 검사. underflow, narrowing, signed/unsigned, shift, 단위·주소 공간 변환 검증.
- 함수는 한 책임, 입력 버퍼+길이, `const`, 명시적 소유권·수명·전제/후조건·실패 후 출력 상태. mutable global 최소화. Vendor include/extern은 Provider 경계 밖 금지.
- `submit`≠`complete`, `set_target`≠`stable`, `cancel`≠`cancelled`, `read`≠`get_cached`.
- 주석은 왜/계약 ID/단위/부작용/수명/근거를 기술. 문서·헤더·스키마의 중복 정본 방지.
- C17은 후보이며 실제 GCC/SDK에 맞춰 결정. GCC warnings/-fanalyzer, clang-format, Clang LibTooling/clang-tidy, Cppcheck, Ctags, Doxygen, 필요 시 Coccinelle/CBMC/Frama-C/fuzzing. 정적분석 통과는 실제 장비 검증의 대체가 아니다.

## 6. Docs-First 정본과 추적성

- Source Manifest, Git lineage, build profile/compile_commands, 유효 Build Configuration Matrix.
- API/함수/전역·지역변수/타입/상수/매크로/메모리/파일/전처리 분기 인벤토리. Vendor Primitive/ABI/Runtime allowlist.
- Vendor Original vs Mixed Code의 확정/추정 Delta, 대표 Item의 실제 평가 목적·순서·상태·판정·workaround.
- NVMe/PCIe Field Schema, Internal Command Dialect Registry, UART Contract, Capability/Limitations/Non-equivalence Matrix.
- Port/Service/Item Specs, Coding/Naming Rules, ADR, `AGENTS.md`, Docs Index, Traceability, Static Analysis/Verification Evidence, SDK Compatibility/Upgrade Runbook.
- 사실마다 source location + symbol + hash + build profile + 근거 상태(source_observation/vendor_document/standard_reference/measurement/inference/UNKNOWN)를 연결. 원본에는 주석을 추가하지 않고 sidecar Docs 사용.
- LLM은 승인된 Item 명세→관련 계약→필드 schema→코드/시험을 읽고 제한된 범위 수정. 없는 함수·필드·Capability를 만들지 않고, 명세 변경·실제 검사·미실행 HIL을 보고.

## 7. 실행 계획과 게이트

이전 세션은 R2 ZIP에 **W00~W36, 총 37개 Work**와 세부 계획·템플릿이 있다고 설명했다. 새 세션은 **첨부 ZIP을 직접 열어 실제 파일/내용을 확인**해야 한다. 이 문서는 그 계획을 대체하지 않는다.

1. 원본/혼합 코드·Git commit·SDK 릴리스·해시·권한 및 안전 실행 범위 동결.
2. 모델/세부타입/폼팩터별 실제 GCC Translation Unit, `-D`, include 순서, linker, 함수·변수·상수·구조체·메모리·매크로 전체 인벤토리. 미해석 구성 표시.
3. Original vs Mixed 차이와 대표 Item의 평가 의도·상태·메모리·타이밍·숨은 workaround 복원.
4. Unknown/Question/Blocker를 실제 소스·내부 문서에서 먼저 조사. 중요 미확정은 작성자에게 질문, 임의 확정 금지.
5. A~F 및 V1~V3를 실제 소스에 대조해 최종 ADR·계약·빌드 구조 결정.
6. Coding Rules/Docs/Build/Fake-only 골격 → 순수 parser/command builder → 한 모델 Provider → 나머지 모델 → UART/환경/자원 → 대표 Item → 전체 Custom 재구현.
7. 모델별 GCC 빌드, 정적분석, 독립 Golden Vector, Fake Contract, 승인 HIL/타이밍 시험, LLM Item 변경 실험, Vendor Upgrade 리허설, 운영 전환.

각 Work는 목적·입력·대상 파일·구체 단계·산출물·검증 명령/실제 결과·반례·차단 조건·문의·종료 조건을 기록. 실제 장비 제어/파괴적 명령은 승인 전 실행 금지.

## 8. 마지막 필수 체크리스트

- [ ] 정확성과 공통화를 모두 검토했고 충돌 시 한계·원인·영향·대안·증거를 작성자에게 보고했는가?
- [ ] Vendor 원본과 Mixed Code의 Git/SDK 계보를 확인했는가?
- [ ] 모델·세부타입·폼팩터·SDK/FW·DUT의 유효 조합만 빌드·분석하는가?
- [ ] API/함수/변수/상수/타입/메모리/파일/매크로/조건부 분기의 실제 조사 범위와 미분석 영역을 기록했는가?
- [ ] 필수 Vendor Primitive/ABI/runtime와 재작성 상위 코드를 분리했는가?
- [ ] NVMe/PCIe 표준, 장비 지원 선언, 실측 검증 Capability를 구별했는가?
- [ ] 내부 명령 개정, UART 프레임/응답/로그 관계를 관리하는가?
- [ ] Chamber/Voltage 설정·측정·안정화, 공유 자원, 타이밍을 구분했는가?
- [ ] overflow/underflow, buffer lifetime, late callback, stale sensor, Reset, 잘린 응답 반례가 있는가?
- [ ] 명명/타입/레이어 의존성/정적분석 규칙을 자동 검사하는가?
- [ ] 실행/DUT/증거/판정 상태와 로그 출처를 분리했는가?
- [ ] LLM의 없는 API 생성·시험 정책 무단 변경·무승인 실장비 실행을 차단하는가?
- [ ] Vendor 업데이트 영향 분석과 호환성 검증이 가능한가?

## 9. 외부 참고 후보

CMake, GCC analyzer, Clang LibTooling/clang-tidy, Universal Ctags, Doxygen, Bear, Cppcheck, Coccinelle, Unity/CMock/Ceedling, CBMC, Frama-C, libFuzzer, libnvme, xNVMe, SPDK, OpenSpec, Spec Kit, Superpowers, Agent Skills. 실제 지원·라이선스·규격 이용 권한·사내 설치 적합성은 미검증. 이 도구들은 비공개 Vendor API의 실제 의미를 자동 확정하지 않는다. 공식 출처는 R2 패키지의 Source Registry를 참조.

## 10. 새 세션의 행동 원칙

사용자 요구와 과거 어시스턴트 제안을 구별한다. 이전 대화에서 이미 확인된 사실을 불필요하게 다시 묻지 않는다. 실제 원문을 읽지 않았으면 읽었다고 주장하지 않는다. 코드가 기존 가설을 반증하면 설계를 수정한다. 중요한 정보가 부족하면 확인할 파일·담당자·영향을 적어 질문한다. 문서/도구/레이어 수를 성과로 삼지 말고 **평가 정확성·공통화·유지보수·가독성·LLM 개발·검증 가능성**을 기준으로 판단한다.
