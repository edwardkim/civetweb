# Task #2 수행계획서 — upstream 빌드 수정 패치를 fork에서 우선 유지하고 검증

GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
마일스톤: M117

## 목적

현재 `upstream/master`에서 컴파일되지 않는 `get_request()`의 요청 프레이밍 처리 코드를 최소 범위로 수정하고, 의도된 `Transfer-Encoding` 및 `Content-Length` 검증 동작을 회귀 테스트로 고정한다. Make, CMake, Docker 검증을 통과한 패치를 `edwardkim/civetweb` fork에 먼저 유지한다.

동일 수정이 포함된 upstream PR을 무시하거나 중복 저작물처럼 취급하지 않는다. 기존 패치를 비교해 불필요한 변경을 배제하고, 실질적으로 재사용하는 변경의 원 저작자와 참고 PR을 커밋 및 작업 문서에 남긴다.

## 배경

`upstream/master`와 `origin/master`의 현재 HEAD는 `588860e3`이며, `src/civetweb.c`의 `get_request()`에 잘못 분리된 `if` 조건과 더 이상 선언되지 않은 `cl` 참조가 포함되어 있다. 깨끗한 detached worktree에서 `docker build --no-cache`를 실행한 결과 `make build`가 `expected ';'`, `cl undeclared`, `else without a previous if` 오류로 종료됨을 확인했다.

upstream에는 동일 현상을 다루는 이슈 #1386, #1388, #1408과 PR #1385, #1392, #1400, #1412가 있다. PR #1400은 `src/civetweb.c`의 최소 5줄 교체이며 Linux GCC·Clang을 포함한 CI 빌드가 통과했다. PR #1392와 #1412는 빌드 복구 외에 형 변환, form 처리 또는 CI 변경을 함께 포함하므로 그대로 가져오면 이번 이슈 범위를 확장한다. maintainer 응답과 merge가 지연되고 있어 작업지시자 승인에 따라 fork 우선 유지 전략을 적용한다.

## 범위

### 포함

- 최신 `upstream/master` 기반 `local/task2`에서 `get_request()`의 조건문과 변수 참조를 최소 수정한다.
- 다음 요청 프레이밍 경우를 검증한다.
  - `Transfer-Encoding: chunked` 단독 요청 허용 및 chunked 상태 설정
  - `Transfer-Encoding: identity`와 유효한 `Content-Length` 처리
  - `Transfer-Encoding: chunked`와 `Content-Length` 동시 사용 거부
  - 지원하지 않는 Transfer-Encoding 거부
  - 유효·무효·음수 `Content-Length` 처리
- 기존 private unit test 구조를 우선 사용해 회귀 테스트를 추가한다.
- Make 기본·전체 기능 빌드, CMake/CTest, 깨끗한 worktree의 Docker 빌드를 검증한다.
- 기존 PR의 패치 또는 논리를 재사용할 경우 원 저작자와 참고 PR을 커밋·보고서에 명시한다.
- 새로 작성하거나 수정하는 소스 주석은 영어로 작성한다.
- 최종 승인 뒤에만 `publish/task2`를 fork에 게시하고 `edwardkim/civetweb:master` 대상 내부 PR을 준비한다.
- 게시된 PR #3의 CI 검증을 막는 macOS OpenSSL 1.1 matrix를 OpenSSL 3으로 전환한다.
- fork 전용 `personal/hyper-waterfall` push는 `CI build` 대상에서 제외한다.
- CI 보정은 새 이슈나 새 PR로 분리하지 않고 기존 Task #2와 PR #3의 검증 보완으로 처리한다.

### 제외

- 이번 task에서 `civetweb/civetweb` upstream PR, issue 댓글 또는 review를 게시하지 않는다.
- `src/handle_form.inl`과 OpenSSL 설정과 관련 없는 컴파일 경고를 수정하지 않는다.
- CIFuzz의 upstream 저장소 고정 checkout 문제와 `fail-fast` 정책은 수정하지 않는다.
- API, 멀티스레딩, 구조 또는 요청 프레이밍 정책 자체를 변경하지 않는다.
- 개인 전용 `AGENTS.md` 심볼릭 링크가 Docker context에 미치는 문제를 제품 Dockerfile 변경으로 해결하지 않는다.
- Hyper-Waterfall Task #1의 branch, 계획서 또는 migration 변경을 포함하지 않는다.
- upstream merge 일정이나 maintainer 활동 재개 시점을 전제하지 않는다.

## 설계 방향

