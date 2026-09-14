---
name: merge-safe
description: Use when a feature branch cut from master/main must also go into a shared test/dev/staging branch and the merge conflicts — especially when that shared branch holds unreleased work that must not leak back into the feature branch
---

# merge-safe

## Overview

test 같은 통합 브랜치에는 운영에 안 나갈 코드도 섞여 있다. 기능 브랜치에서 `git merge origin/test`로 충돌을 풀면 **그 코드가 전부 기능 브랜치로 들어오고**, 나중에 master PR에 같이 실려 나간다.

핵심 원칙: **충돌은 기능 브랜치가 아니라 통합 브랜치에서 딴 별도 브랜치에서 푼다.** 기능 브랜치는 끝까지 master 기준 그대로 둔다.

```
master ──●── feature (그대로 유지) ───────────────→ master PR
                     \
origin/test ──●───────●── feature-to-test (충돌은 여기서만) → test PR
```

## When to Use

- 기능 브랜치를 test/dev에 PR하려는데 충돌이 난다
- "원본 브랜치 손상되면 안 돼", "test엔 운영에 안 나갈 것도 있어"

**Not for:** 통합 브랜치 없이 master로만 가는 흐름, 충돌이 없는 경우(그냥 PR).

## 절차

### 1. 먼저 읽기 전용으로 충돌을 본다

```bash
git fetch origin
git merge-tree --write-tree origin/test <feature-branch>   # exit 0 = 충돌 없음
git merge-tree --write-tree origin/master <feature-branch> # master 쪽도 같이 확인
```

작업트리를 전혀 건드리지 않는다. 충돌 파일 목록과 master 쪽이 깨끗한지를 먼저 사용자에게 보고한다.

### 2. 통합 브랜치에서 별도 브랜치를 worktree로 딴다

```bash
git worktree add --no-track -b <feature>-to-test ../<repo>-<feature>-to-test origin/test
```

- `--no-track`: `origin/test`를 upstream으로 잡으면 실수로 `git push`가 test에 직접 올라갈 수 있다.
- worktree를 쓰면 원본 작업트리(미커밋 변경 포함)를 stash·checkout 없이 그대로 둔다.
- 앱이 `.env`를 읽으면 원본에서 복사한다(gitignore라 worktree엔 없다). 의존성은 `node_modules` 심볼릭 링크 등으로 연결.

### 3. 거기서 기능 브랜치를 merge하고 충돌을 푼다

```bash
git -C ../<repo>-<feature>-to-test merge --no-edit <feature-branch>
```

충돌 해결 원칙:
- **통합 브랜치에만 있는 블록** → 그대로 살린다 (남의 작업)
- **내 기능 블록** → 살린다
- 한쪽이 **포매터 재정렬만** 한 경우 → 로직이 있는 쪽을 살리고 포맷 차이는 버린다
- 양쪽이 같은 줄을 실제로 다르게 바꿨으면 멈추고 사용자에게 묻는다

`git diff --check`, 충돌 마커 grep으로 남은 마커가 없는지 확인한다.

### 4. 검증하고 push

- 그 worktree에서 타입 체크/빌드를 **실제로 실행**한다. 통합 브랜치의 다른 코드와 섞였을 때 깨지는 게 여기서 잡힌다.
- `git push -u origin <feature>-to-test` → `<feature>-to-test` → `test`로 PR.
- 기능 브랜치는 `git log`로 통합 브랜치 커밋이 안 섞였는지 확인한다.

### 5. 기능 브랜치가 또 바뀌면

원본 브랜치에 커밋·푸시 → to-test worktree에서 **다시 merge** → 양쪽 모두 타입 체크 → push. 반대 방향(to-test → feature)으로는 절대 merge하지 않는다.

- worktree에 커밋 안 된 변경(포매터 등)이 있으면 커밋에 섞지 말고 보고한다.
- 푸시 후 보안 리뷰 같은 자동 경고가 오면 내 변경인지 통합 브랜치에 원래 있던 코드인지 구분해 보고한다.

### 6. 끝나면

test PR이 머지되면 `git worktree remove ../<repo>-<feature>-to-test`. 브랜치 삭제는 사용자 승인 후.

## Common Mistakes

| 실수 | 결과 |
| --- | --- |
| 기능 브랜치에서 `git merge origin/test` | 미출시 코드가 master PR에 같이 실림 |
| 기능 브랜치를 test 위로 rebase | 위와 같음 + 이미 push된 브랜치면 force push 필요 |
| `--no-track` 없이 origin/test에서 브랜치 생성 | `git push`가 test에 직접 올라갈 위험 |
| 원본 작업트리에서 checkout으로 전환 | 미커밋 변경 충돌·분실 |
| to-test 브랜치에서만 코드 수정 | 원본 브랜치와 갈라져 master엔 수정이 빠짐 — 수정은 항상 원본에서 |
| worktree에서 타입 체크 생략 | 합쳐진 상태에서만 나는 에러를 test 서버에서 발견 |
