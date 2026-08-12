# Task #2 Stage 3 완료보고서 — 통합 빌드 검증과 fork 게시 준비

GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
구현계획서: [`task_m117_2_impl.md`](../plans/task_m117_2_impl.md)
Stage: 3
검증 시각: 2026-08-12T21:05:22+09:00
소스 기준 commit: `783c167681642eb50be20a6593424e5be61a68f3`
upstream 기준 commit: `588860e30721bf5453b0440c390865a8e85dcae5`

## 단계 목적

Stage 1의 최소 컴파일 수정과 Stage 2의 요청 프레이밍 회귀 테스트를 source 변경 없이 통합 검증한다. Make 기본·전체 기능 build, CMake test build, 전체 CTest와 Docker image build를 수행하고, fork 게시 후보가 승인된 source 파일과 commit만 포함하는지 확인한다.

## 산출물

| 산출물 | 결과 | commit 여부 |
|---|---|---|
| Make 기본 build | 성공 | build output은 제외 |
| Make `WITH_ALL=1` build | 성공 | build output은 제외 |
| CMake/OpenSSL 3 test build | 성공 | `output/`은 제외 |
| CTest | targeted 1/1, 전체 50/50 성공 | test output은 제외 |
| Docker image `civetweb:task2` | 성공, image ID `sha256:267a9888497cc80ca759faf52911ec8086f302b97ab6423a8f393225ebb35421` | 로컬 image는 제외 |
| source branch 검증 | 파일 2개, commit 2개 | 기존 Stage 1·2 commit만 유지 |

Stage 3에서는 source commit을 추가하지 않았다. 검증 결과와 승인 요청만 이 개인 운영 문서에 기록한다.

## 본문 변경 정도 / 본문 무손실 여부

Stage 3에서 tracked source와 test 파일은 변경하지 않았다. `make`, CMake, CTest가 만든 파일은 기존 `.gitignore` 규칙으로 제외되며 Docker image도 Git commit 대상이 아니다.

Docker context는 개인 운영 symlink가 없는 `/home/edward/vsworks/myweb/civetweb-task2`를 사용했다. source branch에는 `.hyper-waterfall/`, `mydocs/` 또는 개인 운영 문서가 포함되지 않는다.

## 검증 결과

### Make 기본 build

```bash
make clean
make build
```

결과:

- OK — `src/civetweb.c`, `src/main.c` compile과 `civetweb` link가 exit code 0으로 완료됨.
- OK — Task #2의 `get_request()` compile 오류와 새 경고가 출력되지 않음.

### Make 전체 기능 build

```bash
make clean
make WITH_ALL=1
```

결과:

- OK — Lua, SQLite, Duktape, Zlib, HTTP/2, IPv6, WebSocket, Unix domain socket과 server statistics를 포함한 build가 exit code 0으로 완료됨.
- 참고 — `src/http2.inl`과 bundled Lua, SQLite, Duktape에서 기존 unused, fallthrough, macro 재정의 및 format 경고가 출력됨. Task #2 변경 파일에서 새로 발생한 경고는 없음.

### CMake와 CTest

host OpenSSL 3.0.13에 맞춰 fresh 구성했다.

```bash
cmake --fresh -S . -B output \
  -DCIVETWEB_BUILD_TESTING=ON \
  -DCIVETWEB_SSL_OPENSSL_API_1_1=OFF \
  -DCIVETWEB_SSL_OPENSSL_API_3_0=ON
cmake --build output --parallel 2
cc unittest/cgi_test.c -o output/cgi_test.cgi
ctest --test-dir output \
  -R '^test-private-http-message$' \
  --output-on-failure
ctest --test-dir output --output-on-failure
```

결과:

```text
targeted: 100% tests passed, 0 tests failed out of 1
full:     100% tests passed, 0 tests failed out of 50
Total Test time (real) = 351.38 sec
```

- OK — CMake fresh 구성과 test binary build가 성공함.
- OK — Check unit test framework 다운로드가 필요한 build 단계는 네트워크 허용 환경에서 완료함.
- OK — CMake가 자동 생성하지 않는 CGI helper를 로컬 `output/`에 준비했으며 commit하지 않음.
- OK — port binding과 TLS가 필요한 전체 CTest는 해당 기능이 허용된 실행 환경에서 수행함.
- 참고 — `src/handle_form.inl`, 기존 SSL 형 변환, `unittest/public_server.c`와 `unittest/private_exe.c`의 기존 경고가 재현됐으나 Task #2 변경 범위 밖임.

### Docker image build

