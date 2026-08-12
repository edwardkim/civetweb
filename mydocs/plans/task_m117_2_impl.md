# Task #2 구현계획서 — upstream 빌드 수정 패치를 fork에서 우선 유지하고 검증

수행계획서: [`task_m117_2.md`](task_m117_2.md)
GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
마일스톤: M117

## 단계 개요

| Stage | 제목 | 주요 산출 | 검증 |
|---|---|---|---|
| 1 | 최소 컴파일 수정과 출처 보존 | `src/civetweb.c` | 기본 Make 빌드, diff·경고 확인 |
| 2 | 요청 프레이밍 회귀 테스트 | `unittest/private.c` | targeted CTest, 전체 CTest |
| 3 | 통합 빌드 검증과 fork 게시 준비 | 검증 로그 요약과 Stage 보고서 | Make 기본·전체 기능, CMake/CTest, Docker, branch diff |
| 4 | 기존 PR의 CI 실행 기반 복구 | `cibuild.yml`, CI 재검증 | workflow 정적 검사, 기존 PR Actions |

## 작업 공간 및 문서 위치 확인

| 구분 | branch | worktree | commit 대상 | 일치 여부 |
|---|---|---|---|---|
| 제품 소스·테스트·공용 CI | `local/task2` | `/home/edward/vsworks/myweb/civetweb-task2` | `src/civetweb.c`, `unittest/private.c`, `.github/workflows/cibuild.yml` | OK |
| 개인 운영 문서 | `personal/task2` | `/home/edward/vsworks/myweb/civetweb-workflow-task2` | `mydocs/` 산출물 | OK |
| 기존 Task #1 문서 | `local/task1` | `/home/edward/vsworks/myweb/civetweb-workflow` | 변경하지 않음 | OK |

제품 사용자를 위한 공식 문서는 변경하지 않는다. 계획서와 단계 보고서는 fork 개인 운영 자료이므로 `mydocs/`에만 두며, 소스 commit과 fork master 대상 PR에는 포함하지 않는다. 일반 Hyper-Waterfall의 “소스와 단계 보고서를 같은 commit에 포함” 규칙은 프로젝트의 fork 전용 문서 분리 규칙에 따라 소스 commit과 문서 commit으로 나눠 적용한다.

## Stage 1 — 최소 컴파일 수정과 출처 보존

### 산출물

소스 수정:

- `src/civetweb.c`

개인 운영 문서:

- `mydocs/working/task_m117_2_stage1.md`

### 변경 내용

1. `get_request()`의 헤더 변수 선언을 다음 책임으로 분리한다.
   - `h_zip`은 `USE_ZLIB`이 활성화된 빌드에서만 선언한다.
   - `h_chunk`와 `h_len`은 모든 빌드에서 선언한다.
2. `Transfer-Encoding`과 `Content-Length` 조회문의 기존 CivetWeb tab 기반 들여쓰기를 복원한다.
3. 손상된 조건식을 `if ((h_chunk != NULL) && mg_strcasecmp(h_chunk, "identity"))` 의미로 복원한다.
4. chunked 유효성 검사에는 `h_chunk`, 길이 변환과 종료 포인터 검사에는 `h_len`을 사용한다.
5. `src/handle_form.inl`, SSL 형 변환, CI workflow 등 다른 PR에 포함된 별도 변경은 가져오지 않는다.
6. 핵심 변수·괄호 수정의 최초 확인 PR #1385와 Zlib 조건부 선언을 포함한 PR #1392를 commit 본문에 기록한다. 독립적인 최소 수정 PR #1400과 통합 검증 PR #1412는 비교 참고로 기록한다.

### 적용 방식과 저작자 표시

두 기존 commit을 그대로 cherry-pick하면 서로 중복되는 핵심 patch와 PR #1392의 범위 밖 변경까지 포함된다. 따라서 승인된 줄만 수동 병합하고, 결합 patch의 commit author는 현재 작업자로 유지하되 다음 `Co-authored-by` trailer로 실질 기여자를 보존한다.

```text
Co-authored-by: V.Shkriabets <vshcryabets@gmail.com>
Co-authored-by: yubiuser <github@yubiuser.dev>
```

commit 본문에는 다음 참조를 남긴다.

```text
References: civetweb/civetweb#1385, civetweb/civetweb#1392,
civetweb/civetweb#1400, civetweb/civetweb#1412
```

다른 사람의 `Signed-off-by`는 대신 작성하지 않는다.

### 검증

```bash
git diff --check
git diff -- src/civetweb.c
rg -n "mg_strcasecmp\(cl|strtoll\(cl|endptr == cl" src/civetweb.c
make clean
make build
git status --short --branch
```

