---
layout: post
title: "Investigating a Drop in User Engagement"
date: 2024-12-20T16:49:03Z
authors: ["Jiyeon Hwang"]
categories: ["SQL", "Cohort Analysis", "User Engagement", "CTR", "Weekly Active Users", "Kyung Hee University Business Analytics & Consulting Society"]
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

## 상세 분석 과정 노트 및 결과


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

### 3. 데이터 테이블
### Table1 :  tutorial.yammer_users

유저 당 고유한 한 행을 가지는 테이블. 유저 계정의 정보가 담겨있음

| Column | Description |
|------|-------------|
| user_id | 유저의 고유 ID |
| created_at | 처음 회원가입한 시점 |
| state | 유저의 상태 (active or pending) |
| activated_at | 활성화 된 계정이라면 활성화 된 시점 |
| company_id | 유저의 회사 ID |
| language | 유저가 선택한 언어 |

### Table2 : tutorial.yammer_events

이벤트 당 한개의 행을 가지고 있음. (이벤트 = 유저가 yammer에서 하는 활동)

로그인, 메세지, 검색, 로그인 과정, 회원가입 과정 등

| Column | Description |
|---|---|
| user_id | 유저의 고유 ID |
| occurred_at | 이벤트 발생 시간 |
| event_type | 이벤트 타입. signup_flow, engagement 두 가지 타입 |
| event_name | create_user: 회원가입 과정 동안 유저가 Yammer 데이터베이스에 추가되는 이벤트|
|  | enter_email: 회원가입 과정에서 이메일 입력|
|  | enter_info: 회원가입 과정에서 개인정보 입력|
|  | complete_signup: 회원가입 및 인증 과정을 모두 완료한 이벤트|
|  | home_page: 유저가 홈페이지를 로드한 이벤트|
|  | like_message: 다른 유저 메시지에 좋아요를 누른 이벤트|
|  | login: 로그인|
|  | search_autocomplete: 검색 자동완성 기능에서 항목을 선택한 이벤트|
|  | search_run: 유저가 검색 쿼리를 실행하고 결과를 가져오는 이벤트|
|  | search_click_result_X: 검색 결과 페이지에서 n번째 결과를 클릭한 이벤트|
|  | send_message: 메시지 전송|
|  | view_inbox: 메시지 인박스 조회|
| location | IP 주소를 바탕으로 추정한 이벤트 발생 국가 |
| device | 각 이벤트 로그에 사용된 디바이스 타입 |


### Table3 : tutorial.yammer_emails

이메일을 보내는 특정 이벤트 정보를 포함

| Column | Description |
|------|-------------|
| user_id | 유저 고유 ID |
| occurred_at | 이벤트가 일어난 시각 |
| action | sent_weekly_digest : 기존의 관련 대화를 보여주는 요약 이메일을 받음|
|  | email_open : 유저가 이메일 조회 |
|  | email_clickthrough : 이메일에 있는 링크 클릭 |


### Table4 : benn.dimension_rollup_periods

특정 시간 단위로 집계할 때 활용하는 보조 테이블

최근 7일 간의 데이터 같은 롤링 기간을 정의할 때 유용

SQL에서 INTERVAL()을 사용할 수도 있지만, 미리 정의된 테이블을 사용하면

복잡한 쿼리를 단순화 할 수 있음. → 데이터 분석을 할 때 특정한 시간 단위별(예: 7일 단위, 30일 단위)로 데이터를 쉽게 조회 할 수 있음.

| Column | Description |
|------|-------------|
| period_id | 기간 집계의 타입. period 1007은 7일을 단위로 집계 |
| time_id | 특정한 데이터 포인트를 나타내는 ID. 차트의 축에 들어감.|
| | time_id가 2014-08-01이고 7일 간격으로 집계했을 경우에 8월 1일을 끝으로 하는 7일치 데이터를 나타냄|
| pst_start | 이 롤업 기간이 시작되는 시간(태평양 표준시) |
| pst_end | 이 롤업 시간이 끝나는 시간 |
| utc_start | pst_start의 UTC 버전 |
| utc_end | pst_end의 UTC 버전 |


