---
name: aws-helper
description: Use before running any `aws` CLI command — when inspecting AWS resources, checking configuration, or advising on AWS setup (e.g. "이 람다 스펙 어떻게 잡지"). Picks the right credential profile from a measured capability cache, asks the user when the choice is ambiguous, and never executes writes. Triggered by requests like "AWS 좀 봐줘", "람다 설정 확인해줘", "이 테이블 스키마 뭐야", or any question answerable only by calling AWS.
---

# AWS Helper

## Overview

여러 자격증명 프로파일이 서로 다른 권한을 갖고 있을 때, **어떤 프로파일로 조회할지 먼저 정하고** AWS를 읽는다. 조회는 자유롭게 하되 **쓰기는 절대 실행하지 않고 명령어만 만들어 넘긴다.**

전제: 이 환경의 프로파일들은 **자기 IAM 정책을 읽을 수 없다** (`iam:ListAttachedUserPolicies` 전부 AccessDenied). 그래서 권한 정보는 API 조회가 아니라 **실제 명령을 던져본 결과를 누적한 캐시**에서만 나온다.

## When to Use

- AWS 리소스를 조회·확인해야 할 때 (람다 설정, DynamoDB 스키마, S3 버킷, 로그 등)
- 현재 AWS 구성을 파악해서 조언해야 할 때 ("이 람다 메모리 얼마로 잡는 게 좋냐")
- 사용자가 쓰기를 요청했을 때 — 실행하지 않고 명령어를 만들어주기 위해 진입

**Not for:** AWS를 전혀 건드리지 않는 일반 코드 작업, 이미 이 세션에서 프로파일이 확정됐고 같은 서비스를 계속 조회하는 경우(캐시대로 진행).

## 권한 캐시

`~/.claude/aws-profile-capabilities.json` — **레포 밖에 둔다.** 계정 ID와 사용자 ARN이 들어가므로 절대 커밋하지 않는다.

```json
{
  "<프로파일명>": {
    "account": "<계정 ID>",
    "arn": "arn:aws:iam::<계정 ID>:user/<사용자명>",
    "region": "ap-northeast-2",
    "allow": ["dynamodb:DescribeTable", "dynamodb:Scan", "dynamodb:UpdateTable"],
    "deny": ["iam:ListAttachedUserPolicies", "firehose:ListDeliveryStreams"],
    "checked_at": "2026-09-02"
  }
}
```

- `allow` / `deny`는 **액션 단위**로 기록한다 (`서비스:액션`). 서비스 단위로 뭉뜽그리지 않는다 — 같은 DynamoDB라도 Scan은 되고 UpdateTable은 안 되는 경우가 실제로 있다.
- 캐시에 없는 액션은 "권한 없음"이 아니라 **"모름"**이다. 둘을 절대 혼동하지 않는다.

## 절차

**1. 캐시를 읽는다.** 파일이 없으면 부트스트랩:

```bash
for p in $(grep -E '^\[' ~/.aws/credentials | tr -d '[]'); do
  ( echo "$p -> $(aws sts get-caller-identity --profile "$p" --query Arn --output text 2>&1 | tail -1)" ) &
done; wait
```

전체 프로파일 병렬로 약 6초. 결과로 `account` / `arn`을 채운다.

**2. 필요한 액션을 특정한다.** "람다 환경변수 확인" → `lambda:GetFunctionConfiguration`.

**3. 후보를 고른다.**

| 캐시 상태 | 행동 |
| --- | --- |
| `allow`에 그 액션이 있는 프로파일이 **정확히 1개** | 자동 선택. `<프로파일명>으로 조회합니다` 한 줄만 명시하고 진행 |
| 후보가 **2개 이상** | AskUserQuestion |
| 후보가 **0개** (기록 없음) | 4번 프로브 후 재판정 |
| 후보가 **0개** (전부 `deny`) | 프로파일 부족을 사용자에게 알린다. 임의로 다른 걸 시도하지 않는다 |

AskUserQuestion 선택지에는 **캐시의 실측 내용을 그대로** 넣는다 — 계정 번호, 확인된 액션, 막힌 액션. "아마 될 것 같다"를 쓰지 않는다.

**4. 빈칸만 프로브한다.** 그 액션 기록이 없는 프로파일에 한해, 부작용 없는 read 명령 1개를 병렬로 던져 결과를 캐시에 채운다. 이미 기록이 있으면 다시 던지지 않는다.

**5. 실행하고 캐시에 기록한다.** 성공 → `allow`, AccessDenied → `deny`, `checked_at` 갱신. **이 스킬 밖에서 실행한 aws 명령의 결과도 보이는 대로 기록한다.**

## 읽기 / 쓰기 판정

화이트리스트 방식. 아래로 시작하는 서브커맨드만 읽기로 보고, **나머지는 전부 쓰기로 간주한다.**

```
describe-*   get-*   list-*   scan   query   head-*   batch-get-*   lookup-*
```

예외로 읽기에 포함:
- `aws s3 ls`, `aws s3api head-object`
- `aws logs filter-log-events`, `aws logs tail`
- `aws sts get-caller-identity`
- `aws s3 cp <s3://...> <로컬경로>` — **다운로드 방향만.** 업로드는 쓰기다

애매하면 쓰기로 처리한다. 오탐(읽기를 쓰기로 봄)은 한 번 더 묻고 끝나지만, 반대는 되돌릴 수 없다.

## 쓰기 요청이 들어왔을 때

**실행하지 않는다.** 대신 이 세 가지를 낸다.

1. **쓰기 권한이 확인된 프로파일** — 캐시 기준. 없으면 "확인된 프로파일 없음"이라고 쓴다
2. **그대로 복붙할 명령어** — `--profile`, `--region` 포함한 완성형
3. **예상 영향과 롤백 방법** — 무엇이 바뀌고, 잘못됐을 때 어떻게 되돌리는지

콘솔에서 하는 게 나은 작업(IAM 정책 편집 등)이면 그렇게 말한다.

## 계정 분리 가드

프로파일이 **여러 AWS 계정에 걸쳐 있을 수 있다.** 캐시의 `account` 필드로 판별한다.

후보에 서로 다른 계정이 섞이면 **자동 선택하지 않고 반드시 묻는다.** 대화 맥락이 한쪽을 가리켜도 마찬가지다 — 계정을 잘못 짚으면 엉뚱한 환경을 보고 보고하게 된다.

**같은 IAM 사용자를 가리키는 프로파일**(흔한 예: `default`가 다른 프로파일의 별칭)은 캐시에서 `arn`이 같은 것으로 판별해 **후보로 중복 나열하지 않는다.**

## Common Mistakes

- **캐시에 없는 걸 "권한 없음"으로 단정** — 기록 없음은 모름이다. 프로브하거나 묻는다
- **AccessDenied를 보고 권한 문제로 결론** — 프로파일을 잘못 골랐을 가능성이 먼저다. 다른 후보를 확인한다
- **프로브를 매번 전 프로파일에 돌림** — 캐시에 빈칸인 것만 던진다
- **쓰기를 "확인차" 실행** — dry-run 옵션이 있어도 실행하지 않는다. 명령어만 낸다
- **자동 선택했으면서 어떤 프로파일을 썼는지 말하지 않음** — 결과의 신뢰도가 프로파일에 달려 있으므로 항상 명시한다
- **계정이 다른 프로파일을 맥락만 보고 자동 선택**
