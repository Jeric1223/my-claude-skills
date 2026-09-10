# 출력 골격

Postman Collection v2.1. 아래 구조를 채운다. `<...>`는 소스에서 확인한 값으로 치환하고, 확인 못 한 자리는 `※ 확인 필요`로 남긴다.

```json
{
  "info": {
    "name": "<API군 이름 (티켓번호 있으면 괄호로)>",
    "description": "<엔드포인트 한 줄 요약>\n\n■ <비슷한 기존 API>와의 차이 N가지\n1. <차이>\n2. <차이>\n\n■ 인증\n<미들웨어 본문에서 확인한 실제 방식. Bearer / signed cookie / API Key 중 무엇인지, 어떤 헤더·쿠키가 필요한지>\n\n■ <여러 요청이 공유하는 코드표>\n| 코드 | 이름 |\n|---|---|\n| <CODE> | <라벨> |\n\n■ 요청은 필터를 모두 켠 상태로 만들어 두었다. 불필요한 파라미터는 Params 탭에서 체크 해제해서 쓰면 된다. 파라미터별 설명은 Params 탭의 Description 열 참고.",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    { "key": "server", "value": "https://CHANGE_ME.example.com", "type": "string" },
    { "key": "<변하는 값>", "value": "<SET_..._HERE>", "type": "string" }
  ],
  "item": [
    {
      "name": "<기능 이름> (<METHOD> <경로>)",
      "request": {
        "method": "<METHOD>",
        "header": [
          { "key": "<인증 헤더>", "value": "<값>" }
        ],
        "url": {
          "raw": "{{server}}<경로>?<query[]와 일치하는 쿼리스트링>",
          "host": ["{{server}}"],
          "path": ["<경로>", "<세그먼트>"],
          "query": [
            {
              "key": "<파라미터>",
              "value": "<예시값>",
              "description": "[필수] <타입·범위·기본값>. <미전달 시 동작>.\n<허용값1> <라벨1> | <허용값2> <라벨2>\n※ <이 파라미터에만 걸리는 함정>"
            },
            {
              "key": "<복수 전달 파라미터>",
              "value": "<두 번째 예시값>",
              "description": "(위 <파라미터명>의 두 번째 값 — 복수 전달 예시)"
            }
          ]
        },
        "description": "<기능 한 줄>.\n\n■ 조회 제외 규칙 (파라미터 아님, 항상 적용)\n- <서버가 하드코딩으로 거르는 조건>\n\n■ <고정 동작>\n<정렬·건수 상한 등 클라이언트가 못 바꾸는 것>\n\n■ 응답: <형태>\n<필드 목록. 리포지토리의 attributes/select에서 확인한 것만>\n※ <응답에 없는데 있을 거라 오해하기 쉬운 것>\n\n■ 에러\n<코드> <조건>"
      }
    }
  ]
}
```

## 채울 때 지키는 것

- `raw`의 쿼리스트링과 `query[]` 배열의 key·순서를 일치시킨다
- 줄바꿈은 JSON 문자열 안에서 `\n`. 마크다운 테이블도 `\n`으로 이어 붙이면 렌더링된다
- 선택 파라미터도 활성 상태로 둔다. 상호 배타적인 것만 `"disabled": true` + 이유를 description에
- 실제 호스트·계정·토큰 값을 넣지 않는다. 전부 variable로
- 작성 후 `node -e "JSON.parse(...)"`로 파싱 확인
