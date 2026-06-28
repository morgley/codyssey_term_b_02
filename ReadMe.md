# AI 뉴스 자동 수집 및 요약 워크플로우

## 1. 프로젝트 개요

본 프로젝트는 n8n을 이용하여 AI 관련 뉴스를 자동으로 수집하고, 로컬 LLM인 Ollama를 통해 한국어로 요약한 뒤, Notion 데이터베이스에 저장하는 자동화 워크플로우이다.

워크플로우는 매일 오전 09:00에 실행되며, 여러 RSS 피드에서 최신 기술 뉴스를 수집한다. 이후 AI 관련 키워드가 포함된 기사만 선별하고, 중복 저장 여부를 확인한 뒤, 새로운 기사만 요약하여 Notion에 저장한다.

------

## 2. 사용 도구

| 구분        | 도구                | 역할                                     |
| ----------- | ------------------- | ---------------------------------------- |
| 자동화 도구 | n8n                 | 전체 워크플로우 실행 및 노드 연결        |
| 뉴스 수집   | RSS Feed Read Node  | 기술 뉴스 RSS 피드 수집                  |
| 데이터 처리 | Code Node           | 키워드 필터링, 최신 기사 선택, 해시 생성 |
| AI 요약     | Ollama `qwen2.5:3b` | 뉴스 본문 한국어 요약                    |
| 저장소      | Notion Database     | 요약 결과 저장 및 중복 확인              |
| 실행 방식   | Schedule Trigger    | 매일 09:00 자동 실행                     |

------

## 3. 전체 워크플로우 구조

text

```
Daily 09:00 Trigger
        ↓
RSS Feed 3개 수집
        ↓
Merge
        ↓
AI 관련 뉴스 필터링
        ↓
최신 기사 1건 선택
        ↓
Notion 중복 확인
        ↓
중복이면 종료
중복이 아니면 Ollama 요약
        ↓
Notion 저장
```

------

## 4. 입력 데이터 수집

### 4.1 실행 주기

워크플로우는 n8n의 `Schedule Trigger`를 사용하여 매일 오전 09:00에 실행된다.

json

```
"triggerAtHour": 9
```

또한 최종 JSON 파일에서 다음 설정이 확인된다.

json

```
"active": true
```

따라서 워크플로우는 자동 실행 가능한 활성 상태이다.

------

### 4.2 RSS 수집 대상

총 3개의 RSS 피드를 사용한다.

| 번호 | RSS 이름              | URL                                                          |
| ---- | --------------------- | ------------------------------------------------------------ |
| 1    | TechCrunch AI         | `https://techcrunch.com/category/artificial-intelligence/feed/` |
| 2    | MIT Technology Review | `https://www.technologyreview.com/feed/`                     |
| 3    | GeekNews              | `https://feeds.feedburner.com/geeknews-feed`                 |

각 RSS 노드는 실패 시 재시도 정책을 적용한다.

json

```
"retryOnFail": true,
"maxTries": 2,
"waitBetweenTries": 3000
```

------

## 5. 뉴스 필터링 기준

수집된 RSS 기사 중 AI 관련 키워드가 포함된 기사만 선별한다.

### 5.1 필터링 대상 필드

다음 필드를 합쳐 검색 대상으로 사용한다.

text

```
title
content
contentSnippet
description
summary
```

즉, 기사 제목뿐 아니라 본문 요약 정보까지 포함하여 AI 관련성을 판단한다.

------

### 5.2 사용 키워드

다음 키워드 중 하나라도 포함되면 AI 관련 기사로 판단한다.

text

```
ai
artificial intelligence
generative ai
llm
large language model
chatgpt
openai
anthropic
google deepmind
machine learning
deep learning
agent
ai agent
인공지능
생성형 ai
머신러닝
딥러닝
```

대소문자 차이를 방지하기 위해 검색 대상 문장은 모두 소문자로 변환한 뒤 비교한다.

