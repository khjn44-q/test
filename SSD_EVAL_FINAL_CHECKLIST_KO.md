# SSD 신뢰성 평가 플랫폼 R2 — 최종 보완·유의사항·체크리스트

**2026-11-03 | 세 번째 사내 LLM 인수인계 문서 | 실제 원문 미열람 / 설계 가이드**

## 0. 최상위 원칙

**평가 정확성과 코드 공통화는 모두 필수 목표다.** 공통화를 위해 평가 조건·명령 필드·시간·관측·판정을 조용히 약화하지 않는다. 반대로 정확성을 이유로 모델별 코드를 무분별하게 복제하지 않는다. 먼저 공통 의미 계약과 모델별 좁은 Provider/Capability로 **두 목표의 동시 달성**을 검토한다. 불가능하거나 아직 증명되지 않았다면 한계·영향·검토한 대안·증거를 기록하고 **작성자에게 명시적으로 통지**한다. 승인 전 해당 기능은 `BLOCKED/UNSUPPORTED/INCONCLUSIVE`로 분류한다. 대체 시험을 동일 Item으로 위장하지 않는다.

이 문서는 **정답이 아니다**. 사내 원문·Git·SDK·장비·규격을 실제로 검토한 담당자가 R2의 모든 가설을 반증 가능하게 재검토해야 한다. 기존 R2 전체 보고서·세부 작업계획과 함께 사용한다. 충돌은 `DOCUMENT_CONFLICT`로 등록한다. 미확인은 `UNKNOWN`, 미실행은 `NOT_RUN`으로 남긴다. 체크박스는 사전 완료를 뜻하지 않는다.

## 1. P0 — 소스와 빌드의 전면 감사

- [ ] A/B/C별 순수 Vendor 원본, 운영 혼합 코드, SDK 헤더·바이너리, Git commit/tag, 버전, 해시, 접근 범위를 확보했다.
- [ ] 혼합 코드가 출발한 Vendor 릴리스를 추적했다. 서로 다른 릴리스 차이를 사내 수정으로 오인하지 않았다.
- [ ] 전체 C/H 파일, 함수/API, 전역·지역변수·인자, 구조체/필드, enum/상수, 매크로, 메모리, 콜백, 함수 포인터, 생성 코드, 실행 진입점을 목록화했다.
- [ ] GCC·링커 버전, 타깃 ABI, 컴파일 옵션, include 순서, `#define`, 링커 스크립트, 실제 Translation Unit과 라이브러리를 프로필별로 기록했다.
- [ ] 비활성 `#if` 경로, AST 해석 실패, 간접 호출, 비공개 API 내부 등 분석 사각지대를 별도로 보고했다.
- [ ] Vendor 원본/운영 코드는 보존한다. 변경 불가능한 API·ABI·필수 전역/메모리·runtime과 참고용 상위 함수/Item을 구분했다.
- [ ] 오래된 Vendor 함수의 대기·재시도·보정·초기화가 실제 HW workaround인지 검증했다. 기존 코드도 정답으로 간주하지 않는다.

## 2. P0 — 모델·세부타입·폼팩터 구성 행렬

- [ ] 모델, 92K-A/B 같은 세부타입, HW revision, SDK/FW/FPGA, OS/ABI, 폼팩터, Fixture/Slot, DUT/FW를 별도 축으로 정의했다.
- [ ] 실제 유효한 조합만 `configuration_matrix`에 등록하고 불가능·미검증 조합을 구분했다.
- [ ] 모든 `#if/#define`의 활성 조건, 동작 차이, 근거, 신규 책임 위치를 기록했다.
- [ ] ABI/소스 차이는 빌드 선택, 배선·범위 차이는 Profile, 구현 차이는 Provider, 평가 기준 차이는 Item Policy로 분리 가능한지 검토했다.
- [ ] 빌드별 단일 모델/SDK/ABI를 고정하고 미선택 Vendor 코드가 컴파일·링크되지 않음을 확인했다.
- [ ] 분석하지 않은 매크로 조합을 PASS로 표시하지 않는다. 실행 전 실제 장비·Fixture·DUT와 빌드 프로필 일치를 검사한다.

## 3. P0 — NVMe·PCIe·내부 명령·UART·환경

