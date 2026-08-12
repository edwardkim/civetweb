# Task #2 최종 결과보고서 — upstream 빌드 수정 패치를 fork에서 우선 유지하고 검증

GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
기존 fork PR: [#3](https://github.com/edwardkim/civetweb/pull/3)
마일스톤: M117 — CivetWeb v1.17 fork-maintained fixes and validation
구현계획서: [`task_m117_2_impl.md`](../plans/task_m117_2_impl.md)
단계 수: 4
source 기준: `upstream/master` commit `588860e30721bf5453b0440c390865a8e85dcae5`
source 결과: `local/task2` commit `cd59216490815d8548f87283e5738712c34cb090`

## 작업 요약

`civetweb/civetweb:master`의 `get_request()`에 남아 있던 손상된 조건식과 제거된 `cl` 변수 참조 때문에 Make와 Docker build가 실패하는 문제를 fork에서 우선 복구했다. 기존 upstream PR 네 개를 비교해 최소 source patch만 수동 병합하고 실질 기여자를 commit에 보존했다.

수정된 요청 프레이밍 동작은 네트워크 I/O 없는 private fixture와 여섯 회귀 경우로 고정했다. Make 기본·전체 기능 build, CMake/OpenSSL 3 build, CTest 50개와 Docker no-cache build를 통합 검증했다.

fork PR #3 게시 뒤 macOS matrix가 Homebrew에서 제거된 `openssl@1.1` 때문에 source compile 전에 실패하는 것을 확인했다. 이 오류를 해결하기 위한 별도 이슈나 별도 PR을 만들지 않고 Task #2와 기존 PR #3을 재개했다. macOS matrix를 OpenSSL 3으로 전환한 뒤 `push`와 `pull_request` CI build가 각각 13/13 job에 성공했다.

개인 운영 branch의 문서 push가 제품 CI를 실행하지 않도록 `personal/hyper-waterfall` 제외 조건을 fork 전용 branch에만 적용했다. 두 차례 실제 push 후 새 `CI build` run이 생성되지 않았으며, 이 filter는 source branch와 PR #3에 포함하지 않았다.

upstream 저장소에는 새 PR, issue comment 또는 review를 게시하지 않았다.

## 단계별 결과

| Stage | 결과 | source commit | 완료보고서 |
|---|---|---|---|
| 1 — 최소 컴파일 수정과 출처 보존 | `get_request()` compile 복구, 기본 Make build 성공 | `8629836d` | [`task_m117_2_stage1.md`](../working/task_m117_2_stage1.md) |
| 2 — 요청 프레이밍 회귀 테스트 | 독립 fixture와 6개 경우 추가, CTest 50/50 성공 | `783c1676` | [`task_m117_2_stage2.md`](../working/task_m117_2_stage2.md) |
| 3 — 통합 빌드 검증과 fork 게시 준비 | Make, CMake/CTest, Docker와 branch 범위 검증 성공 | source commit 없음 | [`task_m117_2_stage3.md`](../working/task_m117_2_stage3.md) |
| 4 — 기존 PR의 CI 실행 기반 복구 | macOS OpenSSL 3 전환, CI build 13/13 성공, 개인 branch 제외 확인 | `cd592164` | [`task_m117_2_stage4.md`](../working/task_m117_2_stage4.md) |

## 변경 파일 목록과 영향 범위

기존 PR #3의 source 변경:

| 파일 | 변경량 | 영향 |
|---|---:|---|
| `.github/workflows/cibuild.yml` | 7줄 추가, 49줄 삭제 | macOS job 두 개를 OpenSSL 3으로 통일하고 OpenSSL 1.1 전용 matrix와 setup 제거 |
| `src/civetweb.c` | 14줄 추가, 12줄 삭제 | `get_request()`의 header 변수, 조건식과 들여쓰기 복원. Zlib 비활성 build의 `h_zip` 미사용 경고 방지 |
| `unittest/private.c` | 90줄 추가 | `Transfer-Encoding`과 `Content-Length` 조합 6개에 대한 private 회귀 테스트 |

합계:

```text
3 files changed, 111 insertions(+), 61 deletions(-)
```

공개 API, 함수 signature, 요청 프레이밍 정책, 공식 사용자 문서와 Dockerfile은 변경하지 않았다. `unittest/public_server.c`, `unittest/CMakeLists.txt`, `src/handle_form.inl`, SSL 형 변환, Linux matrix, `fail-fast`와 CIFuzz workflow도 변경하지 않았다.

개인 운영 branch의 `.github/workflows/cibuild.yml`에는 위 OpenSSL 3 전환과 정확한 `personal/hyper-waterfall` push 제외 조건이 함께 있다. 개인 filter와 `mydocs/`는 PR #3 및 향후 upstream 기여 diff에 포함되지 않는다. 새 test 식별자, 구현과 수정한 workflow 주석은 영어로 작성했다.

## 변경 전·후 정량 비교

| 항목 | 변경 전 | 변경 후 |
|---|---|---|
| 기본 Make build | `get_request()` 조건식과 `cl` 참조 compile 오류 | exit code 0, 관련 오류·경고 없음 |
| 요청 프레이밍 전용 회귀 경우 | 없음 | 6개 |
| targeted HTTP message CTest | 수정 동작을 직접 고정하지 않음 | 1/1 통과 |
| 전체 CTest | source가 compile되지 않아 정상 실행 불가 | 50/50 통과 |
| 전체 기능 Make build | source compile 오류로 차단 | `WITH_ALL=1` 성공 |
| Docker build | source compile 오류로 차단 | no-cache build 성공 |
| macOS CI SSL | 제거된 `openssl@1.1` setup에서 실패 | OpenSSL 3 job 2/2 성공 |
| CI build `push` run | matrix fail-fast 실패 | 13/13 성공 |
| CI build `pull_request` run | matrix fail-fast 실패 | 13/13 성공 |
| 개인 문서 branch push | 전체 제품 CI 실행 | 새 `CI build` run 없음 |
| source 변경 파일 | 해당 없음 | 3개 |
| source commit | 해당 없음 | 3개 |

회귀 테스트는 다음 동작을 명시적으로 검증한다.

1. `Transfer-Encoding: chunked` 허용
2. `Transfer-Encoding: identity`와 유효한 `Content-Length` 허용
3. chunked와 `Content-Length` 동시 사용을 error 400으로 거부
4. 지원하지 않는 transfer encoding을 error 400으로 거부
5. 숫자가 아닌 `Content-Length`를 error 411로 거부
6. 음수 `Content-Length`를 error 411로 거부

## source commit과 저작자 보존

```text
cd592164 Task #2 Stage 4: macOS CI를 OpenSSL 3으로 전환
783c1676 Task #2 Stage 2: 요청 프레이밍 회귀 테스트 추가
8629836d Task #2 Stage 1: get_request 컴파일 오류 수정
```

Stage 1은 중복 upstream patch를 그대로 cherry-pick하지 않고 승인된 최소 범위만 결합했다. commit 본문은 PR #1385, #1392, #1400, #1412를 참조하고 다음 실질 기여자를 보존한다.

- `Co-authored-by: V.Shkriabets <vshcryabets@gmail.com>`
- `Co-authored-by: yubiuser <github@yubiuser.dev>`

Stage 4는 upstream PR #1412의 CI commit `a3fa7336`을 비교 기준으로 사용하고 다음 원 작성자를 보존했다.

- `Co-authored-by: DL6ER <dl6er@dl6er.de>`

다른 사람의 `Signed-off-by`는 대신 작성하지 않았다.

## 검증 결과

### 수용 기준별 결과

| 수용 기준 | 결과 | 근거 |
|---|---|---|
| `get_request()` compile 오류 제거 | OK | Make 기본·전체 기능, CMake와 Docker compile 성공 |
| 요청 프레이밍 정책 회귀 검증 | OK | 6개 독립 fixture와 targeted CTest 1/1 통과 |
| 전체 unit test 통과 | OK | 로컬 CTest 50/50 및 CI build 전체 job 통과 |
| Docker clean build 통과 | OK | `--no-cache` image build 성공 |
| macOS OpenSSL 1.1 의존 제거 | OK | OpenSSL 3 macOS job 2/2가 두 run에서 모두 성공 |
| 전체 CI build 통과 | OK | push run과 pull_request run 각각 13/13 성공 |
| 개인 branch CI 제외 | OK | `2f330722`, `7bfd3a22` push 후 새 CI build 미생성 |
| fork 전용 설정 분리 | OK | source branch에 개인 branch filter 없음 |
| 저작자와 upstream 참고 정보 보존 | OK | Stage 1·4 commit의 참조와 `Co-authored-by` 확인 |
| 기존 PR 재사용 | OK | 별도 이슈·branch·PR 없이 PR #3 head 갱신 |

MISS 항목은 없다. 단, CIFuzz는 아래 잔여 위험에 기록한 별도 원인으로 실패한다.

### 실행 결과 요약

```text
make build:                    PASS
make WITH_ALL=1:               PASS
CMake/OpenSSL 3 build:         PASS
targeted CTest:                1/1 PASS
full CTest:                    50/50 PASS
Docker no-cache build:         PASS
workflow YAML/structure check: PASS
CI build push run:             13/13 PASS
CI build pull_request run:     13/13 PASS
personal branch CI exclusion:  PASS
```

GitHub Actions:

- `push`: [run 31598147716](https://github.com/edwardkim/civetweb/actions/runs/31598147716), 13/13 성공
- `pull_request`: [run 31598151933](https://github.com/edwardkim/civetweb/actions/runs/31598151933), 13/13 성공
- macOS NoDynLoad OpenSSL 3: 두 event 모두 성공
- macOS package OpenSSL 3: 두 event 모두 성공

Docker image:

```text
civetweb:task2
sha256:267a9888497cc80ca759faf52911ec8086f302b97ab6423a8f393225ebb35421
amd64 linux, user civetweb
```

## upstream 상태와 fork 유지 전략

2026-08-12 최종 확인 시 `civetweb/civetweb:master`는 `588860e3`이며 local `upstream/master`와 일치했다.

| 관련 PR | 상태 |
|---|---|
| #1385 | open, 미병합 |
| #1392 | open, 미병합 |
| #1400 | closed, 미병합 |
| #1412 | open, 미병합 |

관련 수정이 upstream master에 병합되지 않았으므로 검증된 patch를 `edwardkim/civetweb` fork의 기존 PR #3에서 먼저 유지하는 전략은 유효하다. upstream maintainer에게 새 PR이나 comment를 게시하는 작업은 계속 별도 판단 대상으로 둔다.

## 잔여 위험과 후속 작업

- PR #3의 CIFuzz run [31598151893](https://github.com/edwardkim/civetweb/actions/runs/31598151893)은 실패한다. OSS-Fuzz container가 upstream `civetweb/civetweb`을 고정 clone한 뒤 fork 전용 merge ref를 찾지 못하는 별도 원인이며 CI build나 Task #2 patch의 실패가 아니다.
- `push`와 `pull_request`가 같은 PR head commit에서 CI build를 중복 실행한다. 이번 범위에는 event 중복 최적화를 포함하지 않았다.
- Homebrew의 formula alias와 tap trust warning annotation은 현재 job 결과에 영향을 주지 않지만 runner image 변경 시 재검토해야 한다.
- HTTP/2와 bundled Lua, SQLite, Duktape, 기존 form·SSL·unittest 코드의 compiler warning은 Task #2 diff 밖이므로 수정하지 않았다.
- GitHub Issue #2는 open 상태로 유지한다. PR #3 review·merge 또는 작업지시자의 별도 승인 전에는 close하지 않는다.
- 로컬 Docker image와 ignored build output은 검증 근거로 유지하며 게시·merge 후 정리 절차에서 제거한다.

현재 게시 범위:

```text
local/task2 -> origin/publish/task2
target: edwardkim/civetweb:master
source commits: 8629836d, 783c1676, cd592164
existing PR: #3
```

개인 운영 문서와 `personal/hyper-waterfall` filter는 fork 전용 branch에만 유지한다.

## 작업지시자 확인 사항

- Task #2의 source 수정, 회귀 테스트, macOS OpenSSL 3 전환과 CI build 13/13 성공을 최종 결과로 기록한다.
- 새 PR을 만들지 않고 기존 PR #3을 review 대상으로 유지한다.
- CIFuzz 수정, PR merge와 Issue #2 close는 이번 최종보고 범위에 포함하지 않는다.