------

## 6. 기사 선택 정책

AI 관련 뉴스가 여러 개 발견될 경우, 발행일 기준으로 최신 기사 1건만 선택한다.

정렬 기준은 다음과 같다.

javascript

```
new Date(b.pubDate).getTime() - new Date(a.pubDate).getTime()
```

선택 결과는 다음 정보를 포함한다.

| 필드        | 설명                |
| ----------- | ------------------- |
| `title`     | 기사 제목           |
| `link`      | 원문 링크           |
| `pubDate`   | 발행일              |
| `content`   | 기사 내용           |
| `source`    | 출처                |
| `uniqueKey` | 중복 방지용 고유 키 |
| `status`    | 처리 상태           |

------

## 7. 중복 방지 정책

중복 저장을 막기 위해 기사 링크를 기반으로 `UniqueKey`를 생성한다.

### 7.1 UniqueKey 생성 방식

기사의 `link`가 있으면 link를 사용하고, 없으면 `guid`, `title` 순서로 대체한다.

javascript

```
const keySource = selected.link || selected.guid || selected.title;
const uniqueKey = simpleHash(keySource);
```

생성된 값은 다음과 같은 형태를 가진다.

text

```
news_123456789
```

------

### 7.2 Notion 중복 확인 방식

Notion 데이터베이스에서 `UniqueKey` 속성이 같은 페이지가 있는지 조회한다.

text

```
UniqueKey equals selected.uniqueKey
```

조회 결과가 존재하면 중복 기사로 판단하고 이후 요약 및 저장 과정을 실행하지 않는다.

text

```
중복 있음 → 종료
중복 없음 → Ollama 요약 → Notion 저장
```

------

## 8. AI 요약 정책

중복이 아닌 새 기사만 Ollama를 통해 요약한다.

### 8.1 사용 모델

text

```
qwen2.5:3b
```

### 8.2 호출 API

text

```
http://host.docker.internal:11434/api/generate
```

### 8.3 요약 프롬프트 조건

Ollama에는 다음 조건을 부여한다.

text

```
- 반드시 한국어로만 작성
- 영어와 중국어 사용 금지
- 요약 3개 작성
- 핵심 키워드 3개 작성
```

출력 형식은 다음과 같다.

text

```
요약:
1.
2.
3.

핵심 키워드:
-
-
-
```

본문은 너무 길어지는 것을 방지하기 위해 최대 3000자까지만 전달한다.

javascript

```
content.slice(0, 3000)
```

------

## 9. Notion 저장 구조

요약이 완료된 기사는 Notion 데이터베이스에 저장된다.

### 9.1 저장 대상 데이터베이스

text

```
뉴스 저장소
```

### 9.2 저장 필드

| Notion 속성 | 저장 내용        |
| ----------- | ---------------- |
| 제목        | 기사 제목        |
| 요약        | Ollama 요약 결과 |
| 원문링크    | 기사 URL         |
| 발행일시    | 기사 발행일      |
| UniqueKey   | 중복 방지용 키   |
| Source      | 뉴스 출처        |
| CreatedAt   | 저장 시각        |
| Status      | 저장 상태        |

------

## 10. 재시도 정책

외부 요청 실패에 대비하여 주요 노드에 재시도 정책을 적용하였다.

| 대상 노드        | 재시도 여부 | 최대 시도 횟수 | 대기 시간 |
| ---------------- | ----------- | -------------- | --------- |
| RSS Feed Read    | 적용        | 2회            | 3000ms    |
| Notion 중복 확인 | 적용        | 2회            | 3000ms    |
| Ollama 요약 요청 | 적용        | 2회            | 3000ms    |
| Notion 저장      | 적용        | 2회            | 3000ms    |

재시도 설정은 다음과 같다.

json

```
"retryOnFail": true,
"maxTries": 2,
"waitBetweenTries": 3000
```

------

## 11. 에러 및 예외 처리

