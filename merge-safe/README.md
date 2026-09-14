# merge-safe

master에서 딴 기능 브랜치를 test 같은 통합 브랜치에도 넣어야 하는데 충돌이 날 때, **기능 브랜치를 오염시키지 않고** 충돌을 푸는 절차입니다.

## 뭘 하는지

```
master ──●── feature (그대로 유지) ───────────────→ master PR
                     \
origin/test ──●───────●── feature-to-test (충돌은 여기서만) → test PR
```

1. `git merge-tree`로 작업트리를 건드리지 않고 test·master 양쪽 충돌을 미리 봅니다
2. `origin/test`에서 `--no-track` 브랜치를 **별도 worktree**로 땁니다
3. 거기서 기능 브랜치를 merge하고 충돌을 풉니다
4. 합쳐진 상태로 타입 체크 후 push, test로 PR
5. 기능 브랜치가 바뀌면 원본에 커밋 → to-test에 다시 merge

## 왜 이렇게 설계했나

### 흔한 해결법이 원본을 망가뜨린다

충돌이 나면 보통 기능 브랜치에서 `git merge origin/test`를 합니다. 그런데 test에는 운영에 안 나갈 다른 기능도 들어 있어서, 이 순간 그 코드가 전부 기능 브랜치로 들어오고 master PR에 같이 실립니다. 그래서 충돌 해결 장소를 통합 브랜치 쪽 복제 브랜치로 옮겼습니다.

### worktree + `--no-track`

- **worktree**: 원본 작업트리에 커밋 안 된 변경이 있어도 stash·checkout 없이 그대로 둡니다.
- **`--no-track`**: `origin/test`를 upstream으로 잡으면 무심코 친 `git push`가 test에 바로 올라갈 수 있습니다.

### 방향은 한쪽으로만

수정은 항상 원본 기능 브랜치에서 하고, to-test는 받기만 합니다. 반대로 흐르면 master에 들어갈 브랜치에 test 코드가 섞입니다.

## 쓰는 법

```
test에 넣으려는데 충돌 나. 원본 브랜치는 건드리면 안 돼
기능 브랜치 고쳤으니 to-test에도 다시 반영해줘
```

## 안 쓰는 경우

- 통합 브랜치 없이 master로만 배포하는 흐름
- 충돌이 없어서 바로 PR하면 되는 경우
