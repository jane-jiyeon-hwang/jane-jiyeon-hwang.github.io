---
layout: post
title: "Investigating a Drop in User Engagement"
date: 2024-12-20T16:49:03Z
authors: ["Jiyeon Hwang"]
categories: ["SQL", "Cohort Analysis", "User Engagement", "Behavior Analysis"]
description:
thumbnail: "/assets/images/gen/blog/Daily Signups.png"
image: "/assets/images/gen/blog/Daily Signups.png"
external_url: "https://jane-jiyeon-hwang.github.io/blog/2024-12-20-user-engagement/"
---
- Investigated the decline in weekly active users using SQL by analyzing user engagement patterns, cohort trends, device-level behavior, and email click-through rates.
- Identified key causes of user drop-off and proposed product improvement strategies.

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

At the same time, a number of ambiguities in the informal specification had attracted attention.These issues spurred the creation of tools such as Babelmark to compare the output of various implementations, and an effort by some developers of Markdown parsers for standardisation. However, Gruber has argued that complete standardization would be a mistake:

```js
$(window).scroll(function () {
  // this will work when your window scrolled.
  var scroll = $(window).scrollTop();
  if (scroll > 100) {
    $(".header").addClass("header-scrolled");
  } else {
    $(".header").removeClass("header-scrolled");
  }
});
```

Gruber avoided using curly braces in Markdown to unofficially reserve them for implementation-specific extensions. Markdown Extra adds the following features to Markdown:

- markdown markup inside HTML blocks
- elements with id/class attribute
- fenced code blocks that span multiple lines of code
- tables
- definition lists
- footnotes
- abbreviations