- 소스 branch `local/task2`는 `upstream/master`의 `588860e3`에서 시작하며 `/home/edward/vsworks/myweb/civetweb-task2` worktree에서만 수정한다.
- 작업 문서는 `origin/personal/hyper-waterfall`에서 분기한 개인 전용 `personal/task2` branch와 `/home/edward/vsworks/myweb/civetweb-workflow-task2` worktree에서 관리한다. 운영 문서는 소스 commit과 fork master PR에 포함하지 않는다.
- 핵심 수정은 `h_chunk`를 Transfer-Encoding 비교에, `h_len`을 Content-Length 변환과 종료 포인터 검증에 사용하고 손상된 `if` 괄호를 복원하는 것이다.
- PR #1400의 최소 diff를 우선 비교 기준으로 삼는다. 다른 PR의 별도 보정은 현재 오류 해결과 검증에 필수라는 증거가 있을 때만 범위 변경 승인을 요청한다.
- 동일 패치를 직접 재작성해 원 저작자를 가리지 않는다. 기존 commit을 실질적으로 채택하면 Author 또는 `Co-authored-by`와 원 PR/commit 참조를 보존하고, 구현계획서에서 정확한 적용 방식을 확정한다.
- 회귀 테스트는 `unittest/private.c`에서 static `get_request()`에 접근하는 기존 구조를 우선 사용한다. 안정적인 fixture 구성이 불가능하면 소켓 수준의 `unittest/public_server.c` 대안을 적용하기 전에 범위 변경을 보고한다.
- fork 우선 유지 결정은 이번 task의 게시 대상만 `edwardkim/civetweb:master`로 바꾸는 명시적 예외다. 추후 upstream 기여는 별도 승인과 중복 PR 상태 재확인을 거친다.

## 예상 변경 파일

소스 수정:

- `src/civetweb.c`
- `unittest/private.c`
- `.github/workflows/cibuild.yml` — 기존 PR에는 macOS OpenSSL 3 전환만 포함

조건부 소스 수정(기존 private fixture로 검증할 수 없는 경우에만 별도 승인):

- `unittest/public_server.c`
- `unittest/CMakeLists.txt`

개인 운영 문서(소스 commit 및 fork master PR에서 제외):

- `mydocs/orders/20260812.md`
- `mydocs/plans/task_m117_2.md`
- `mydocs/plans/task_m117_2_impl.md`
- `mydocs/working/task_m117_2_stage{N}.md`
- `mydocs/report/task_m117_2_report.md`
- `.github/workflows/cibuild.yml` — `personal/hyper-waterfall` branch에는 fork 전용 push 제외 조건을 추가

변경하지 않음:

- `Dockerfile`, `docs/`, `.github/workflows/cifuzz.yml`, `src/handle_form.inl`

## 잠정 단계

- **Stage 1 — 최소 컴파일 수정과 출처 보존**
  - 네 개 upstream PR의 관련 diff를 최종 대조하고 `src/civetweb.c`에 요청 프레이밍 블록만 적용한다.
  - 선택한 적용 방식에 따라 원 저작자, commit 및 PR 참조를 보존한다.
  - `make build`와 컴파일 경고를 확인해 최소 수정이 독립적으로 빌드되는지 검증한다.
- **Stage 2 — 요청 프레이밍 회귀 테스트**
  - `unittest/private.c`에 Transfer-Encoding과 Content-Length 조합 테스트를 추가한다.
  - 새 테스트와 전체 CTest를 실행해 거부·허용 상태 및 오류 코드를 확인한다.
  - fixture 한계로 예상 파일 범위가 바뀌면 변경 전에 작업지시자 승인을 요청한다.
- **Stage 3 — 통합 빌드 검증과 fork 게시 준비**
  - Make 기본·전체 기능, CMake/CTest, Docker 빌드를 깨끗한 Task #2 worktree에서 실행한다.
  - `upstream/master...local/task2` diff에 승인된 소스·테스트 파일만 포함되는지 확인한다.
  - 최종 보고 승인 뒤 사용할 fork 내부 `publish/task2` 게시 범위와 기존 upstream PR 중복 상태를 정리한다.
- **Stage 4 — 기존 PR의 CI 실행 기반 복구**
  - `personal/hyper-waterfall` branch의 push에서 `CI build`를 제외한다.
  - 기존 PR #3의 macOS matrix를 OpenSSL 3으로 전환하고 OpenSSL 1.1 전용 job과 setup을 제거한다.
  - fork 전용 branch filter와 upstream에 재사용 가능한 OpenSSL 변경을 서로 다른 branch와 commit으로 유지한다.
  - 정적 workflow 검증 뒤 기존 `publish/task2`와 `personal/hyper-waterfall`만 갱신하며 새 PR은 만들지 않는다.

## 검증 계획

### Stage 1

- `git diff --check`
- `make clean`
- `make build`
- 변경 block에서 `cl` 잔존 참조, 괄호, 들여쓰기와 `h_chunk`·`h_len` 사용을 수동 대조한다.
- PR #1385, #1392, #1400, #1412의 관련 diff와 최종 patch 및 저작자 표시를 대조한다.

### Stage 2