```bash
docker build --no-cache --progress=plain -t civetweb:task2 .
docker image inspect civetweb:task2 \
  --format '{{.Id}} {{.Architecture}} {{.Os}} {{.Config.User}}'
```

결과:

```text
sha256:267a9888497cc80ca759faf52911ec8086f302b97ab6423a8f393225ebb35421 amd64 linux civetweb
```

- OK — 기존 Dockerfile을 수정하지 않고 Alpine 3.18 build stage에서 기본 build, 전체 기능 build와 install이 모두 성공함.
- OK — 최종 image가 `civetweb:task2`로 생성됐으며 비-root 사용자 `civetweb`으로 구성됨.
- 참고 — 전체 기능 build의 기존 HTTP/2 및 third-party 경고는 local Make build와 동일하게 재현됨.

### branch와 worktree 범위

```bash
git diff --check upstream/master...local/task2
git diff --name-only upstream/master...local/task2
git diff --stat upstream/master...local/task2
git log --format='%h %s' upstream/master..local/task2
git status --short --branch
```

결과:

```text
src/civetweb.c
unittest/private.c

src/civetweb.c     | 26 ++++++++--------
unittest/private.c | 90 ++++++++++++++++++++++++++++++++++++++++++++++++++++++
2 files changed, 104 insertions(+), 12 deletions(-)

783c1676 Task #2 Stage 2: 요청 프레이밍 회귀 테스트 추가
8629836d Task #2 Stage 1: get_request 컴파일 오류 수정
```

- OK — whitespace 검사가 경고 없이 통과함.
- OK — source branch는 `local/task2...upstream/master [ahead 2]`이고 tracked 변경이 없음.
- OK — fork 게시 후보에는 승인된 source 파일 두 개와 Stage 1·2 commit 두 개만 존재함.
- OK — main source, Task #1, Task #2 source와 Task #2 개인 문서 worktree가 모두 깨끗하며 각 기존 branch를 유지함.
- OK — local `upstream/master`와 `origin/master`는 모두 `588860e3`임.

### upstream 외부 상태 재확인

GitHub에서 2026-08-12에 읽기 전용으로 재확인했다.

| 항목 | 상태 |
|---|---|
| `civetweb/civetweb:master` | `588860e3`, local 기준과 일치 |
| PR #1385 | open, 미병합 |
| PR #1392 | open, 미병합 |
| PR #1400 | closed, 미병합 |
| PR #1412 | open, 미병합 |

upstream master와 관련 PR 상태에는 fork 우선 유지 전략을 변경할 새 병합이 없다.

## 잔여 위험

- Linux amd64 host와 Alpine amd64 Docker build를 검증했지만 upstream GitHub Actions의 전체 OS/compiler matrix를 로컬에서 대체하지는 않는다.
- Make 전체 기능 build와 Docker build의 HTTP/2 및 bundled third-party 경고는 기존 코드에 남아 있다. Task #2의 최소 범위를 유지하기 위해 수정하지 않았다.
- CMake test build는 host OpenSSL 3 API 선택과 수동 CGI helper 준비가 필요하다. 이 환경 조건은 향후 재검증에서도 명시해야 한다.
- source branch는 아직 fork에 push되지 않았고 fork 내부 PR도 생성되지 않았다. upstream에도 PR, comment 또는 issue 변경을 하지 않았다.
- upstream 관련 중복 PR이 장기간 미병합 상태이므로 fork에서 유지하는 동안 upstream master 변화와 conflict 여부를 게시 시점에 다시 확인해야 한다.

## 다음 단계 영향

- Stage 3 승인 후 별도의 최종 결과보고서를 작성해 Task #2 전체 결과와 게시 범위를 확정한다.
- 이후 별도 승인을 받아 source commit 두 개만 게시용 branch에 반영하고 fork에 push하거나 fork 내부 PR을 준비한다.
- 개인 운영 문서 commit은 `personal/task2`에만 유지하며 source 게시 branch와 upstream PR에 포함하지 않는다.
- 최종 결과보고서, push, fork 내부 PR 또는 upstream 제안은 이번 Stage 승인 범위에 포함하지 않는다.

## 승인 요청

- Make 기본·전체 기능 build, CMake/OpenSSL 3 build, CTest 50/50과 Docker no-cache build 결과를 승인 요청한다.
- source diff가 파일 두 개와 commit 두 개로 제한되고 모든 worktree가 깨끗한 게시 준비 상태임을 승인 요청한다.
- 승인되면 Hyper-Waterfall 절차에 따라 Task #2 최종 결과보고서 작성 단계로 진행한다.