### 11.1 AI 관련 뉴스가 없는 경우

필터링 결과 AI 관련 뉴스가 없으면 다음 상태를 생성한다.

json

```
{
  "noNews": true,
  "status": "NoNews",
  "message": "AI 관련 뉴스가 RSS에서 발견되지 않았습니다."
}
```

이 경우 Notion 저장 단계로 진행하지 않고 종료된다.

------

### 11.2 중복 뉴스인 경우

Notion에서 동일한 `UniqueKey`가 발견되면 중복 기사로 판단한다.

text

```
중복 기사 → 요약하지 않음 → 저장하지 않음 → 종료
```

이를 통해 같은 기사가 반복 저장되는 문제를 방지한다.

------

### 11.3 외부 서비스 실패

RSS, Ollama, Notion과 같은 외부 서비스 호출이 실패할 수 있으므로 각 주요 노드에 최대 2회 재시도를 적용하였다.

text

```
1차 실패 → 3초 대기 → 2차 재시도
```

2회 모두 실패하면 해당 실행은 실패로 기록되며, n8n 실행 로그에서 원인을 확인할 수 있다.

------

## 12. 워크플로우 활성화 상태

제출 JSON 기준 워크플로우는 활성화되어 있다.

json

```
"active": true
```

따라서 Schedule Trigger에 따라 매일 오전 09:00에 자동 실행된다.

------

## 13. 실행 결과 예시

정상 실행 시 Notion에는 다음과 같은 형태로 데이터가 저장된다.

text

```
제목: AI 관련 뉴스 제목
요약:
1. 주요 내용 요약
2. 기술적 의미 요약
3. 산업적 영향 요약

원문링크: https://...
발행일시: 2026-...
UniqueKey: news_123456789
Source: Tech RSS
Status: Saved
CreatedAt: 실행 시각
```

------

## 14. 프로젝트 특징

본 워크플로우의 핵심 특징은 다음과 같다.

| 특징             | 설명                       |
| ---------------- | -------------------------- |
| 자동 실행        | 매일 09:00 스케줄 실행     |
| 다중 RSS 수집    | 3개 RSS 피드에서 뉴스 수집 |
| AI 키워드 필터링 | AI 관련 기사만 선별        |
| 최신 기사 선택   | 최신 기사 1건만 요약       |
| 중복 방지        | 링크 기반 UniqueKey 사용   |
| 비용 절감        | 로컬 Ollama LLM 사용       |
| 안정성 확보      | 주요 노드 재시도 정책 적용 |
| Notion 저장      | 결과를 구조화된 DB에 저장  |

------

## 15. 결론

본 프로젝트는 n8n을 기반으로 AI 뉴스 수집, 필터링, 중복 확인, AI 요약, Notion 저장까지 자동화한 워크플로우이다.

특히 링크 기반 `UniqueKey`를 활용하여 중복 저장을 방지하고, Ollama 로컬 LLM을 사용하여 API 비용 없이 한국어 요약을 생성한다. 또한 주요 외부 요청 노드에 재시도 정책을 적용하여 안정성을 높였다.

최종 JSON 파일에서 `"active": true`가 확인되므로, 워크플로우는 자동 실행 가능한 상태이다.

------

# 검토 요약 보고

| 항목            | 검토 결과               |
| --------------- | ----------------------- |
| 워크플로우 이름 | `TermProject_B_V01`     |
| 자동 실행       | 매일 09:00              |
| 활성화 상태     | `active: true` 확인     |
| RSS 수집        | 3개 피드 사용           |
| AI 필터링       | 키워드 기반 적용        |
| 최신 기사 선택  | 최신 1건 선택           |
| 중복 방지       | Notion `UniqueKey` 조회 |
| AI 요약         | Ollama `qwen2.5:3b`     |
| 저장 위치       | Notion 뉴스 저장소      |
| 재시도 정책     | 주요 노드 최대 2회      |