- `cmake -S . -B output -DCIVETWEB_BUILD_TESTING=ON`
- `cmake --build output --parallel 2`
- `ctest --test-dir output --output-on-failure`
- 최소한 다음 결과를 assertion으로 확인한다.
  - chunked 단독 요청: 성공, `is_chunked == 1`, `content_len == 0`
  - identity와 유효 Content-Length: 성공, 길이 값 반영
  - chunked와 Content-Length 동시 사용: 실패, HTTP 400
  - 지원하지 않는 Transfer-Encoding: 실패, HTTP 400
  - 무효 또는 음수 Content-Length: 실패, HTTP 411

### Stage 3

- `make clean && make build`
- `make clean && make WITH_ALL=1`
- `cmake -S . -B output -DCIVETWEB_BUILD_TESTING=ON`
- `cmake --build output --parallel 2`
- `ctest --test-dir output --output-on-failure`
- `docker build --no-cache --progress=plain -t civetweb:task2 .`
- `git diff --check`
- `git diff --name-only upstream/master...local/task2`
- source, workflow, 기존 Task #1 worktree의 `git status --short --branch`를 각각 확인한다.

### Stage 4

- `git diff --check`
- `.github/workflows/cibuild.yml`의 matrix와 조건식을 수동 대조한다.
- 사용 가능하면 `actionlint`로 workflow를 정적 검증한다.
- source branch diff에는 macOS OpenSSL 3 전환만, 개인 branch diff에는 정확한 `personal/hyper-waterfall` push 제외 조건이 포함되는지 확인한다.
- 기존 PR #3의 GitHub Actions 결과와 `personal/hyper-waterfall` push에서 `CI build`가 생성되지 않는지 확인한다.

## 리스크

- **중복 패치의 저작자 누락**: 동일 수정이 여러 upstream PR에 존재한다. 최종 patch 출처를 대조하고 실질적으로 채택한 저작자와 PR을 commit 및 보고서에 기록한다.
- **범위 확대**: PR #1392와 #1412에는 form 처리와 CI 변경이 섞여 있다. 이번 task는 `get_request()`와 직접 회귀 테스트만 포함하며 다른 오류는 별도 이슈로 분리한다.
- **요청 프레이밍 의미 변경**: 컴파일만 통과해도 chunked와 Content-Length 동시 요청을 잘못 허용할 수 있다. 허용·거부 조합을 명시적인 assertion으로 고정한다.
- **private test fixture 복잡도**: `get_request()`는 connection과 domain context를 요구한다. 기존 구조로 안정적으로 구성되지 않으면 임의 mock 확장 대신 소켓 수준 테스트 전환을 승인받는다.
- **로컬 Docker false failure**: 메인 source worktree의 개인 symlink는 `COPY *.md`에 걸릴 수 있다. 개인 링크가 없는 전용 `civetweb-task2` worktree에서 검증해 제품 오류와 분리한다.
- **upstream과 fork의 장기 분기**: fork 패치는 upstream 반영 전까지 별도 유지가 필요하다. upstream 변경을 주기적으로 재확인하고 추후 게시 시 중복·충돌을 다시 평가한다.
- **병렬 Task 문서 충돌**: Task #1과 Task #2가 같은 날짜의 orders 파일을 만든다. 각 문서 branch를 분리하고 `personal/hyper-waterfall` 통합 시 두 행을 보존하는 병합을 별도 승인 게이트에서 수행한다.
- **CI 복구의 순환 PR화**: CI 오류를 별도 이슈와 PR로 분리하면 그 PR도 같은 실패 workflow에 의존한다. 기존 Task #2와 PR #3 안에서 복구하고 별도 PR을 만들지 않는다.
- **fork 전용 설정의 upstream 혼입**: `personal/hyper-waterfall` 제외 조건은 개인 branch에만 commit하고 기존 PR #3에는 포함하지 않는다.

## 승인 요청 사항

- 최소 소스 수정, private 회귀 테스트, 통합 검증으로 구성한 3개 Stage를 승인 요청한다.
- PR #1400의 최소 patch를 우선 비교 기준으로 사용하고, 실질적으로 채택한 기존 저작자 정보를 보존하는 정책을 승인 요청한다.
- `unittest/private.c`를 기본 테스트 위치로 사용하되 fixture 한계가 확인되면 다른 테스트 파일 변경 전에 다시 승인받는 조건을 승인 요청한다.
- 제품 `Dockerfile`이나 개인 symlink를 수정하지 않고 깨끗한 Task #2 worktree에서 Docker를 검증하는 방향을 승인 요청한다.
- 최종 승인 후 upstream이 아니라 `edwardkim/civetweb:master`에 먼저 게시하고 upstream 기여는 별도 결정하는 예외를 승인 요청한다.

승인되면 `task_m117_2_impl.md`에서 단계별 적용 방식, 저작자 표시, 검증 명령과 커밋 메시지를 구체화한다.