### 4. Growth 확인

신규 가입자가 늘고 있는지를 보자는 것.
측정하기도 쉽고 보통의 회사들에서 신규가입자 대시보드를 이미 가지고 있음.

```sql
SELECT DATE_TRUNC('day',created_at) AS day,  #DATE_TRUNC : 원하는 시간 간격에 맞춰 시간을 반올림해주는 함수, 칼럼 내 데이터를 day까지만 잘라줌
       COUNT(*) AS all_users, #모든 유저 카운트
       COUNT(CASE WHEN activated_at IS NOT NULL THEN u.user_id ELSE NULL END) AS activated_users #user 테이블에서 활성화 컬럼이 null이 아니면 user_id 컬럼을 카운트하고 그게 아니면 null을 카운트(0)
  FROM tutorial.yammer_users u
 WHERE created_at >= '2014-06-01'
   AND created_at < '2014-09-01' #2014년 6월 1일 이후부터 2014년 9월 1일 이전까지 조건
 GROUP BY 1 #첫번째 컬럼 day로 그룹바이
 ORDER BY 1 #첫번째 컬럼 day로 오름차순
```

{% include framework/shortcodes/figure.html src="assets/images/gen/blog/Daily Signups.png" alt="DailySignups" %}

주말에는 떨어졌다가 주중에는 올라가는 패턴이 계속 유지되고 있으며, 안정적임을 확인하였다.

### 5. Cohort Analysis 코호트 분석 

위 Growth 분석 결과, 신규 가입자 패턴에는 이상이 없음을 확인하였고, 따라서 인게이지먼트 하락의 원인이 신규 유저 감소가 아닌 기존 유저의 참여율 저하나 이탈일 가능성이 있다.

이를 확인하기 위해 유저가 가입한 시점을 1주일 단위로 나누어 코호트 분석을 진행하였다.


```sql
SELECT DATE_TRUNC('week',z.occurred_at) AS "week",
       AVG(z.age_at_event) AS "Average age during week",
       COUNT(DISTINCT CASE WHEN z.user_age > 70 THEN z.user_id ELSE NULL END) AS "10+ weeks", #user_age가 70초과면 user_id 카운트 하고 아니면 null처리(0)
       COUNT(DISTINCT CASE WHEN z.user_age < 70 AND z.user_age >= 63 THEN z.user_id ELSE NULL END) AS "9 weeks", 
       COUNT(DISTINCT CASE WHEN z.user_age < 63 AND z.user_age >= 56 THEN z.user_id ELSE NULL END) AS "8 weeks",
       COUNT(DISTINCT CASE WHEN z.user_age < 56 AND z.user_age >= 49 THEN z.user_id ELSE NULL END) AS "7 weeks",
       COUNT(DISTINCT CASE WHEN z.user_age < 49 AND z.user_age >= 42 THEN z.user_id ELSE NULL END) AS "6 weeks",
       COUNT(DISTINCT CASE WHEN z.user_age < 42 AND z.user_age >= 35 THEN z.user_id ELSE NULL END) AS "5 weeks",
       COUNT(DISTINCT CASE WHEN z.user_age < 35 AND z.user_age >= 28 THEN z.user_id ELSE NULL END) AS "4 weeks",
       COUNT(DISTINCT CASE WHEN z.user_age < 28 AND z.user_age >= 21 THEN z.user_id ELSE NULL END) AS "3 weeks",
       COUNT(DISTINCT CASE WHEN z.user_age < 21 AND z.user_age >= 14 THEN z.user_id ELSE NULL END) AS "2 weeks",
       COUNT(DISTINCT CASE WHEN z.user_age < 14 AND z.user_age >= 7 THEN z.user_id ELSE NULL END) AS "1 week",
       COUNT(DISTINCT CASE WHEN z.user_age < 7 THEN z.user_id ELSE NULL END) AS "Less than a week"
  FROM (
        SELECT e.occurred_at,
               u.user_id,
               DATE_TRUNC('week',u.activated_at) AS activation_week, #활성화 된 시점에서 주만 추출
               EXTRACT('day' FROM e.occurred_at - u.activated_at) AS age_at_event, #이벤트 발생 시점에서 활성화 된 시점 빼서 날짜만 추출
               EXTRACT('day' FROM '2014-09-01'::TIMESTAMP - u.activated_at) AS user_age #2014년 9월 1일 기준으로 활성화된 시점 빼서 날짜만 추출
          FROM tutorial.yammer_users u
          JOIN tutorial.yammer_events e #이벤트 테이블이랑 유저 테이블을 user_id를 키로 조인
            ON e.user_id = u.user_id
           AND e.event_type = 'engagement' #join절에 조건 먼저 넣어주기 (where절에 넣는거랑 큰 차이 없음. 아래 링크 참고)
           AND e.event_name = 'login'
           AND e.occurred_at >= '2014-05-01'
           AND e.occurred_at < '2014-09-01'
         WHERE u.activated_at IS NOT NULL
       ) z

 GROUP BY 1
 ORDER BY 1
LIMIT 100
```
{% include framework/shortcodes/figure.html src="/assets/images/gen/blog/Cohort.png" alt="cohort" %}

