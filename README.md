<div align="center">

# my-claude-skills

**Claude Code에서 반복해서 쓰는 워크플로우를 재사용 가능한 스킬로 정리한 개인 마켓플레이스**

한 번 쓰고 버리는 프롬프트 대신, 작업의 순서·형식·검증 기준을 문서로 못 박아<br/>다음에도 같은 품질로 재현되게 만드는 것이 목적입니다.

**[등록된 스킬 둘러보기 →](https://my-claude-skills-site.vercel.app)**

</div>

---

## 설치

이 저장소 자체가 Claude Code 플러그인 마켓플레이스입니다. 한 번 등록해두면 원하는 스킬만 골라 설치할 수 있습니다.

```bash
/plugin marketplace add Jeric1223/my-claude-skills
/plugin install travel-itinerary@jeric-skills
```

설치된 스킬은 `/plugin`으로 둘러보고, 새 스킬이 추가되면 `/plugin marketplace update jeric-skills`로 갱신합니다.

## 스킬

| 스킬 | 설치 | 하는 일 |
| :--- | :--- | :--- |
| **[travel-itinerary](./travel-itinerary-skill)** | `travel-itinerary@jeric-skills` | 목적지·날짜·인원만 주면 검증된 정보, 구글맵 링크, 예약 체크리스트가 포함된 일차별 여행 일정표를 만듭니다 |
| **[session-to-skill](./session-to-skill)** | `session-to-skill@jeric-skills` | 진행 중인 대화에서 재사용할 만한 패턴을 찾아 제안하고, 선택한 것을 이 저장소 컨벤션에 맞는 스킬로 등록한 뒤, 원하면 민감정보를 검사하고 이 저장소로 PR까지 열어줍니다 |
| **[aws-helper](./aws-helper)** | `aws-helper@jeric-skills` | AWS를 조회하기 전에 실측 권한 캐시로 자격증명 프로파일을 고르고, 쓰기 작업은 실행하지 않고 명령어와 롤백 방법만 만들어 넘깁니다 |
| **[postman-collection-from-code](./postman-collection-from-code)** | `postman-collection@jeric-skills` | API를 만들거나 고친 뒤 라우터·검증 스키마·enum·인증 미들웨어를 소스에서 읽어, 파라미터마다 허용값이 적힌 Postman 컬렉션 JSON을 만들거나 기존 컬렉션을 갱신합니다 |
| **[readme-screenshots](./readme-screenshots)** | `readme-screenshots@jeric-skills` | 앱을 실제로 띄워 고정된 뷰포트·테마로 화면을 캡처하고, 실명·토큰이 찍혔는지 검사한 뒤 README에 상대경로 + 확대 링크 형태로 삽입하거나 갱신합니다 |
| **[dev-start](./dev-start)** | `dev-start@jeric-skills` | 티켓 하나를 이슈 생성 → 브랜치 → 설계 승인 → 구현 → 검증 → Postman 컬렉션 → 변경 내역·직접 할 일 정리까지 같은 순서로 진행합니다 |
| **[merge-safe](./merge-safe)** | `merge-safe@jeric-skills` | master 기준 기능 브랜치를 test 같은 통합 브랜치에 넣을 때, 충돌을 별도 worktree 브랜치에서만 풀어 기능 브랜치에 미출시 코드가 섞이지 않게 합니다 |
| **[blog-post](./blog-post)** | `blog-post@jeric-skills` | 레포의 git 히스토리에서 글로 남길 만한 마찰 지점(삽질·방향 전환·외부 제약)을 찾아 후보를 제안하고, 고른 주제로 실제 화면 캡처가 들어간 기술 블로그 초안을 만든 뒤 AI 티를 벗깁니다 |
| **[mentor-mode-teaching](./mentor-mode-teaching)** | `mentor-mode-teaching@jeric-skills` | 사수 역할로 새 기술의 개념을 먼저 설명하고 예측 질문을 던진 뒤, 맞은 부분·교정·용어표·면접용 한 문장 형식으로 피드백하며 프로젝트를 같이 진행합니다 |
| **[save-learning-log](./save-learning-log)** | `save-learning-log@jeric-skills` | 학습 세션을 끝낼 때 다음에 할 일·숙제·미답변 질문·날짜별 Q&A 요약을 프로젝트 `MEMORY.md`에 남기고, 다음 세션이 그 지점부터 이어가도록 `CLAUDE.md`에 연결합니다 |

## 설계 원칙

스킬을 여러 개 만들면서 정착한 네 가지 규칙입니다. 새 스킬은 전부 이걸 따릅니다.

#### 1. 환각을 구조로 막는다

시간에 따라 바뀌는 사실(영업시간, 정기휴무일, 계절 이벤트, 요금)은 모델의 사전지식이 아니라 실시간 검색으로 검증합니다. 확인이 안 되면 그럴듯하게 채우지 않고 **"확인 필요"라고 남깁니다.** 빈칸으로 두는 편이 틀린 값보다 낫습니다.

#### 2. 결정론적 계산은 LLM에 맡기지 않는다

예약 마감일을 "출발 3주 전"처럼 상대 표현으로 적으면 쓸 때마다 사람이 다시 계산해야 하고, 모델이 계산하면 틀립니다. 출발일 기준으로 역산한 **실제 날짜로 못 박습니다.**

#### 3. 출력 형식을 템플릿으로 강제한다

매번 다른 모양으로 나오면 재사용이 안 됩니다. 출력 구조를 `template.md`로 분리해 스킬이 그 골격을 채우게 합니다.

#### 4. 모호하면 넘겨짚지 않고 묻는다

여행 성격, 동행 유형, 예산 감각처럼 결과물이 크게 갈리는 조건은 구조화된 질문으로 먼저 확인합니다. 잘못된 가정 위에 쌓인 결과물은 고치는 것보다 다시 만드는 게 빠릅니다.

## 왜 만들었나

Claude와 여행 일정을 짜봤는데 결과물은 만족스러웠지만 **재현성이 없었습니다.** 다음 여행에 같은 품질을 얻으려면 이전 대화의 맥락과 요령을 처음부터 다시 설명해야 했습니다.

그때 실제로 걸렸던 문제가 두 개 있었습니다. 개화 시기 같은 시효성 있는 정보를 확인 없이 단정한 것, 그리고 예약 마감일을 상대 표현으로만 적어서 나중에 다시 계산해야 했던 것입니다. 이런 걸 규칙으로 못 박아 스킬로 만들면 같은 실수를 반복하지 않고 결과물 형식도 일관되게 유지됩니다 — 위 설계 원칙 1번과 2번이 여기서 나왔습니다.

## 스킬 구조

새 스킬은 저장소 루트 바로 밑에 폴더를 만들고 아래 형식을 따릅니다.

```
skill-name/
├── SKILL.md      # 필수 — 트리거 조건, 절차, 흔한 실수 (에이전트가 읽는 파일)
├── README.md     # 필수 — 사람이 읽는 설명: 뭘 하는지, 왜 이렇게 설계했는지
└── template.md   # 선택 — 출력 형식이 고정되어야 하는 경우의 스켈레톤
```

`SKILL.md`와 `README.md`의 역할은 섞지 않습니다. `SKILL.md`는 에이전트가 실행 중에 참조하는 절차 문서이고, `README.md`는 저장소를 훑어보는 사람을 위한 설명입니다.

폴더 루트에 `SKILL.md`가 있고 `skills/` 하위 폴더가 없으면 Claude Code가 이를 **단일 스킬 플러그인**으로 자동 인식합니다. 그래서 폴더마다 `plugin.json`을 둘 필요가 없습니다.

폴더를 만든 뒤 루트 [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json)의 `plugins` 배열에 항목을 추가해야 설치 대상이 됩니다. 이 파일이 마켓플레이스의 **유일한 레지스트리**입니다.

```json
{
  "name": "설치 이름 (kebab-case)",
  "source": "./skill-name",
  "description": "언제 쓰는 스킬인지",
  "version": "1.0.0",
  "author": { "name": "Jeric1223" }
}
```

`name`은 폴더명과 달라도 됩니다 — 사용자가 `/plugin install <name>@jeric-skills`에 입력하는 이름입니다.

## 기여

다른 프로젝트에서도 통할 만한 스킬이라면 PR을 환영합니다. 폴더 구조, 버전 규칙, 리뷰 기준은 [CONTRIBUTING.md](./CONTRIBUTING.md)에 있습니다.

## 이 프로젝트를 만들며

매번 같은 요청을 새 세션에서 다시 설명하던 문제를, 프롬프트가 아니라 절차를 파일로 못 박는 방향으로 푼 과정을 글로 남겼습니다.

→ [매 세션 똑같은 설명을 반복하고 있다면, 프롬프트 문제가 아닙니다]([https://velog.io/@hoohoo0889/%EB%A7%A4-%EC%84%B8%EC%85%98-%EB%98%91%EA%B0%99%EC%9D%80-%EC%84%A4%EB%AA%85%EC%9D%84-%EB%B0%98%EB%B3%B5%ED%95%98%EA%B3%A0-%EC%9E%88%EB%8B%A4%EB%A9%B4-%ED%94%84%EB%A1%AC%ED%94%84%ED%8A%B8-%EB%AC%B8%EC%A0%9C%EA%B0%80-%EC%95%84%EB%8B%99%EB%8B%88%EB%8B%A4](https://velog.io/@hoohoo0889/%EB%A7%A4-%EC%84%B8%EC%85%98-%EB%98%91%EA%B0%99%EC%9D%80-%EC%84%A4%EB%AA%85%EC%9D%84-%EB%B0%98%EB%B3%B5%ED%95%98%EA%B3%A0-%EC%9E%88%EB%8B%A4%EB%A9%B4-%ED%94%84%EB%A1%AC%ED%94%84%ED%8A%B8-%EB%AC%B8%EC%A0%9C%EA%B0%80-%EC%95%84%EB%8B%99%EB%8B%88%EB%8B%A4))

---

<div align="center">
<sub>Not affiliated with Anthropic.</sub>
</div>
