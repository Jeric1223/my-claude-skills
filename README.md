<div align="center">

# my-claude-skills

**Claude Code에서 반복해서 쓰는 워크플로우를 재사용 가능한 스킬로 정리한 개인 마켓플레이스**

한 번 쓰고 버리는 프롬프트 대신, 작업의 순서·형식·검증 기준을 문서로 못 박아<br/>다음에도 같은 품질로 재현되게 만드는 것이 목적이다.

**[등록된 스킬 둘러보기 →](https://my-claude-skills-site.vercel.app)**

</div>

---

## 설치

이 저장소 자체가 Claude Code 플러그인 마켓플레이스다. 한 번 등록해두면 원하는 스킬만 골라 설치할 수 있다.

```bash
/plugin marketplace add Jeric1223/my-claude-skills
/plugin install travel-itinerary@jeric-skills
```

설치된 스킬은 `/plugin`으로 둘러보고, 새 스킬이 추가되면 `/plugin marketplace update jeric-skills`로 갱신한다.

## 스킬

| 스킬 | 설치 | 하는 일 |
| :--- | :--- | :--- |
| **[travel-itinerary](./travel-itinerary-skill)** | `travel-itinerary@jeric-skills` | 목적지·날짜·인원만 주면 검증된 정보, 구글맵 링크, 예약 체크리스트가 포함된 일차별 여행 일정표를 만든다 |
| **[session-to-skill](./session-to-skill)** | `session-to-skill@jeric-skills` | 진행 중인 대화에서 재사용할 만한 패턴을 찾아 제안하고, 선택한 것을 이 저장소 컨벤션에 맞는 스킬로 등록한다 |

## 설계 원칙

스킬을 여러 개 만들면서 정착한 네 가지 규칙이다. 새 스킬은 전부 이걸 따른다.

#### 1. 환각을 구조로 막는다

시간에 따라 바뀌는 사실(영업시간, 정기휴무일, 계절 이벤트, 요금)은 모델의 사전지식이 아니라 실시간 검색으로 검증한다. 확인이 안 되면 그럴듯하게 채우지 않고 **"확인 필요"라고 남긴다.** 빈칸으로 두는 편이 틀린 값보다 낫다.

#### 2. 결정론적 계산은 LLM에 맡기지 않는다

예약 마감일을 "출발 3주 전"처럼 상대 표현으로 적으면 쓸 때마다 사람이 다시 계산해야 하고, 모델이 계산하면 틀린다. 출발일 기준으로 역산한 **실제 날짜로 못 박는다.**

#### 3. 출력 형식을 템플릿으로 강제한다

매번 다른 모양으로 나오면 재사용이 안 된다. 출력 구조를 `template.md`로 분리해 스킬이 그 골격을 채우게 한다.

#### 4. 모호하면 넘겨짚지 않고 묻는다

여행 성격, 동행 유형, 예산 감각처럼 결과물이 크게 갈리는 조건은 구조화된 질문으로 먼저 확인한다. 잘못된 가정 위에 쌓인 결과물은 고치는 것보다 다시 만드는 게 빠르다.

## 왜 만들었나

Claude와 여행 일정을 짜봤는데 결과물은 만족스러웠지만 **재현성이 없었다.** 다음 여행에 같은 품질을 얻으려면 이전 대화의 맥락과 요령을 처음부터 다시 설명해야 했다.

그때 실제로 걸렸던 문제가 두 개 있었다. 개화 시기 같은 시효성 있는 정보를 확인 없이 단정한 것, 그리고 예약 마감일을 상대 표현으로만 적어서 나중에 다시 계산해야 했던 것. 이런 걸 규칙으로 못 박아 스킬로 만들면 같은 실수를 반복하지 않고 결과물 형식도 일관되게 유지된다 — 위 설계 원칙 1번과 2번이 여기서 나왔다.

## 스킬 구조

새 스킬은 저장소 루트 바로 밑에 폴더를 만들고 아래 형식을 따른다.

```
skill-name/
├── SKILL.md      # 필수 — 트리거 조건, 절차, 흔한 실수 (에이전트가 읽는 파일)
├── README.md     # 필수 — 사람이 읽는 설명: 뭘 하는지, 왜 이렇게 설계했는지
└── template.md   # 선택 — 출력 형식이 고정되어야 하는 경우의 스켈레톤
```

`SKILL.md`와 `README.md`의 역할을 섞지 않는다. `SKILL.md`는 에이전트가 실행 중에 참조하는 절차 문서이고, `README.md`는 저장소를 훑어보는 사람을 위한 설명이다.

폴더 루트에 `SKILL.md`가 있고 `skills/` 하위 폴더가 없으면 Claude Code가 이를 **단일 스킬 플러그인**으로 자동 인식한다. 그래서 폴더마다 `plugin.json`을 둘 필요가 없다.

폴더를 만든 뒤 루트 [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json)의 `plugins` 배열에 항목을 추가해야 설치 대상이 된다. 이 파일이 마켓플레이스의 **유일한 레지스트리**다.

```json
{
  "name": "설치 이름 (kebab-case)",
  "source": "./skill-name",
  "description": "언제 쓰는 스킬인지",
  "version": "1.0.0",
  "author": { "name": "Jeric1223" }
}
```

`name`은 폴더명과 달라도 된다 — 사용자가 `/plugin install <name>@jeric-skills`에 입력하는 이름이다.

## 기여

다른 프로젝트에서도 통할 만한 스킬이라면 PR을 환영한다. 폴더 구조, 버전 규칙, 리뷰 기준은 [CONTRIBUTING.md](./CONTRIBUTING.md)에 있다.

---

<div align="center">
<sub>Not affiliated with Anthropic.</sub>
</div>
