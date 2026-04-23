![[../../../Repository/Pasted image 20260416104822.png]]
# 요약

**이슈 계층** — Epic(큰 목표) → Story(기능 단위) → Task/Bug/하위 작업 순으로 쪼개는 거예요. 스터디로 치면 "바킹독 알고리즘 완주"가 Epic, "배열 강의 학습"이 Story, "예제 코드 Java 변환"이 Task가 되는 식이에요.

**스프린트 흐름** — Backlog(전체 할 일 목록)에서 이번 주 할 것만 골라 Sprint를 시작하면, Board에서 To Do → In Progress → Done으로 옮겨가며 진행 상황을 시각화해요.

**협업** — 친구를 팀원으로 초대하고 각 이슈에 Assignee 지정하면 돼요. Sprint 끝날 때 Review/Retrospective로 주간 회고도 할 수 있어요.

---
# 이슈 계층 구조

Jira에서 모든 할 일은 **이슈(Issue)** 라는 단위로 관리돼요. 그 이슈에 종류가 있는 거예요.

**에픽 (Epic)** — 가장 큰 단위. 완료까지 몇 주~몇 달 걸리는 큰 목표예요. 스터디로 치면 "바킹독 알고리즘 시리즈 완주" 하나가 에픽 하나예요. 에픽 안에 여러 스토리가 들어가요.

**스토리 (Story)** — 에픽을 구성하는 기능/학습 단위예요. "배열 강의 학습", "스택 강의 학습" 이런 식으로 강의 한 챕터가 스토리 하나가 되면 딱 맞아요.

**작업 (Task)** — 스토리를 실제로 수행하는 세부 행동이에요. "강의 영상 시청", "예제 코드 Java 변환", "백준 문제 3개 풀기" 이런 것들이 Task예요.

**하위 작업 (Subtask)** — Task를 더 잘게 쪼개야 할 때 써요. 사실 스터디 수준에서는 Task까지만 써도 충분해요.

**버그 (Bug)** — 개발 프로젝트용이라 스터디에선 "이해 안 된 개념 재학습" 같은 용도로 응용해서 쓸 수 있어요.

---

## 스프린트 흐름

**Backlog** — 앞으로 해야 할 모든 이슈를 쌓아두는 창고예요. 지금 당장 안 해도 되는 것들도 다 여기 넣어두면 돼요. "BFS/DFS 강의", "DP 강의" 같은 것들을 미리 다 만들어두는 거죠.

**Sprint 계획** — Backlog에서 "이번 1~2주 안에 할 것"만 골라서 Sprint에 담는 거예요. 예를 들어 이번 주 Sprint엔 "스택 강의"와 "큐 강의"만 넣는 식이에요. 욕심부려서 너무 많이 넣으면 매주 실패하게 되니까 현실적으로 잡는 게 핵심이에요.

**스크럼 보드** — Sprint에 담긴 이슈들이 To Do → In Progress → Done 세 칸으로 나뉘어 보여요. 오늘 "스택 강의 영상 봤다" 싶으면 카드를 Done으로 드래그하면 끝이에요. 한눈에 이번 주 진행 상황이 보이는 게 핵심이에요.

**Sprint Review / Retrospective** — Sprint가 끝나면(보통 1~2주) 뭘 완료했는지 확인하고, 다음 Sprint 계획해요. 친구랑 같이 "이번 주 뭐 못 했지?", "다음 주엔 뭐 하자" 하는 짧은 회고 자리예요.

---

## 실제로 어떻게 쓰냐면

```
에픽:   바킹독 알고리즘 완주
  ├─ 스토리: 스택 강의 학습
  │     ├─ 작업: 강의 영상 시청 (Assignee: 나)
  │     ├─ 작업: Java 코드 변환 (Assignee: 나)
  │     └─ 작업: 관련 문제 풀기 (Assignee: 친구)
  └─ 스토리: 큐 강의 학습
        ├─ 작업: 강의 영상 시청
        └─ 작업: 관련 문제 풀기
```

---

좋은 질문이에요! Jira 기능 전체를 스터디 관점에서 정리해드릴게요.각 항목 클릭하면 바로 더 자세히 물어볼 수 있어요.

---


![[../../../Repository/Use Jira.png]]
스터디 관점에서 특히 모를 만한 것들 짚어드리면:

**당장 쓸만한 것** — 스토리 포인트를 각 하위 작업에 붙이면 Sprint 끝날 때 Velocity Chart가 자동으로 생겨요. "이번 주 12포인트 했네, 다음 주도 12포인트 목표로 잡자" 이런 식으로 페이스 관리가 돼요.

**가장 강력한 팁** — GitHub 레포에서 커밋 메시지를 `PLM-4: 스택 Java 변환 완료` 형식으로 쓰면 Jira 이슈에 커밋이 자동 연결돼요. 나중에 이슈 보면 "이 작업할 때 어떤 코드 짰지?"가 바로 추적되고, 스터디 포트폴리오로도 활용할 수 있어요.

**자동화 꿀팁** — "하위 작업 전부 Done → 스토리 자동 Done" 규칙 하나만 만들어도 수동 클릭이 확 
줄어요. Project Settings → Automation에서 5분이면 만들 수 있어요.

---
## MCP 작동 방식 먼저

```
Claude → MCP 서버(Atlassian) → Jira REST API → Jira DB
```

Claude가 직접 Jira API를 호출하는 게 아니라, MCP 서버에 도구 실행을 요청하면 MCP 서버가 Jira API를 대신 호출하는 구조예요.

---

## 이번에 사용한 도구: `createJiraIssue`

에픽 4개 모두 같은 도구를 썼고, 파라미터만 달랐어요. PLM-9 "Java 기초 완성" 생성을 예시로 보면 이렇게 요청했어요.

```json
{
  "cloudId": "a1fab88b-e172-46bc-a010-110f98f5f35e",
  "projectKey": "PLM",
  "issueTypeName": "에픽",
  "summary": "Java 기초 완성",
  "description": "자바 언어 핵심 개념 학습 — JVM, 컬렉션 프레임워크, OOP, 예외처리 등",
  "contentFormat": "markdown",
  "additional_fields": {
    "duedate": "2026-06-16",
    "priority": { "name": "High" }
  }
}
```

각 파라미터가 뭘 의미하는지 풀면:

|파라미터|값|역할|
|---|---|---|
|`cloudId`|`a1fab88b-...`|어느 Jira 사이트인지 식별. 이전에 `getAccessibleAtlassianResources`로 조회한 값|
|`projectKey`|`PLM`|어느 프로젝트에 만들지. "Project Like Me"의 약자|
|`issueTypeName`|`에픽`|이슈 종류. 스토리/작업/버그/하위작업 중 선택|
|`summary`|`Java 기초 완성`|이슈 제목|
|`description`|`...`|이슈 본문 설명|
|`contentFormat`|`markdown`|description을 마크다운으로 파싱하도록 지정|
|`additional_fields.duedate`|`2026-06-16`|마감일. ISO 8601 날짜 형식|
|`additional_fields.priority`|`{ "name": "High" }`|우선순위. Jira가 객체 형태로 받음|

---

## `additional_fields`가 따로 있는 이유

Jira REST API는 필드가 수백 개라서 MCP 도구가 자주 쓰는 것만 전용 파라미터로 뽑아두고, 나머지는 `additional_fields`라는 객체 하나로 뭉쳐서 받아요. 그래서 `duedate`나 `priority`처럼 "있으면 좋지만 필수는 아닌" 필드들이 거기 들어간 거예요.

---

## Jira가 돌려준 응답

요청이 성공하면 Jira가 생성된 이슈 정보를 JSON으로 반환해요.

```json
{
  "id": "10016",
  "key": "PLM-9",
  "webUrl": "https://wddy2001.atlassian.net/browse/PLM-9",
  "fields": {
    "summary": "Java 기초 완성",
    "status": { "name": "해야 할 일" },
    ...
  }
}
```

여기서 `key: PLM-9`가 이슈 번호가 되는 거예요. Jira가 자동으로 채번한 값이라 Claude가 미리 알 수는 없고, 응답 받은 후에야 알 수 있어요.

---

한 줄로 요약하면 — Claude가 `createJiraIssue` 도구에 JSON 파라미터를 담아 MCP 서버에 보내면, MCP 서버가 Jira REST API의 `POST /rest/api/3/issue` 엔드포인트를 대신 호출하는 구조예요.

# 토큰 사용량
공격적으로 잡아서 일간 사용량 3퍼센트 내외