10주 이전에 가입한 사람의 리텐션 차트를 보면 완만하게 감소하다가 갑자기 감소하는 지점이 존재하는 것을 확인할 수 있다. 따라서 신규 유저들의 인게이지먼트에 영향을 미치는 마케팅이나 검색엔진의 문제가 아니라 올드 유저들에게 인게이지먼트 하락이 나타나고 있다고 추론이 가능하다.

### 6. 디바이스별 인게이지먼트 확인
문제가 마케팅 트래픽이나 새로운 트래픽 또는 검색 엔진 때문은 아니라는걸 코호트 분석을 통해 알아냈다.
따라서 이제 여러 디바이스에서 문제가 서로 다르게 나타나는지 조금 더 국지적으로 확인해보겠다.

```sql
SELECT DATE_TRUNC('week', occurred_at) AS week, #이벤트 발생 시점에서 주만 뽑기
       COUNT(DISTINCT e.user_id) AS weekly_active_users, #이벤트 컬럼에서 유저 아이디 고유값만 카운트
       COUNT(DISTINCT CASE WHEN e.device IN ('macbook pro','lenovo thinkpad','macbook air','dell inspiron notebook',
          'asus chromebook','dell inspiron desktop','acer aspire notebook','hp pavilion desktop','acer aspire desktop','mac mini')
          THEN e.user_id ELSE NULL END) AS computer, #디바이스 컬럼에서 컴퓨터일 경우에 user_id카운트 하고 아니면 null 카운트(0)
       COUNT(DISTINCT CASE WHEN e.device IN ('iphone 5','samsung galaxy s4','nexus 5','iphone 5s','iphone 4s','nokia lumia 635',
       'htc one','samsung galaxy note','amazon fire phone') THEN e.user_id ELSE NULL END) AS phone, #디바이스 컬럼에서 휴대폰일 경우에 user_id카운트 하고 아니면 null 카운트(0)
        COUNT(DISTINCT CASE WHEN e.device IN ('ipad air','nexus 7','ipad mini','nexus 10','kindle fire','windows surface',
        'samsumg galaxy tablet') THEN e.user_id ELSE NULL END) AS tablet #디바이스 컬럼에서 태블릿인 경우에 user_id카운트 하고 아니면 null 카운트(0)
  FROM tutorial.yammer_events e
 WHERE e.event_type = 'engagement' #이벤트 타입이 인게이지먼트이면서 이벤트 네임이 로그인인것만 조건
   AND e.event_name = 'login'
 GROUP BY 1 #주 컬럼을 기준으로 그룹바이
 ORDER BY 1 #주 컬럼을 오름차순으로 정렬
LIMIT 100
```

{% include framework/shortcodes/figure.html src="/assets/images/gen/blog/device.png" alt="device" %}

분석 결과, 다른 디바이스보다 휴대폰에서 인게이지먼트 하락의 폭이 큰 것을 확인할 수 있다.

