# 검증 보고서 · 2026-11-02

## 결론

**59개 자동 테스트 통과, TypeScript strict 컴파일 통과.** 이것은 전체 REFS 제품의 완료 판정이 아니라 제한된 참조 알고리즘·Git reader·합성 수식·기계 계약·패키징 코드의 검증이다. 제품의 현재 상태는 Designed이며 G0~G10의 사내 구현/acceptance가 남아 있다.

## 실제 실행한 검증

| 범위 | 결과 | 의미 |
|---|---|---|
| TypeScript pure core | 24/24 PASS | RRF, 역방향 무효화, 증분 계획, all-of/deny helper, coverage, manifest 검사 |
| 실제 임시 Git reader | 11/11 PASS | dirty working tree와 commit 구분, Unicode/공백/대시 경로, CRLF byte mapping, hash/range, symlink, binary, 크기 제한 |
| 합성 한 노드 thermal fixture | 8/8 PASS | Euler step, 평형, 입력 범위, scalar stability guard, 온도차 계산 |
| JSON Schema·SQL 계약 | 13/13 PASS | 예시 payload, enum/필드, OID format, FK, coverage bound, 단일 writer 제약 |
| source-link 조립 회귀 | 3/3 PASS | 기존 링크 보존, 반복 조립 idempotence, source range 확장 |
| TypeScript strict compilation | PASS | reference-core와 application port 선언 |
| 정적 산출물 검사 | PASS | 내부 링크/anchor, JSON/YAML, fence, fixture 경로, R01~R44→T00~T13 연결 |

산출물 검사 횟수와 파일 수는 [artifact-checks.json](artifact-checks.json)에 기록된다. 이 수를 제품 테스트 수에 더하지 않는다. 문서 연결을 검사한 것이 semantic correctness나 Mermaid 렌더링을 증명하지는 않는다.

## 재실행 명령

승인된 Node·Git·TypeScript·Python 및 Python jsonschema/PyYAML/markdown-it-py가 있는 환경에서 패키지 루트를 기준으로 실행한다.

```sh
cd reference-core
npm test
cd ..
tsc contracts/ports.ts --strict --noEmit --target ES2022 --module NodeNext --moduleResolution NodeNext
python -m unittest discover -s examples/tests -v
python -m unittest discover -s verification -p 'test_*.py' -v
python research/assemble.py
python verification/check_artifacts.py
```

이 환경은 Linux, Node 22.16.0, TypeScript 5.8.3, Python 3.13.5, Git 2.47.3이다. 설치된 전역 TypeScript를 사용했으며 이 결과로 회사 Windows에서의 호환성을 주장하지 않는다. 테스트의 짧은 실행 시간은 제품 검색 지연 성능이 아니다.

## 로그

[Node 전체 결과](reference-suite.log), [수식 결과](thermal-green.log), [계약·패키징 전체 결과](contracts-and-packaging.log), [포트 컴파일](ports-build.log), [정적 패키지 검사](artifact-checks.log), [기계 요약](run_summary.json).

Red/Green 기록도 보존했다. core의 최초 red는 24개 중 20개 실패와 4개 통과(오류 발생 검사), Git reader는 11개 미구현 실패, thermal은 8개 미구현 오류, 패키지 조립은 3개 중 2개 실패였다. 최종 green에서는 해당 실패를 해결했다. 초기 red 로그가 남아 있다는 사실은 현재 suite 실패를 뜻하지 않는다.

## 수행하지 않은 것

실제 사내 저장소 clone/API 접속, 회사 계정 인증, Jira API 호출, 실제 SSD 모델/실측 데이터, QMD 설치·통합, embedding/reranker 모델 실행 및 사내 retrieval benchmark, Windows native 실행, 실제 OMP/Skill 활성화, 중앙 RLS/권한 회수 침투·운영 시험, Mermaid 렌더링은 수행하지 않았다. 이들의 완료를 위 테스트 결과로 대체하지 않는다.

공개 근거 36개는 공식 문서·공개 코드 읽기 수준의 조사다. 공개 main과 웹 캐시를 실제 승인 release lock으로 취급하지 않는다. `release-lock.example.json`의 null들은 미확정이며 운영 배포에 사용하면 안 된다.

## 자체 검토에서 수정한 사항

원문 출처 링크가 반복 조립에서 중복되는 결함을 재현하고 변환을 idempotent하게 수정했다. 회귀 테스트 3개를 추가했다. 중앙 서비스가 UI의 필수 선행 단계처럼 보이던 DAG를 수정해 개인형 G10 뒤 UI도 선택 가능하게 했다. ExecutionBridge 포트와 source encoding/fingerprint 필드를 보완해 설계와 계약을 맞췄다. coverage의 pending ID도 권한 없는 저장소 존재를 노출하지 않도록 명시했다.

독립 서브에이전트 검토를 실행했다고 주장하지 않는다. 구조·계약·요구사항은 자체 대조했고, 실제 실행한 검사는 위 로그로 분리했다.
