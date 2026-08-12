# Task #2 최종 결과보고서 — upstream 빌드 수정 패치를 fork에서 우선 유지하고 검증

GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
마일스톤: M117 — CivetWeb v1.17 fork-maintained fixes and validation
구현계획서: [`task_m117_2_impl.md`](../plans/task_m117_2_impl.md)
단계 수: 3
source 기준: `upstream/master` commit `588860e30721bf5453b0440c390865a8e85dcae5`
source 결과: `local/task2` commit `783c167681642eb50be20a6593424e5be61a68f3`

## 작업 요약

`civetweb/civetweb:master`의 `get_request()`에 남아 있던 손상된 조건식과 제거된 `cl` 변수 참조 때문에 Make와 Docker build가 실패하는 문제를 fork에서 우선 복구했다. 기존 upstream PR 네 개를 비교해 최소 source patch만 수동 병합하고 실질 기여자를 commit에 보존했다.

수정된 요청 프레이밍 동작은 네트워크 I/O 없는 private fixture와 여섯 회귀 경우로 고정했다. 마지막으로 Make 기본·전체 기능 build, CMake/OpenSSL 3 build, CTest 50개와 Docker no-cache build를 통합 검증했다.

upstream PR이나 issue에는 새 PR, comment 또는 상태 변경을 하지 않았다. source branch도 아직 원격에 게시하지 않았으며, 이번 최종보고 승인 뒤 별도 게시 승인을 받도록 유지한다.

## 단계별 결과

| Stage | 결과 | source commit | 완료보고서 |
|---|---|---|---|
| 1 — 최소 컴파일 수정과 출처 보존 | `get_request()` compile 복구, 기본 Make build 성공 | `8629836d` | [`task_m117_2_stage1.md`](../working/task_m117_2_stage1.md) |
| 2 — 요청 프레이밍 회귀 테스트 | 독립 fixture와 6개 경우 추가, CTest 50/50 성공 | `783c1676` | [`task_m117_2_stage2.md`](../working/task_m117_2_stage2.md) |
| 3 — 통합 빌드 검증과 fork 게시 준비 | Make, CMake/CTest, Docker와 branch 범위 검증 성공 | source commit 없음 | [`task_m117_2_stage3.md`](../working/task_m117_2_stage3.md) |

## 변경 파일 목록과 영향 범위

| 파일 | 변경량 | 영향 |
|---|---:|---|
| `src/civetweb.c` | 14줄 추가, 12줄 삭제 | `get_request()`의 header 변수, 조건식과 들여쓰기 복원. Zlib 비활성 build의 `h_zip` 미사용 경고 방지 |
| `unittest/private.c` | 90줄 추가 | `Transfer-Encoding`과 `Content-Length` 조합 6개에 대한 private 회귀 테스트 |

합계:

```text
2 files changed, 104 insertions(+), 12 deletions(-)
```

공개 API, 함수 signature, 요청 프레이밍 정책, 공식 사용자 문서와 Dockerfile은 변경하지 않았다. `unittest/public_server.c`, `unittest/CMakeLists.txt`, `src/handle_form.inl`, SSL 형 변환과 CI workflow도 범위에서 제외했다.

개인 운영 문서와 `mydocs/`는 `personal/task2`에만 있으며 source branch 및 향후 upstream 기여 diff에 포함되지 않는다. 새 source 주석을 추가할 필요는 없었고, 새 test 식별자와 구현은 영어로 작성했다.

## 변경 전·후 정량 비교

| 항목 | 변경 전 | 변경 후 |
|---|---|---|
| 기본 Make build | `get_request()` 조건식과 `cl` 참조 compile 오류 | exit code 0, 관련 오류·경고 없음 |
| 요청 프레이밍 전용 회귀 경우 | 없음 | 6개 |
| targeted HTTP message CTest | 수정 동작을 직접 고정하지 않음 | 1/1 통과 |
| 전체 CTest | source가 compile되지 않아 정상 실행 불가 | 50/50 통과 |
| 전체 기능 Make build | source compile 오류로 차단 | `WITH_ALL=1` 성공 |
| Docker build | source compile 오류로 차단 | no-cache build 성공 |
| source 변경 파일 | 해당 없음 | 2개 |
| source commit | 해당 없음 | 2개 |

회귀 테스트는 다음 동작을 명시적으로 검증한다.

1. `Transfer-Encoding: chunked` 허용
2. `Transfer-Encoding: identity`와 유효한 `Content-Length` 허용
3. chunked와 `Content-Length` 동시 사용을 error 400으로 거부
4. 지원하지 않는 transfer encoding을 error 400으로 거부
5. 숫자가 아닌 `Content-Length`를 error 411로 거부
6. 음수 `Content-Length`를 error 411로 거부

## source commit과 저작자 보존

```text
783c1676 Task #2 Stage 2: 요청 프레이밍 회귀 테스트 추가
8629836d Task #2 Stage 1: get_request 컴파일 오류 수정
```