### 7. Email Actions 분석
5, 6번 분석을 바탕으로 모바일 앱에서 장기간 유저 리텐션에 문제가 있는 것을 도출해낼 수 있다.

따라서 첫번째로 시도해 볼 수 있는 것은:
a. 모바일 앱에서 문제가 있었는지 확인

두번쨰로 시도해 볼 수 있는 것은:
b. 장기간 유저의 리텐션을 확인하기 위해서 주간 요약 리포트 이메일과 관련된 데이터를 확인해보는 것. 왜냐하면 주간 요약 리포트 이메일을 보내는 이유는 프로덕트에 다시 돌아오게 하기 위함이기 때문이다.

```sql
SELECT DATE_TRUNC('week', occurred_at) AS week, #이메일 테이블에서 이벤트 일어난 시점 중 주만 뽑기
       COUNT(CASE WHEN e.action = 'sent_weekly_digest' THEN e.user_id ELSE NULL END) AS weekly_emails, #이메일 테이블에서 액션이 'sent_weekly_digest'이면 user_id 카운트 하고 아니면 null 카운트(0)
       COUNT(CASE WHEN e.action = 'sent_reengagement_email' THEN e.user_id ELSE NULL END) AS reengagement_emails, #이메일 테이블에서 액션이 'sent_reengagement_email'이면 user_id 카운트 하고 아니면 null 카운트(0)
       COUNT(CASE WHEN e.action = 'email_open' THEN e.user_id ELSE NULL END) AS email_opens, #이메일 테이블에서 액션이 'email_open'이면 user_id 카운트 하고 아니면 null 카운트(0)
       COUNT(CASE WHEN e.action = 'email_clickthrough' THEN e.user_id ELSE NULL END) AS email_clickthroughs #이메일 테이블에서 액션이 'email_clickthrough'이면 user_id 카운트 하고 아니면 null 카운트(0)
  FROM tutorial.yammer_emails e
 GROUP BY 1
 ORDER BY 1
```

{% include framework/shortcodes/figure.html src="/assets/images/gen/blog/email.png" alt="email" %}

차트를 보면 주간 요약 리포트 이메일 전송은 계속 증가하고 있고 이메일을 열람 또한 계속 증가하고 있다. 반면, 이메일에 있는 링크를 클릭하는 이벤트는 감소하는 것을 확인할 수 있다.

### 8. Email open, CTR 분석

위 분석에서 이메일 내부 링크 클릭이 감소하는 것을 확인할 수 있었기 때문에, 더 상세하게 이메일에서의 클릭 그리고 이메일 열어본 비율을 차트를 통해 확인해보고자 한다.

```sql
SELECT week,
       weekly_opens/CASE WHEN weekly_emails = 0 THEN 1 ELSE weekly_emails END::FLOAT AS weekly_open_rate,
       weekly_ctr/CASE WHEN weekly_opens = 0 THEN 1 ELSE weekly_opens END::FLOAT AS weekly_ctr,
       retain_opens/CASE WHEN retain_emails = 0 THEN 1 ELSE retain_emails END::FLOAT AS retain_open_rate,
       retain_ctr/CASE WHEN retain_opens = 0 THEN 1 ELSE retain_opens END::FLOAT AS retain_ctr
  FROM (
SELECT DATE_TRUNC('week',e1.occurred_at) AS week,
       COUNT(CASE WHEN e1.action = 'sent_weekly_digest' THEN e1.user_id ELSE NULL END) AS weekly_emails,
       COUNT(CASE WHEN e1.action = 'sent_weekly_digest' THEN e2.user_id ELSE NULL END) AS weekly_opens,
       COUNT(CASE WHEN e1.action = 'sent_weekly_digest' THEN e3.user_id ELSE NULL END) AS weekly_ctr,
       COUNT(CASE WHEN e1.action = 'sent_reengagement_email' THEN e1.user_id ELSE NULL END) AS retain_emails,
       COUNT(CASE WHEN e1.action = 'sent_reengagement_email' THEN e2.user_id ELSE NULL END) AS retain_opens,
       COUNT(CASE WHEN e1.action = 'sent_reengagement_email' THEN e3.user_id ELSE NULL END) AS retain_ctr
  FROM tutorial.yammer_emails e1
  LEFT JOIN tutorial.yammer_emails e2
    ON e2.occurred_at >= e1.occurred_at
   AND e2.occurred_at < e1.occurred_at + INTERVAL '5 MINUTE'
   AND e2.user_id = e1.user_id
   AND e2.action = 'email_open'
  LEFT JOIN tutorial.yammer_emails e3
    ON e3.occurred_at >= e2.occurred_at
   AND e3.occurred_at < e2.occurred_at + INTERVAL '5 MINUTE'
   AND e3.user_id = e2.user_id
   AND e3.action = 'email_clickthrough'
 WHERE e1.occurred_at >= '2014-06-01'
   AND e1.occurred_at < '2014-09-01'
   AND e1.action IN ('sent_weekly_digest','sent_reengagement_email')
 GROUP BY 1
       ) a
 ORDER BY 1
```

