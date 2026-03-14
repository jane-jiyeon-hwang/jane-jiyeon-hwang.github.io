---
layout: post
title: "Investigating a Drop in User Engagement"
date: 2024-12-20T16:49:03Z
authors: ["Jiyeon Hwang"]
categories: ["SQL", "Cohort Analysis", "User Engagement", "Behavior Analysis"]
description:
thumbnail: "/assets/images/gen/blog/Daily Signups.png"
image: "/assets/images/gen/blog/WAU.png"
external_url: "https://jane-jiyeon-hwang.github.io/blog/2024-12-20-user-engagement/"
---
- Yammer의 주간 활성 사용자(WAU)감소 원인을 파악하기 위해 유저, 이벤트, 이메일 로그 데이터를 SQL로 분석
- 코호트 분석, 디바이스별 인게이지먼트 분석, 이메일 클릭률(CTR) 분석을 통해 모바일 환경에서의 인게이지먼트 감소와 재참여 클릭률 저하가 핵심 원인임을 도출


## Situation

Yammer의 user engagement 대시보드에서 8월 한 달간 WAU가 지속 하락하는 문제가 발생했고, Product Head가 원인 진단과 대응 방향을 요청함.
{% include framework/shortcodes/figure.html src="/assets/images/gen/blog/WAU.png" alt="WAU" %}

## Task
WAU 하락이 신규 유입 감소, 기존 유저 리텐션 저하, 디바이스 문제, 로그인 퍼널 마찰, 이메일 리인게이지먼트 문제 등 어떤 요인에서 비롯됐는지 SQL 기반으로 규명하고, 실행 가능한 다음 액션을 제안해야 했음.

## Action

- engagement login 기준 WAU 추이 재집계
- 신규 가입 및 활성화 유저 추이 분석으로 growth 문제 여부 검증
- 활성화 시점 기준 코호트 분석으로 기존/신규 유저 retention 변화 확인
- 디바이스별 WAU 분석을 통해 모바일 환경 이상 징후가 있는지 확인
- weekly digest / re-engagement email 발송, 오픈, clickthrough 퍼널 분석
- open rate 대비 clickthrough rate 저하를 확인해 이메일 리인게이지먼트 경로 이상 가능성 도출

## Result

- 신규 가입자 감소가 주원인은 아님을 확인
- 기존 유저군, 특히 모바일 환경에서 참여 하락이 두드러짐을 확인
- 이메일 오픈은 유지/증가했지만 clickthrough 감소를 확인해 리인게이지먼트 경로의 friction 가능성 제시
- 분석 결과를 바탕으로 모바일 앱 점검 및 이메일 링크/랜딩 경험 검증 우선순위를 제안

## 분석 과정 및 결과


### 1. 주간 활성화 유저(WAU) 확인
```sql
SELECT DATE_TRUNC('week', e.occurred_at), #DATE_TRUNC : 원하는 시간 간격에 맞춰 시간을 반올림해주는 함수, 칼럼 내 데이터를 week까지만 잘라줌
       COUNT(DISTINCT e.user_id) AS weekly_active_users
  FROM tutorial.yammer_events e
 WHERE e.event_type = 'engagement'
   AND e.event_name = 'login'
 GROUP BY 1
 ORDER BY 1;
```
{% include framework/shortcodes/figure.html src="/assets/images/gen/blog/WAU.png" alt="WAU" %}

차트를 보면, 7월 마지막주 이후부터 8월 한달간 주간 활성화 유저가 매주 지속적으로 하락 추이를 띄는 것을 확인할 수 있다.
yammer는 engagement를 프로덕트와 상호작용함으로써 서버가 응답하는 경우로 정의했으며, 차트에 나타나는 모든 포인트들은 해당 주 동안 최소한 한번의 engagement event가 있었던 유저들의 숫자를 의미한다.

차트의 끝 부분에 나타난 하락의 원인을 찾고, 가능하다면 문제에 대한 해결방안을 제시하는 것까지 하는 것을 프로젝트의 목표로 설정하였다.

### 2. 가설 설정

- 위 쿼리에서 주간 활성화 유저를 event type이 engagement이면서 event name이 login인 것으로 집계하였음
- event type = engagement는 사용자가 로그인 한 이후로 프로덕트 내에서 하는 일반적인 이벤트를 뜻함
- event name = login은 yammer에 로그인 한 이벤트를 뜻함

따라서 차트를 바탕으로 파악한 **원인 가설은 크게 4가지로 볼 수 있을 것 같다.**

가설 1 : 앱이나 웹 자체의 발견하지 못한 오류로 접속에 문제가 있어서 로그인이 줄어들었을 것이다.

가설 2 : 서비스 이탈자가 있어서 로그인이 줄어들었을 것이다.

가설 3 : 주간 서비스 가입자 추이가 감소하면서 이에 따라 주간 활성화 유저도 떨어졌을 것이다.

가설 4 : 로그인 과정에서 이용자의 pain point가 발생해서(속도나 인증 등) 접속 과정에서 이탈자가 발생했을 것이다.

가설 5 : 여름 휴가를 많이 가는 시즌이어서?

**가설 검증 방법 및 순서, 순서의 기준**

가설 1 : 개발 팀에 오류 히스토리 확인해보기 → 세번째로 확인

가설 2 : 서비스 이탈자가 있었는지 확인할 수 있는 테이블과 컬럼이 있다면 그걸로 확인 → 첫번째로 확인 (제일 확인하기 쉬울 것 같아서)

가설 3 : event_name에 있는 sign up 관련 컬럼을 확인해서 서비스 가입자 추이와 주간 활성화 유저 추이가 나타내는 그래프가 비슷한지 확인해보기 → 두번째로 확인(가장 유력한 가설 후보일 것 같아서)

가설 4 : 로그인 과정에 해당하는 이벤트들을 유저별로 발생 횟수를 확인해본다. 그래서 로그인 과정 끝까지 가지 않고 중간에 이탈한 유저들이 있는지 파악해보고, 만약 로그인 과정 중 이탈한 유저가 있다면 로그인 과정들의 각 타임스탬프의 간격을 확인해서 로그인 과정에서 속도가 늘어났거나 어떤 인증 과정이 추가되어 pain point가 있었는지 확인해본다. → 네번째로 확인(서비스 이탈자도 없었고, 가입 유저 추이도 변화가 없고, 서버에 오류도 없었다면 로그인 과정 자체에 어떤 문제가 있었는지 마지막으로 확인해보기)

가설 5 : 여름 휴가를 많이 가는 시즌이 언제인지 산업 특성을 고려해서 산업별로 조사해본다
