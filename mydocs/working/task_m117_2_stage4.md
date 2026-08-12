# Task #2 Stage 4 완료보고서 — 기존 PR의 CI 실행 기반 복구

GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
기존 PR: [#3](https://github.com/edwardkim/civetweb/pull/3)
구현계획서: [`task_m117_2_impl.md`](../plans/task_m117_2_impl.md)
Stage: 4
검증 시각: 2026-08-12T21:57:37+09:00
소스 결과 commit: `cd59216490815d8548f87283e5738712c34cb090`
개인 운영 commit: `2f330722`

## 단계 목적

기존 PR #3을 검증하던 GitHub Actions가 Homebrew에서 제거된 `openssl@1.1` 때문에 source compile 전에 실패하는 문제를 같은 Task와 PR 안에서 복구한다. 별도 이슈나 별도 PR을 만들지 않으며, fork 전용 `personal/hyper-waterfall` push 제외 조건은 upstream에 재사용 가능한 macOS OpenSSL 변경과 branch·commit 수준에서 분리한다.

## 산출물

| branch | 산출물 | 결과 |
|---|---|---|
| `local/task2` / `publish/task2` | `.github/workflows/cibuild.yml` | macOS job 두 개를 OpenSSL 3으로 통일, commit `cd592164` |
| `personal/task2` / `personal/hyper-waterfall` | `.github/workflows/cibuild.yml` | 동일 OpenSSL 3 전환과 정확한 개인 branch push 제외, commit `2f330722` |
| `personal/task2` | 수행·구현 계획서와 오늘할일 | Stage 4 재개 범위 및 진행 상태 반영 |

source workflow commit은 7줄 추가, 49줄 삭제다. 개인 workflow commit은 branch filter 2줄을 더 포함해 9줄 추가, 49줄 삭제다.

## 본문 변경 정도 / 본문 무손실 여부

`src/civetweb.c`와 `unittest/private.c`의 Stage 1·2 변경은 수정하지 않았다. source branch에는 다음 CI 변경만 추가했다.

- `Clang-OSX-Complete-NoLua-Release-OpenSSL_1_1_NoDynLoad`를 OpenSSL 3 NoDynLoad job으로 전환
- 중복된 `OSX-Package_OpenSSL_1_1` matrix 항목 제거
- macOS OpenSSL 1.1 setup step 제거
- OpenSSL 3 setup의 현재 Homebrew 상태를 설명하는 영어 주석 적용

`personal/hyper-waterfall` 제외 조건은 개인 운영 branch에만 존재한다. source branch와 기존 PR #3에는 `branches-ignore`나 개인 branch 이름이 포함되지 않는다. CIFuzz, Linux matrix와 `fail-fast`는 변경하지 않았다.

## 검증 결과

### 로컬 workflow 검사

```bash
git diff --check
python3 -c 'import yaml; ...'
```

- OK — source와 개인 운영 worktree 모두 `git diff --check`를 통과했다.
- OK — PyYAML 6.0.1로 두 workflow를 파싱했다.
- OK — source workflow는 macOS matrix 2개가 모두 `OPENSSL_1_1: NO`, `OPENSSL_3_0: YES`임을 assertion으로 확인했다.
- OK — 개인 workflow는 위 matrix 조건과 `push.branches-ignore == ["personal/hyper-waterfall"]`를 assertion으로 확인했다.
- 참고 — 로컬에 `actionlint`가 설치되어 있지 않아 새 도구를 설치하지 않고 실제 GitHub Actions 실행을 최종 workflow 검증으로 사용했다.

### 기존 PR #3 CI build

source commit `cd592164`를 새 PR 없이 기존 `publish/task2`에 push했다.

| event | run | 결과 |
|---|---|---|
| `push` | [31598147716](https://github.com/edwardkim/civetweb/actions/runs/31598147716) | 13/13 job 성공 |
| `pull_request` | [31598151933](https://github.com/edwardkim/civetweb/actions/runs/31598151933) | 13/13 job 성공 |

macOS 결과:

| job | push run | pull_request run |
|---|---:|---:|
| `Clang-OSX-Complete-NoLua-Release-OpenSSL_3_0_NoDynLoad` | 성공, 7분 29초 | 성공, 7분 7초 |
| `OSX-Package_OpenSSL_3_0` | 성공, 7분 57초 | 성공, 8분 3초 |

- OK — OpenSSL 3 설치, CMake 구성, 실행 파일 build·확인과 unit test가 모두 성공했다.
- OK — package job은 `Makefile.osx package`까지 성공했다.
- 참고 — `openssl@3.0`이 이미 `openssl@3`으로 link됐다는 Homebrew 문구와 기존 tap trust 문구는 warning annotation이며 job 결론은 성공이다.
- 참고 — 현 workflow가 `push`와 `pull_request`를 모두 구독하므로 같은 PR head commit에 CI build가 두 번 실행된다. 이번 요청 범위는 개인 branch 제외와 macOS OpenSSL 전환이므로 event 중복 최적화는 하지 않았다.

### 개인 branch 제외

적용 전 마지막 개인 branch CI는 실패 run [31595869996](https://github.com/edwardkim/civetweb/actions/runs/31595869996), head `fd3de888`이었다.

`2f330722`를 `origin/personal/hyper-waterfall`에 push한 뒤 동일 branch와 `cibuild.yml` 기준 run 목록을 두 차례 조회했다. 목록에는 기존 run `31595869996`만 남았고 `2f330722`에 대한 새 `CI build` run은 생성되지 않았다.

- OK — `personal/hyper-waterfall` push 제외 조건이 실제 GitHub Actions trigger에서 동작했다.
- OK — 개인 branch 변경은 새 PR 없이 fork 전용 remote branch에 직접 반영했다.

### 범위 밖 check

PR #3의 CIFuzz run [31598151893](https://github.com/edwardkim/civetweb/actions/runs/31598151893)은 기존과 같이 실패했다. OSS-Fuzz container가 upstream `civetweb/civetweb`을 고정 clone한 뒤 fork 전용 merge ref를 찾지 못하는 별도 원인이다. 이번 Stage는 `.github/workflows/cifuzz.yml`을 변경하지 않았으며 CI build 성공과 CIFuzz 실패를 섞어 해석하지 않는다.

## branch와 변경 범위

source branch의 `upstream/master` 대비 최종 범위:

```text
.github/workflows/cibuild.yml | 56 ++++-----------------------
src/civetweb.c                | 26 +++++++------
unittest/private.c            | 90 +++++++++++++++++++++++++++++++++++++++++++
3 files changed, 111 insertions(+), 61 deletions(-)
```

source commit:

```text
cd592164 Task #2 Stage 4: macOS CI를 OpenSSL 3으로 전환
783c1676 Task #2 Stage 2: 요청 프레이밍 회귀 테스트 추가
8629836d Task #2 Stage 1: get_request 컴파일 오류 수정
```

Stage 4 source commit은 upstream PR #1412의 CI commit `a3fa7336`을 참조하고 원 작성자 `DL6ER <dl6er@dl6er.de>`를 `Co-authored-by`로 보존했다.

## 잔여 위험

- PR #3에는 범위 밖 CIFuzz 실패가 남아 있어 전체 check 표시는 아직 실패다. 원인은 CI build나 Task #2 source patch가 아니다.
- `push`와 `pull_request`가 같은 head에 CI build를 중복 실행한다. 개선하려면 별도 범위 판단이 필요하다.
- Homebrew warning annotation은 현재 성공에 영향을 주지 않지만 runner image와 formula alias가 바뀌면 재검토해야 한다.
- 개인 branch filter는 의도적으로 source PR에 포함되지 않으므로 fork 운영 branch에서만 유지해야 한다.

## 다음 단계 영향

- Stage 4 승인 후 기존 최종 결과보고서의 단계 수, 변경 파일, CI 검증과 잔여 위험을 갱신한다.
- 기존 PR #3 본문에서 CI build 실패 설명을 13/13 성공 결과와 Stage 4 commit으로 교체한다.
- 새 이슈, 새 branch 또는 새 PR은 만들지 않는다.

## 승인 요청

- 기존 PR #3 안에서 macOS OpenSSL 3 전환과 CI build 13/13 성공을 달성한 결과를 승인 요청한다.
- `personal/hyper-waterfall` push 제외를 fork 전용 branch에만 적용하고 실제로 새 CI run이 생성되지 않은 결과를 승인 요청한다.
- 승인되면 최종 결과보고서와 기존 PR #3 본문만 갱신한다.