{% include framework/shortcodes/figure.html src="/assets/images/gen/blog/CTR.png" alt="emailCTR" %}

분석 결과, 이메일 내부 링크를 클릭하는 것에 무언가 문제가 있다는 것을 파악할 수 있다.

### 9. 결론
- 문제는 모바일 앱이랑 주간 요약 리포트 이메일에서 나타고 있는 것을 확인함
- 문제를 가장 처음 제시한 프로덕트 헤드에게 문제를 국지화된 관점으로 알려야함
- 모바일 앱에서 뭔가 문제가 있는걸 확인할 필요가 있음

### 10. 분석하면서 느낀점
growth 먼저 확인 → growth에 문제 없어서 가입 시점을 기준으로 코호트 분석해보고 그룹 별로 문제가 있는지 확인
→ 코호트 분석에서 오래된 유저의 리텐션이 떨어지는걸 발견해서 마케팅이나 검색 엔진 문제는 아닌걸 확인하고 디바이스 별로 문제를 확인해보기
→ 휴대폰, 태블릿에서 유저 인게이지먼트 하락 발견 → 모바일 앱, 그리고 장기간 유저 리텐션의 문제가 있다는걸 확인
→ 그래서 모바일 앱 문제를 확인해보거나, 장기간 유저 리텐션과 관련돼서 프로덕트에 다시 돌아오게 만드는 역할을 하는 주간 요약 리포트 이메일 관련 데이터 확인해보기
→ 확인 결과 이메일에 있는 링크를 클릭하는 이벤트가 감소하고 있는 것을 확인할 수 있었음

위는 mode analytics 예시 분석의 전반적인 flow인데,

우리가 처음 발견한 문제는 7월 마지막주를 기준으로 그 이후부터 8월 한달간 주간 활성화 유저가 매주 지속적으로 하락 추이가 나타나는거였고 코호트 분석 결과를 보면 7월 마지막주를 기준으로 10주 이전에 가입한 유저 뿐만 아니라 최근 가입한 유저들도 리텐션이 떨어지고 있었음.

따라서 코호트 분석에서 장기간 유저의 리텐션에 문제가 있다고 바라보는 것의 논리가 잘 이해가 안됐음. 

물론 장기간 유저 차트를 보면 7월 마지막주 이전에는 리텐션이 증가하는 추세이나 7월 마지막주 이후부터는 가입 시점에 상관없이 모든 유저의 리텐션이 감소하고 있음. 따라서 문제가 단순히 장기간 유저 뿐만 아니라 7월 마지막주 그 시점에 모바일 앱에 문제가 있는 것에 집중해야할 필요가 있다고 느꼈음.

그래서 7월 마지막주 이전의 리텐션과 관련된 주간 리포트 요약 이메일 데이터 분석을 해서 7월 마지막주 이전과 이후에 이메일 링크 클릭에 변화가 있었는지, 그리고 코호트 별로 변화가 있었는지도 확인해본다면 장기간 유저의 리텐션이 문제인건지 아니면 그냥 그 시점에 모바일 앱의 문제나 마케팅으로 인해 유지되던 리텐션이 떨어진건지 등 좀 더 자세한 분석이 가능할 것이라고 생각함.