수용 기준:

- `get_request()`에서 제거된 `cl` 참조가 없어야 한다.
- 기본 Make 빌드가 성공해야 한다.
- `h_zip` 미사용 경고와 요청 프레이밍 block의 컴파일 오류가 없어야 한다.
- source diff는 `src/civetweb.c`의 승인된 block으로 한정되어야 한다.

### 커밋

소스 branch:

```text
Task #2 Stage 1: get_request 컴파일 오류 수정
```

문서 branch에서 `task-stage-report` 승인 절차로 별도 생성:

```text
Task #2 Stage 1: 최소 컴파일 수정 완료보고서
```

Stage 1 소스 commit과 완료보고서 commit을 만든 뒤 작업지시자 승인 전에는 Stage 2로 진행하지 않는다.

## Stage 2 — 요청 프레이밍 회귀 테스트

### 산출물

소스 수정:

- `unittest/private.c`

개인 운영 문서:

- `mydocs/working/task_m117_2_stage2.md`

조건부 변경 후보이며 이번 구현계획에는 승인하지 않음:

- `unittest/public_server.c`
- `unittest/CMakeLists.txt`

### 변경 내용

1. `unittest/private.c`에 각 요청마다 새 `mg_context`, `mg_connection`, mutable request buffer와 error buffer를 구성하는 작은 test helper를 추가한다.
2. fixture는 다음 필드를 명시적으로 설정한다.
   - `conn.phys_ctx = &ctx`
   - `conn.dom_ctx = &ctx.dd`
   - `conn.buf`와 `conn.buf_size`
   - 완성된 HTTP 헤더 길이인 `conn.data_len`
3. Host header가 없는 완성 요청을 사용해 `switch_domain_context()`의 외부 domain 설정과 socket read를 피한다. `read_message()`가 미리 채운 buffer에서 헤더 길이를 얻도록 하며 실제 네트워크 I/O를 사용하지 않는다.
4. `START_TEST(test_get_request_framing)`을 만들고 기존 `Private / HTTP Message` TCase에 추가한다. 새 CTest case 이름이나 `unittest/CMakeLists.txt` 변경은 만들지 않는다.
5. 다음 요청과 결과를 독립 fixture로 검증한다.
   - `Transfer-Encoding: chunked`: 성공, `is_chunked == 1`, `content_len == 0`
   - `Transfer-Encoding: identity` + `Content-Length: 7`: 성공, `is_chunked == 0`, `content_len == 7`, request info 길이 7
   - `Transfer-Encoding: chunked` + `Content-Length: 7`: 실패, error 400
   - `Transfer-Encoding: gzip`: 실패, error 400
   - `Content-Length: abc`: 실패, error 411
   - `Content-Length: -1`: 실패, error 411
6. 새 test helper와 주석은 영어로 작성하고 기존 Check assertion 스타일을 따른다.

### 검증

```bash
cmake -S . -B output -DCIVETWEB_BUILD_TESTING=ON
cmake --build output --parallel 2
ctest --test-dir output -R '^test-private-http-message$' --output-on-failure
ctest --test-dir output --output-on-failure
git diff --check
git diff -- unittest/private.c
git status --short --branch
```

수용 기준:

- 여섯 프레이밍 조합의 성공·실패, error code와 connection 상태가 명시적으로 검증되어야 한다.
- targeted test와 전체 CTest가 모두 통과해야 한다.
- 제품 API나 production 코드를 test 편의를 위해 추가 변경하지 않아야 한다.
- private fixture로 안정적인 검증이 불가능하면 `public_server.c`로 전환하지 않고 실패 증거와 필요한 범위를 먼저 보고한다.

### 커밋

소스 branch:

```text
Task #2 Stage 2: 요청 프레이밍 회귀 테스트 추가
```

문서 branch에서 `task-stage-report` 승인 절차로 별도 생성:

```text
Task #2 Stage 2: 요청 프레이밍 회귀 테스트 완료보고서
```

Stage 2 소스 commit과 완료보고서 commit을 만든 뒤 작업지시자 승인 전에는 Stage 3으로 진행하지 않는다.

## Stage 3 — 통합 빌드 검증과 fork 게시 준비

### 산출물

소스 변경 없음:

- build output과 Docker image는 검증용 로컬 산출물이며 commit하지 않는다.

개인 운영 문서:

- `mydocs/working/task_m117_2_stage3.md`

### 변경 내용