- [ ] NVMe Base/Command Set/Transport, PCIe 개정·ECN, 내부 평가 정책, DUT FW 지원을 별도 버전으로 관리한다.
- [ ] Vendor 매크로의 **선언상 지원**, 실제 장비 **검증된 지원**, DUT 지원과 관측 가능성을 구분한다.
- [ ] SQE/CDW 입력·SDK 재작성 비트, Queue/CID/PRP/SGL·DMA 소유권, CQE 관측, 내부 재시도/분할·Timeout을 모델별 확인했다.
- [ ] PCIe Config/MMIO/Link/TLP/DLLP/LTSSM/AER/Reset의 제어 가능·관측 가능·미지원·미검증을 분리했다.
- [ ] SSD 개발사 내부 명령은 opcode만이 아니라 dialect/revision, 담당 조직, DUT/FW, 필드·응답·부작용·승인 근거를 명세했다. 미확인 개정을 임의 해석하지 않는다.
- [ ] UART의 명령 응답/비동기 로그 채널 공유, framing, 분할·합쳐짐·유실·순서 역전·Reset 후 늦은 응답·timestamp·correlation을 시험했다.
- [ ] Chamber/PSU는 목표값, 접수, 실측, 안정화, 센서 위치·교정·stale, 공유 Rail/Slot, 안전 한계를 분리했다.
- [ ] 일반 Utility와 장비 전용 Utility를 분류한다. 비정상 명령 평가에서도 호스트 메모리·설비 보호를 우회하지 않는다.

## 4. P0 — 타입·메모리·상태·평가 정확성

- [ ] Custom C/H 파일명·함수명·변수·타입·상수·단위·약어는 승인된 글로벌 코딩룰을 따른다. Vendor ABI 이름·레이아웃은 경계에서 보존한다.
- [ ] Wire raw, 규격 해석 모델, 평가 의미 모델, Vendor ABI 타입을 구분한다.
- [ ] `count*size`, `offset+length`, `count-1`, signed/unsigned, narrowing, shift, index, endian, alignment, 큰 카운터, 음수 온도의 경계값을 시험했다.
- [ ] `packed`/C bit-field만으로 wire layout을 확정하지 않는다. 독립 byte vector와 정적 layout 검사를 사용한다.
- [ ] CPU pointer, DMA/IOVA, MMIO, 장비 buffer ID의 주소 공간·소유권·수명·동기화·Reset 무효화를 명세했다.
- [ ] 전역 current_slot/CDW, SDK 재진입성, 콜백·스레드·lock, 공유 Chamber/Rail/Root Port의 영향 범위를 확인했다.
- [ ] Timeout은 취소 완료가 아니다. 늦은 Completion, 버퍼 조기 해제, Reset 뒤 stale handle을 명시적으로 다룬다.
- [ ] **Execution State / DUT State / Evidence State / Evaluation Verdict**를 별도 기록한다.
- [ ] 전송 실패, DUT Completion, 파싱 오류, 기대한 DUT 오류, 평가 FAIL을 혼동하지 않는다.
- [ ] 로그 유실·관측 불가·센서 stale·시계 차이·물리 효과 시각의 불확실성을 판정에 반영한다.
- [ ] Run ID, Git commit, SDK/장비 FW, subtype/폼팩터/Fixture, DUT FW, 규격/내부 명령 개정, Item revision, 설정/seed, 원시 근거와 파서 버전을 연결한다.
- [ ] A/B/C 동일 Item의 실제 자극·순서·시간·관측·판정 동등성을 검증한다. 같은 이름·빌드 성공만으로 동등성을 주장하지 않는다.
- [ ] 기존 Mixed Code의 PASS/FAIL은 참고 근거일 뿐 정답이 아니다. 동작 보존/결함 수정/정책 변경/최적화/폐기를 구분한다.

## 5. P1 — Docs-First·LLM·정적분석·Git

- [ ] `AGENTS.md`, 코딩룰, 용어사전, 문서 인덱스, Vendor sidecar, Port 계약, Field schema, Item 명세, ADR, 검증 기록의 정본·소유자·승인 상태를 정의했다.
- [ ] `요구 → 계약 → Custom 함수 → Provider → Vendor primitive → 시험 → 실행 이벤트 → 판정`을 안정적 ID로 추적한다.
- [ ] C 헤더 원형, 승인 필드 스키마, Markdown 설명, 생성물의 정본 책임을 나누고 중복 수동 편집을 피한다.
- [ ] LLM은 해당 Item/계약/필드/Capability부터 읽고 필요한 경우에만 원문을 확장 탐색한다. 미존재 API·필드·지원 기능 생성은 차단한다.
- [ ] 자연어 Item 수정은 명세 차이 → 영향 분석 → 반례 → Custom 코드 → 정적/단위/계약 검사 → 승인된 장비 검증으로 진행한다.
- [ ] GCC 경고·`-fanalyzer`, Cppcheck, 선택적 Clang 분석, 포맷/명명/레이어/문서 검사와 실제 분석한 프로필을 기록한다.
- [ ] 정적분석 통과를 장비 검증으로 표시하지 않는다. 독립 골든 벡터, Fake, Provider 계약, HIL·타이밍 검증을 구분한다.
- [ ] Git PR/커밋에 코드·명세·검증·영향 모델을 연결한다. Vendor 업데이트 시 변경 경계→Provider→Item→필요한 회귀 시험을 추적한다.
- [ ] 평가 현황/장애 요약은 구조화 이벤트와 증거에서 생성한다. LLM 요약이 원시 근거를 덮어쓰지 않는다.
- [ ] 대형 RAG/Graph DB/자체 정적분석 엔진은 기존 도구의 반복 실패와 실제 이득이 확인된 뒤에만 도입한다.

