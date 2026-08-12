# Task #2 Stage 2 완료보고서 — 요청 프레이밍 회귀 테스트

GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
구현계획서: [`task_m117_2_impl.md`](../plans/task_m117_2_impl.md)
Stage: 2
소스 commit: `783c167681642eb50be20a6593424e5be61a68f3`

## 단계 목적

Stage 1에서 복원한 `get_request()`의 요청 프레이밍 처리를 네트워크 I/O 없는 private fixture로 검증한다. `Transfer-Encoding`과 `Content-Length`의 정상 조합과 충돌·무효 입력을 회귀 테스트로 고정해 이후 upstream 동기화나 추가 수정에서 동작이 다시 손상되는 것을 탐지한다.

## 산출물

| 파일 | 변경 요약 |
|---|---|
| `unittest/private.c` | 독립적인 `get_request()` fixture와 요청 프레이밍 6개 경우를 추가하고 기존 `Private / HTTP Message` TCase에 등록함. 90줄 추가 |

소스 commit:

```text
783c1676 Task #2 Stage 2: 요청 프레이밍 회귀 테스트 추가
```

추가한 검증 경우:

1. `Transfer-Encoding: chunked` — 성공, chunked 상태와 본문 길이 확인
2. `Transfer-Encoding: identity`와 `Content-Length: 7` — 성공, connection과 request info 길이 확인
3. `Transfer-Encoding: chunked`와 `Content-Length: 7` — error 400으로 거부
4. `Transfer-Encoding: gzip` — error 400으로 거부
5. `Content-Length: abc` — error 411로 거부
6. `Content-Length: -1` — error 411로 거부

## 본문 변경 정도 / 본문 무손실 여부

production source, 공개 API와 요청 처리 정책은 변경하지 않았다. 각 test 호출은 새 `mg_context`, `mg_connection`, mutable request buffer와 error buffer를 사용하며 실제 socket read 없이 완성된 HTTP header를 `get_request()`에 전달한다.

구현계획에서 승인하지 않은 조건부 후보 `unittest/public_server.c`와 `unittest/CMakeLists.txt`는 변경하지 않았다. 새 주석이 필요한 production code 변경은 없었으며 test 식별자와 구현은 영어로 작성했다.

## 검증 결과

최종 CMake 구성과 빌드:

```bash
cmake -S . -B output \
  -DCIVETWEB_BUILD_TESTING=ON \
  -DCIVETWEB_SSL_OPENSSL_API_1_1=OFF \
  -DCIVETWEB_SSL_OPENSSL_API_3_0=ON
cmake --build output --parallel 2
```

결과:

- OK — CMake 구성과 test binary 빌드가 exit code 0으로 완료됨.
- OK — 실행 환경의 OpenSSL 3.0.13에 맞춰 CMake의 OpenSSL 3 API 선택 항목을 사용함. 제품 파일과 CMake 설정 파일은 변경하지 않음.
- 참고 — build 중 `src/handle_form.inl`, 기존 SSL 형 변환과 기존 unittest 코드의 경고가 출력됐으나 이번 Stage diff 밖의 기존 경고이며 새 test code 관련 경고는 없음.

targeted test:

```bash
ctest --test-dir output \
  -R '^test-private-http-message$' \
  --output-on-failure
```

결과:

```text
100% tests passed, 0 tests failed out of 1
```

전체 CTest 준비와 실행:

```bash
cc unittest/cgi_test.c -o output/cgi_test.cgi
ctest --test-dir output --output-on-failure
```

결과:

```text
100% tests passed, 0 tests failed out of 50
Total Test time (real) = 352.08 sec
```

- OK — CTest 안내에 따라 CMake가 자동 생성하지 않는 CGI test helper를 로컬 build output에 준비함. 이 파일은 commit 대상이 아님.
- OK — 서버 port와 TLS가 필요한 전체 CTest는 해당 기능이 허용된 실행 환경에서 수행함.
- OK — 여섯 프레이밍 조합의 결과, error code, `is_chunked`, connection 길이와 request info 길이를 명시적으로 검증함.

변경 범위 점검:

```bash
git diff --check
git show --check --stat --oneline HEAD
git diff --name-only upstream/master...HEAD
git status --short --branch
git log --oneline upstream/master..HEAD
```

결과:

- OK — whitespace 검사가 경고 없이 통과함.
- OK — Stage 2 commit은 `unittest/private.c` 한 파일의 90줄 추가로 한정됨.
- OK — `upstream/master...HEAD` 전체 변경 파일은 Stage 1의 `src/civetweb.c`와 Stage 2의 `unittest/private.c` 두 개임.
- OK — source worktree는 `local/task2...upstream/master [ahead 2]`이고 tracked 변경이 없음.
- OK — source branch에는 계획한 Stage 1·2 commit 두 개만 존재함.

## 잔여 위험

- 테스트는 현재 복원된 요청 프레이밍 정책을 고정하지만 전체 기능 조합과 Docker image build는 아직 검증하지 않았다.
- CMake 기본값은 OpenSSL 1.1 API를 선택했으며 현재 host는 OpenSSL 3.0.13이다. Stage 3에서도 host와 일치하는 OpenSSL 3 CMake 선택 항목을 명시해야 전체 TLS test를 안정적으로 재현할 수 있다.
- `cgi_test.cgi`는 현재 CMake test build에서 자동 생성되지 않는다. Stage 3 전체 CTest 전에도 같은 로컬 helper 준비가 필요하다.
- upstream 관련 PR 상태와 fork branch의 최종 게시 적합성은 Stage 3에서 다시 확인한다.

## 다음 단계 영향

- Stage 3은 source 변경 없이 Make 기본·전체 기능 build, CMake/CTest와 Docker build를 깨끗한 상태에서 통합 검증한다.
- CMake/CTest는 host OpenSSL 3 선택 항목과 CGI helper 준비 조건을 재현해 실행한다.
- 최종 branch diff가 `src/civetweb.c`, `unittest/private.c`와 source commit 두 개로만 구성됐는지 확인한다.
- 개인 운영 문서 branch와 기존 Task #1 worktree가 변경되지 않았는지도 함께 점검한다.

## 승인 요청

- Stage 2의 private fixture, 여섯 요청 프레이밍 회귀 테스트와 CTest 50/50 통과 결과를 승인 요청한다.
- 승인되면 구현계획에 따라 Stage 3 통합 빌드 검증과 fork 게시 준비로 진행한다.