1. 소스 worktree에서 생성된 이전 Make·CMake 산출물을 정리하고 기본 및 전체 기능 빌드를 각각 깨끗한 상태로 실행한다.
2. CMake test build와 전체 CTest를 다시 실행한다.
3. 개인 운영 symlink가 없는 `/home/edward/vsworks/myweb/civetweb-task2`를 Docker context로 사용한다.
4. Dockerfile 변경 없이 `civetweb:task2` 이미지를 빌드한다.
5. `upstream/master...local/task2`의 commit·파일 diff와 모든 worktree 상태를 확인한다.
6. fork 게시 후보는 소스 branch의 Stage 1·2 commit만 포함하도록 정리한다. `mydocs/` commit은 `personal/task2`에 남긴다.
7. 이 Stage에서는 push, PR 생성, upstream 댓글을 실행하지 않는다.

### 검증

```bash
make clean
make build
make clean
make WITH_ALL=1
cmake -S . -B output -DCIVETWEB_BUILD_TESTING=ON
cmake --build output --parallel 2
ctest --test-dir output --output-on-failure
docker build --no-cache --progress=plain -t civetweb:task2 .
git diff --check
git diff --name-only upstream/master...local/task2
git log --oneline upstream/master..local/task2
git status --short --branch
```

추가 상태 확인:

```bash
git -C /home/edward/vsworks/myweb/civetweb status --short --branch
git -C /home/edward/vsworks/myweb/civetweb-workflow status --short --branch
git -C /home/edward/vsworks/myweb/civetweb-workflow-task2 status --short --branch
```

수용 기준:

- Make 기본·전체 기능 빌드, CMake build, 전체 CTest와 Docker build가 모두 성공해야 한다.
- source branch diff에는 `src/civetweb.c`와 `unittest/private.c`만 있어야 한다.
- source commit은 Stage 1·2 두 개여야 하며 운영 문서가 포함되지 않아야 한다.
- 기존 Task #1과 main worktree는 깨끗하고 기존 branch를 유지해야 한다.
- upstream의 관련 PR 상태가 바뀌었으면 최종 보고 전에 다시 평가하되 외부 상태만으로 이 Stage 결과를 변경하지 않는다.

### 커밋

Stage 3은 검증 단계이므로 source branch에 빈 commit을 만들지 않는다. 문서 branch에서 `task-stage-report` 절차로 검증 결과만 commit한다.

```text
Task #2 Stage 3: 통합 빌드 검증 완료보고서
```

Stage 3 완료보고서 승인 전에는 최종 결과보고서, push 또는 fork 내부 PR을 만들지 않는다.

## Stage 4 — 기존 PR의 CI 실행 기반 복구

PR #3 게시 후 확인된 CI 실패를 같은 Task와 PR 안에서 복구한다. 이 단계는 별도 이슈나 별도 PR을 만들지 않는다.

### 산출물

기존 source branch와 PR #3:

- `.github/workflows/cibuild.yml`의 macOS OpenSSL 3 전환

개인 운영 branch:

- `.github/workflows/cibuild.yml`의 `personal/hyper-waterfall` push 제외
- source branch와 동일한 macOS OpenSSL 3 전환
- `mydocs/working/task_m117_2_stage4.md`
- 갱신된 최종 결과보고서와 오늘할일

### 변경 내용

1. source branch의 macOS NoDynLoad matrix를 OpenSSL 1.1에서 OpenSSL 3으로 바꾼다.
2. 중복된 `OSX-Package_OpenSSL_1_1` matrix 항목과 macOS OpenSSL 1.1 setup step을 제거한다.
3. OpenSSL 3 setup 주석은 현재 Homebrew 상태를 설명하는 영어로 수정한다.
4. 개인 branch에는 위 변경과 함께 `push.branches-ignore: personal/hyper-waterfall`을 추가한다.
5. branch 제외 조건은 source branch와 PR #3에 포함하지 않는다.
6. CIFuzz, Linux matrix와 `fail-fast`는 변경하지 않는다.

### 검증

```bash
git diff --check
actionlint .github/workflows/cibuild.yml  # 설치되어 있는 경우
git diff -- .github/workflows/cibuild.yml
git status --short --branch
```

게시 뒤에는 새 PR을 만들지 않고 기존 PR #3의 checks와 개인 branch push event만 확인한다.

### 커밋

source branch와 기존 PR #3:

```text
Task #2 Stage 4: macOS CI를 OpenSSL 3으로 전환
```

개인 운영 branch:

```text
Task #2 Stage 4: 개인 branch CI 제외와 결과 기록
```

## 통합 검증

