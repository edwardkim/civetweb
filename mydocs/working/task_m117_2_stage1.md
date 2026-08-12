# Task #2 Stage 1 완료보고서 — 최소 컴파일 수정과 출처 보존

GitHub Issue: [#2](https://github.com/edwardkim/civetweb/issues/2)
구현계획서: [`task_m117_2_impl.md`](../plans/task_m117_2_impl.md)
Stage: 1
소스 commit: `8629836d2ad1c06d4f6b5a9bf16d90d1efa11387`

## 단계 목적

`upstream/master` commit `588860e3`에서 컴파일을 막던 `get_request()`의 손상된 조건식과 제거된 `cl` 변수 참조를 승인된 최소 범위로 복원한다. 기본 Make 빌드에서 Zlib 비활성 시 발생할 수 있는 `h_zip` 미사용 경고도 조건부 선언으로 제거하고, 기존 upstream 패치 저작자와 참고 PR을 소스 commit에 보존한다.

## 산출물

| 파일 | 변경 요약 |
|---|---|
| `src/civetweb.c` | `get_request()`의 header별 변수, 조건식, 들여쓰기를 복원하고 Zlib 전용 변수를 조건부 선언함. 14줄 추가, 12줄 삭제 |

소스 commit:

```text
8629836d Task #2 Stage 1: get_request 컴파일 오류 수정
```

저작자 및 참고 정보:

- `Co-authored-by: V.Shkriabets <vshcryabets@gmail.com>` — PR #1385 핵심 변수·괄호 수정
- `Co-authored-by: yubiuser <github@yubiuser.dev>` — PR #1392 Zlib 조건부 선언 보정
- commit 본문에 PR #1385, #1392, #1400, #1412를 참조함
- 다른 저작자의 `Signed-off-by`는 대신 작성하지 않음

## 본문 변경 정도 / 본문 무손실 여부

공개 API, 함수 signature, 요청 프레이밍 정책과 주석은 변경하지 않았다. `Transfer-Encoding` 비교에는 `h_chunk`, `Content-Length` 변환과 종료 포인터 검사에는 `h_len`을 사용하도록 손상 전 의도를 복원했다.

PR #1392와 #1412에 함께 포함된 `src/handle_form.inl`, SSL 형 변환, CI 변경은 가져오지 않았다. Stage 1 source diff는 `src/civetweb.c`의 `get_request()` block 한 곳으로 한정된다.

## 검증 결과

실행 명령:

```bash
git diff --check
git show --check --stat --oneline HEAD
sed -n '19159,19248p' src/civetweb.c \
  | rg -n "mg_strcasecmp\(cl|strtoll\(cl|endptr == cl"
make clean && make build
git diff --name-only upstream/master...HEAD
git status --short --branch
git log --format='%H%n%B' -1
```

결과:

- OK — `git diff --check`와 source commit의 whitespace 검사가 경고 없이 통과함.
- OK — `get_request()` 범위에서 제거된 `cl` 변수 참조가 검출되지 않음. `get_response()`의 정상적인 로컬 `cl` 참조는 변경하지 않음.
- OK — `make clean && make build`가 exit code 0으로 완료됨.
- OK — `src/civetweb.c`와 `src/main.c` 컴파일 및 `civetweb` 링크가 완료됨.
- OK — 기존 `expected ';'`, `cl undeclared`, `else without a previous if` 오류와 `h_zip` 미사용 경고가 출력되지 않음.
- OK — `upstream/master...HEAD` 변경 파일은 `src/civetweb.c` 한 개임.
- OK — source worktree는 `local/task2...upstream/master [ahead 1]`이고 tracked 변경이 없음.
- OK — commit 본문에 승인된 두 `Co-authored-by` trailer와 네 upstream PR 참조가 존재함.

## 잔여 위험

- Stage 1은 컴파일 회복만 검증했다. `Transfer-Encoding`과 `Content-Length` 조합의 실제 허용·거부 동작은 아직 회귀 테스트로 고정하지 않았다.
- Make 기본 빌드만 실행했다. 전체 기능, CMake/CTest와 Docker 검증은 승인된 이후 Stage에서 수행한다.
- fork가 upstream보다 한 commit 앞선 상태이므로 upstream 관련 PR의 merge 여부와 충돌 가능성을 최종 게시 전 다시 확인해야 한다.

## 다음 단계 영향

- Stage 2는 source commit `8629836d`를 기준으로 `unittest/private.c`에 네트워크 없는 `get_request()` fixture를 추가한다.
- chunked 단독, identity와 Content-Length, 두 framing header 동시 사용, 지원하지 않는 encoding, 무효·음수 Content-Length의 여섯 경우를 검증한다.
- private fixture가 실제 socket read 없이 동작하지 않으면 테스트 파일 범위를 임의로 확장하지 않고 작업지시자에게 먼저 보고한다.

## 승인 요청

- Stage 1의 최소 source patch, 저작자 표시와 기본 Make 빌드 결과를 승인 요청한다.
- 승인되면 구현계획에 따라 Stage 2 요청 프레이밍 회귀 테스트로 진행한다.