Stage 1은 중복 upstream patch를 그대로 cherry-pick하지 않고 승인된 최소 범위만 결합했다. commit 본문은 PR #1385, #1392, #1400, #1412를 참조하고 다음 실질 기여자를 보존한다.

- `Co-authored-by: V.Shkriabets <vshcryabets@gmail.com>`
- `Co-authored-by: yubiuser <github@yubiuser.dev>`

Stage 2 회귀 테스트는 현재 작업자가 새로 작성했다. 다른 저작자의 `Signed-off-by`는 대신 작성하지 않았다.

## 검증 결과

### 수용 기준별 결과

| 수용 기준 | 결과 | 근거 |
|---|---|---|
| `get_request()` compile 오류 제거 | OK | Make 기본·전체 기능, CMake와 Docker compile 성공 |
| 승인된 최소 source 범위 유지 | OK | `src/civetweb.c`, `unittest/private.c`만 변경 |
| 요청 프레이밍 정책 회귀 검증 | OK | 6개 독립 fixture와 targeted CTest 1/1 통과 |
| 전체 unit test 통과 | OK | CTest 50/50 통과, 351.38초 |
| Docker clean build 통과 | OK | `--no-cache` image build 성공 |
| 저작자와 upstream 참고 정보 보존 | OK | Stage 1 commit 본문과 `Co-authored-by` trailer 확인 |
| 개인 운영 문서 분리 | OK | source diff에 `.hyper-waterfall/`, `mydocs/` 없음 |
| 기존 Task #1과 main worktree 보존 | OK | 모든 worktree clean, 기존 branch 유지 |
| push와 외부 제안 승인 gate 유지 | OK | 원격 source branch, fork PR, upstream 변경 없음 |

MISS 항목은 없다.

### 실행 결과 요약

```text
make build:                 PASS
make WITH_ALL=1:            PASS
CMake/OpenSSL 3 build:      PASS
targeted CTest:             1/1 PASS
full CTest:                 50/50 PASS
Docker no-cache build:      PASS
source diff check:          PASS
worktree clean check:       PASS
```

Docker image:

```text
civetweb:task2
sha256:267a9888497cc80ca759faf52911ec8086f302b97ab6423a8f393225ebb35421
amd64 linux, user civetweb
```

최종보고 직전 source가 변하지 않았음을 다시 확인했고 targeted CTest 1/1도 재통과했다.

## upstream 상태와 fork 유지 전략

2026-08-12 최종 확인 시 `civetweb/civetweb:master`는 여전히 `588860e3`이며 local `upstream/master`와 일치한다.

| 관련 PR | 상태 |
|---|---|
| #1385 | open, 미병합 |
| #1392 | open, 미병합 |
| #1400 | closed, 미병합 |
| #1412 | open, 미병합 |

관련 수정이 upstream master에 병합되지 않았으므로 검증된 patch를 `edwardkim/civetweb` fork에 먼저 유지하는 전략은 유효하다. upstream maintainer에게 새 PR이나 comment를 게시하는 작업은 계속 별도 판단 대상으로 둔다.

## 잔여 위험과 후속 작업

- Linux amd64 host와 Alpine amd64 Docker build를 검증했지만 upstream GitHub Actions의 전체 OS/compiler matrix를 대체하지는 않는다.
- HTTP/2와 bundled Lua, SQLite, Duktape, 기존 form·SSL·unittest 코드에서 compiler 경고가 남아 있다. 모두 Task #2 diff 밖이므로 최소 범위 원칙에 따라 수정하지 않았다.
- CMake test는 host OpenSSL 3 API 선택과 `output/cgi_test.cgi` helper 준비가 필요하다. 향후 재현 시 같은 환경 조건을 명시해야 한다.
- upstream master가 변경되면 게시 직전에 source commit 두 개의 conflict와 중복 여부를 다시 확인해야 한다.
- GitHub Issue #2는 open 상태로 유지한다. fork 내부 PR의 review·merge 또는 작업지시자의 별도 승인 전에는 close하지 않는다.
- 로컬 Docker image와 ignored build output은 검증 근거로 현재 유지한다. 게시·merge 후 정리 절차에서 재생성 가능한 부산물을 제거한다.

후속 게시 후보 범위:

```text
local/task2 -> origin/publish/task2
target: edwardkim/civetweb:master
source commits: 8629836d, 783c1676
```

개인 운영 문서는 `personal/task2`에만 유지한다. `publish/task2`와 fork 내부 PR에는 source commit 두 개만 포함한다.

## 작업지시자 승인 요청

- Task #2의 source 변경, 회귀 테스트, 통합 검증과 최종 결과를 승인 요청한다.
- 승인되면 별도 게시 gate로 `publish/task2` 원격 push와 `edwardkim/civetweb:master` 대상 fork 내부 Open PR 생성을 준비한다.
- 이번 최종보고 commit에는 push, PR 생성, upstream comment와 issue close를 포함하지 않는다.