- 각 Stage 검증 명령은 해당 단계 보고서 작성 전에 실행한다.
- 실패한 검증은 완료로 처리하지 않고 같은 Stage 안에서 원인을 분리한다.
- 새 네트워크 의존성 다운로드가 필요한 CMake test와 Docker build는 승인된 명령으로 실행하고 실제 결과를 보고서에 남긴다.
- 테스트 fixture, 변경 파일 또는 Stage 분할을 바꿔야 하면 구현계획서를 먼저 갱신해 재승인받는다.
- 최종 diff는 `upstream/master` 기준으로 검증하며 fork 운영 문서 diff와 분리한다.

## 커밋 및 저작자 정책

소스 branch 예상 commit:

1. `Task #2 Stage 1: get_request 컴파일 오류 수정`
2. `Task #2 Stage 2: 요청 프레이밍 회귀 테스트 추가`
3. `Task #2 Stage 4: macOS CI를 OpenSSL 3으로 전환`

문서 branch 예상 commit:

1. 기존 `77313710 Task #2: 수행 계획서 작성과 오늘할일 갱신`
2. `Task #2: 구현 계획서 작성`
3. 각 Stage의 완료보고서 commit
4. 향후 최종 결과보고서와 orders 상태 갱신 commit

Stage 1 commit은 결합 patch이므로 기존 두 저작자를 `Co-authored-by`로 표시하고 네 upstream PR을 본문에서 참조한다. Stage 2의 신규 회귀 테스트는 현재 작업자가 작성한다. 어느 branch도 승인 전에 원격 push하지 않는다.

## 단계 의존성

- Stage 1은 본 구현계획 승인 후에만 시작한다.
- Stage 2는 Stage 1 검증·완료보고서와 작업지시자 승인 후 시작한다.
- Stage 3은 Stage 2 검증·완료보고서와 작업지시자 승인 후 시작한다.
- 최종 결과보고서는 Stage 3 완료보고서 승인 후 별도 절차로 작성한다.
- `publish/task2` push와 fork 내부 PR은 최종 결과보고서 승인 후 별도 승인 게이트에서 수행한다.
- Stage 4는 게시된 PR의 CI 실패를 근거로 작업지시자가 범위를 보정한 재개 단계다. 새 이슈·브랜치·PR을 만들지 않는다.

## 위험과 대응

- **기존 patch 출처 혼합**: 첫 핵심 수정과 Zlib 선언 보정의 출처를 commit 본문과 co-author trailer로 분리해 기록한다.
- **불필요한 cherry-pick 범위**: 기존 commit을 통째로 가져오지 않고 승인된 `get_request()` block만 수동 병합한다.
- **test fixture의 숨은 socket 접근**: `data_len`에 완성 헤더를 미리 넣어 `read_message()`가 `pull_inner()`에 진입하지 않게 하고 targeted test로 먼저 확인한다.
- **C 문자열 parser의 buffer 변경**: 각 assertion마다 새 mutable buffer와 connection을 사용해 이전 parsing 결과가 다음 case에 영향을 주지 않게 한다.
- **CI와 로컬 환경 차이**: GCC 기반 Make, CMake/CTest, Alpine Docker를 함께 통과시키고 upstream PR CI 결과와 비교한다.
- **검증 산출물 혼입**: `output/`, Make 산출물, Docker image를 commit하지 않고 source diff 파일 목록을 강제 확인한다.
- **문서 branch 병합 충돌**: Task #1과 Task #2의 orders 행을 각 branch에서 보존하고 `personal/hyper-waterfall` 통합은 별도 승인 후 수행한다.
- **CI 복구의 재귀적 PR 생성**: 실패 workflow를 고치는 별도 PR을 만들지 않고 기존 PR #3에 commit을 추가한다.
- **개인 branch filter 혼입**: source branch에는 OpenSSL 3 변경만 적용하고 `personal/hyper-waterfall` 제외는 개인 branch에만 적용한다.

## 승인 요청 사항

- Stage 1에서 PR #1385의 핵심 수정과 PR #1392의 Zlib 조건부 선언만 수동 병합하고 두 저작자를 `Co-authored-by`로 표시하는 방식을 승인 요청한다.
- Stage 2에서 network 없는 private `get_request()` fixture와 여섯 요청 프레이밍 case를 추가하는 설계를 승인 요청한다.
- Stage 3을 source 변경 없는 통합 검증 단계로 두고 빈 source commit을 만들지 않는 방식을 승인 요청한다.
- 각 Stage마다 source와 개인 운영 문서를 별도 commit하며 완료보고서 승인 전 다음 Stage로 넘어가지 않는 절차를 승인 요청한다.
- 최종 결과보고서 승인 전에는 push나 PR을 만들지 않고, 이후에도 fork 내부 `edwardkim/civetweb:master`만 우선 대상으로 하는 범위를 승인 요청한다.