## 6. 작성자에게 확인할 필수 질문

| 질문 | 결정에 미치는 영향 |
|---|---|
| Vendor가 main/Item 등록·콜백을 소유하는가? | Vendor Host Shim vs 독립 실행 |
| 92K-A/B는 ABI 차이인가, 파라미터 차이인가? | 별도 Provider vs Profile |
| 폼팩터 분기가 배선·전원·타이밍·평가 정책 중 무엇을 바꾸는가? | 빌드/Profile/Item 분리 |
| 여러 DUT가 Chamber·Rail·Root Port를 공유하는가? | 자원·동시성 |
| SQE raw 비트는 실제 전송까지 보존되는가? | 음성 평가 지원 여부 |
| Timeout·Abort·Reset 뒤 요청·DMA 상태는? | 버퍼 수명·복구 |
| UART 명령/로그가 같은 채널인가? | 세션·프레임·관측 |
| 내부 명령 dialect/FW 매핑의 정본은? | 명령 Codec·판정 |
| Vendor 매크로 한계는 HW/SDK/예제 중 무엇인가? | Capability |
| 승인된 규격·시험 조건·설비 안전 한계의 소유자는? | 실행 승인 |
| 실제 중요 Item·기존 장애 사례·Git 수정 계보는? | 이전 순서·회귀 검증 |

**질문 원칙:** 먼저 사내 소스·Git·매뉴얼에서 찾고, 답에 따라 설계가 달라지는 질문만 작성자에게 한다. 이미 확인된 C/GCC 등은 반복 질문하지 않는다. 미답변은 `UNKNOWN`과 차단 범위로 기록한다.

## 7. 한계·비동등성·작성자 통지 템플릿

```yaml
limitation_id: LIMIT-TBD
status: OPEN
reported_to_author: false
reported_at: null
affected_models: []
affected_subtypes: []
affected_form_factors: []
affected_sdk_firmware_profiles: []
affected_items: []
required_common_behavior: null
observed_constraint: null
evidence_refs: []
impact_on_evaluation_accuracy: UNKNOWN
impact_on_commonality: UNKNOWN
alternatives_examined: []
recommended_option: null
why_full_equivalence_is_unproven: null
required_author_decision: null
approval_ref: null
verification_plan: []
```

**통지 내용:** 대상·차이·정확성 영향·공통화 대안·권고·요청 결정을 작성자에게 직접 보고한다. 문서 파일만 만들고 통지 완료로 표시하지 않는다. 미승인 범위의 실행·동등성 주장은 보류한다.

## 8. R2 작업계획에 추가할 승인 게이트

| Gate | 시점 | 증거 / 실패 시 조치 |
|---|---|---|
| G0 기준선 | W00~W02 | 원본·Git·SDK·빌드·미해석 범위 확인, 불명확하면 재조사 |
| G1 구성 행렬 | 전면 감사 | 모델×subtype×폼팩터×SDK 유효 조합 및 `#if` 매핑 |
| G2 계약 | 아키텍처 승인 전 | Vendor 최소 경계, 내부 명령·UART·메모리·타이밍 확인; 불명확하면 문의 |
| G3 정확성+공통화 | 첫 수직 Item | A/B/C 실제 자극·관측·판정 비교; 불일치 시 재설계 또는 한계 통지 |
| G4 품질 | Custom 구현 | 명명·타입·경고·정적분석·독립 반례·문서 정합성 |
| G5 실장비 | 승인된 HIL | 실제 장비·DUT·Fixture 프로필과 실행 승인·증거, 미실행은 NOT_RUN |
| G6 확장·운영 | 모델/Item 확대 | Git 계보·Vendor 업데이트 영향·지원/미지원·복구/rollback·작성자 승인 |

## 9. 사내 LLM의 최종 보고 형식

1. **실제 확인한 사실:** 파일·심볼·빌드 프로필·Git commit·증거 위치.
2. **설계 후보 비교:** 채택·탈락·보류 이유와 반증 조건.
3. **평가 정확성 검토:** 실제 자극·관측·판정의 모델별 동등성 및 미검증 범위.
4. **공통화 검토:** 공통 Core/Service/Item 비율과 모델별 Provider 차이, 측정 분모.
5. **한계 및 작성자 통지:** Limit ID, 영향, 대안, 필요한 결정, 통지 상태.
6. **검증 기록:** 실제 실행한 명령·프로필·결과와 미실행 Host/HIL/장기 시험.
7. **다음 작업:** 파일·계약·반례·완료 기준이 있는 가장 작은 검증 가능한 단위.

**최종 메시지:** “R2와 이 체크리스트는 실제 소스 확인 전의 후보 및 방법론이다. 먼저 전면 감사와 핵심 질문을 해결하고, 평가 정확성과 공통화 모두를 만족하는 설계를 선택하라. 불가능하거나 증거가 부족한 부분은 숨기지 말고 작성자에게 알려 승인받아라.”
