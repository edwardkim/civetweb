# AGENTS.md

본 저장소에서 작업하는 모든 코딩 에이전트(Codex, Claude Code 등)가 따르는 운영 규칙. 매 턴 시스템 프롬프트로 적재되므로 항상 필요한 정책·제약·인덱스만 둔다. 절차 상세는 매뉴얼·SKILL로 분리한다.

## 프로젝트 개요

본 저장소는 `edwardkim/civetweb` 포크에서 변경을 준비해 원본 `civetweb/civetweb`에 기여하는 CivetWeb 작업 공간이다. CivetWeb은 C/C++ 임베디드 웹 서버이며 원본 저장소의 기본 브랜치는 `master`다.

## 하이퍼-워터폴 핵심 규칙

이 프로젝트는 **하이퍼-워터폴** 방법론을 적용한다. 에이전트의 기본 동작(빠른 실행, 자율 수정)과 충돌하므로 반드시 숙지한다. 상세: [`agent_code_hyperfall_rule_conflict.md`](mydocs/manual/agent_code_hyperfall_rule_conflict.md).

- 소스 수정 전 반드시 작업지시자 승인 요청
- 작업은 GitHub Issue 기준으로 추적
- 새 기능, 버그 수정, 구조 변경은 `이슈 -> 브랜치 -> 오늘할일 -> 계획서 -> 구현 -> 검증 -> 최종 보고서 -> PR` 순서 절대 생략 금지
- 각 단계 완료 후 승인 없이 다음 단계 진행 금지
- 범위가 불명확하거나 기존 작업과 충돌할 가능성이 있으면 먼저 확인
- 사용자나 다른 작업자가 만든 변경은 되돌리지 않음
- 이슈 close는 작업지시자 승인 후 또는 PR merge 확인 후에만 수행
- 문서 수정은 기존 내용을 먼저 읽고 필요한 부분만 수정하며, 불가피할 때만 내용을 추가
- 작업 완료 후 다음 작업에 필요하지 않은 로컬/원격 부산물은 정리
- PR merge와 이슈 close 후에는 `master`로 돌아오고, 더 이상 필요 없는 `local/task{번호}` 브랜치와 임시 worktree를 정리

**승인 간주 조건**: 작업지시자가 같은 스레드에서 "계속 진행", "다음 단계 진행"처럼 명시 지시한 경우에만 해당 단계 승인으로 간주한다.

## 명명 규칙

- 마일스톤: `M{버전}` (예: M100=v1.0.0, M05x=v0.5.x). 문서 파일명은 `m{숫자}` 소문자 (예: `m100`)
- 브랜치: `local/task{이슈번호}` (작업), `publish/task{이슈번호}` (`master` 대상 PR 게시용)
- 커밋 메시지:
  - 기본형: `Task #{번호}: 내용`
  - 단계: `Task #{번호} Stage {N}: 내용`
  - 하위 단계: `Task #{번호} [Stage {N.M}]: 내용`
  - 보고서 묶음: `Task #{번호} Stage {N} + 최종 보고서: 내용`
- 문서 파일명: `task_{milestone}_{이슈번호}{_impl|_stage{N}|_report}?.md`. 신규 문서는 마일스톤 포함 형식 강제. 상세: [`document_structure_guide.md`](mydocs/manual/document_structure_guide.md)
- 모든 문서는 한국어 작성

## 핵심 강제 규칙 (변경 전 매뉴얼 확인 필수)

- `origin`은 개인 포크 `edwardkim/civetweb`, `upstream`은 원본 `civetweb/civetweb`으로 유지한다.
- 모든 기여 PR은 `civetweb/civetweb:master`를 대상으로 하며, 작업 브랜치는 최신 `upstream/master`를 기준으로 만든다.
- 이 저장소는 개발·릴리스 기준 브랜치가 모두 `master`이므로 `master`에서 `master`로 보내는 별도 릴리스 PR 절차는 적용하지 않는다.
- `.hyper-waterfall/`과 로컬에 적용된 하이퍼-워터폴 운영 파일은 개인 전용이며 CivetWeb 기여 커밋이나 PR에 포함하지 않는다.
- 하이퍼-워터폴 운영 파일과 `mydocs/`의 Git 진실 원천은 fork 전용 `personal/hyper-waterfall` 브랜치와 `../civetweb-workflow` worktree다. 현재 소스 worktree의 동일 경로는 해당 worktree를 가리키는 개인 전용 링크다.
- `mydocs/` 변경은 `../civetweb-workflow`에서 별도 커밋해 `origin/personal/hyper-waterfall`에만 push한다. 일반 SKILL의 "소스와 보고서를 같은 커밋에 포함" 지시는 이 개인 전용 구성에서는 적용하지 않는다.
- `personal/hyper-waterfall` 브랜치는 upstream PR의 head 또는 소스 작업 기준으로 사용하지 않는다. 소스 브랜치와 upstream PR에는 CivetWeb 변경만 포함한다.
- 모든 기여는 MIT 라이선스를 따른다. API 변경·멀티스레딩 영향·구조 변경·유지보수에 큰 영향을 주는 변경은 구현 전에 원본 저장소 이슈에서 논의한다.
- PR에는 변경 의도, 예상되는 부작용과 검증 결과를 구체적으로 기록하고 CivetWeb CI 통과 여부를 확인한다.

## 필수 참조 문서

- [`README.md`](README.md) — 프로젝트 개요, 초기 설정, 빌드
- [`mydocs/manual/document_structure_guide.md`](mydocs/manual/document_structure_guide.md) — `mydocs/` 폴더 역할, 문서 파일명, 외부 PR 폴더 정책, Skills 위치 정책
- [`mydocs/manual/task_workflow_guide.md`](mydocs/manual/task_workflow_guide.md) — 타스크 진행 15단계, 커밋 메시지 규칙, 작업 시간 규칙
- [`mydocs/manual/git_workflow_guide.md`](mydocs/manual/git_workflow_guide.md) — 브랜치 정책, Git 다이어그램, 메인테이너/컨트리뷰터 워크플로우
- [`mydocs/manual/pr_process_guide.md`](mydocs/manual/pr_process_guide.md) — 외부 기여 PR 검토
- [`mydocs/manual/agent_code_hyperfall_rule_conflict.md`](mydocs/manual/agent_code_hyperfall_rule_conflict.md) — 하이퍼-워터폴과 에이전트 기본 동작 충돌 규칙
- [`docs/Contribution.md`](docs/Contribution.md) — CivetWeb 기여 조건과 사전 이슈 논의가 필요한 변경 범위
- [`docs/Building.md`](docs/Building.md) — Make 기반 빌드 방법
- [`docs/Docker.md`](docs/Docker.md) — Docker 이미지 빌드 및 실행 방법
- [`SECURITY.md`](SECURITY.md) — 보안 취약점 비공개 제보 절차

## Agent Skills

하이퍼-워터폴 절차의 정형 시점은 SKILL로 분리한다. 진실 원천은 `mydocs/skills/`이며, Codex(`.agents/skills`)와 Claude Code(`.claude/skills`)는 심볼릭 링크로 동일 본문을 인식한다. 상세: [`document_structure_guide.md`](mydocs/manual/document_structure_guide.md) 의 "Agent Skills 위치 정책".

## 작업 규칙

- 작업 시간의 시작과 종료는 작업지시자가 결정한다. 에이전트가 임의로 작업 종료를 제안하거나 시간을 한정하지 않는다.
