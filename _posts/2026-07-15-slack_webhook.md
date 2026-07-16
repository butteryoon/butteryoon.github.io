---
layout: post
comments: true
title: "Slack 메시지 curl로 보내기"
description: "Slack Incoming Webhook을 만들고 curl 한 줄로 채널에 메시지를 보내는 방법 정리"
img: messaging_title.jpg
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-16 02:40:00 +0900
tags: [slack, webhook, curl]
related: slack
categories: tools
---

2020년에 gist만 올려두고 묵혀둔 초안을 보완해서 발행한다. 스크립트나 배치 작업이 끝났을 때 Slack 채널로 알림을 받고 싶다면 Incoming Webhook + curl 조합이 가장 간단하다.
<!--more-->

## Incoming Webhook 만들기

1. [https://api.slack.com/apps](https://api.slack.com/apps){:target="_blank"}에서 **Create New App → From scratch**로 앱을 만든다. (앱 이름, 워크스페이스 선택)
2. 앱 설정 화면에서 **Incoming Webhooks** 메뉴로 들어가 **Activate Incoming Webhooks**를 On으로 켠다.
3. 하단의 **Add New Webhook to Workspace**를 누르고 메시지를 보낼 채널을 선택한 뒤 **Allow(허용)** 한다.
4. `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXX` 형태의 Webhook URL이 생성된다.

이 URL은 사실상 비밀 토큰이다. URL만 알면 누구나 채널에 글을 쓸 수 있으니 공개 저장소에 커밋하지 말고 환경변수 등으로 관리한다. 유출됐다면 앱 대시보드에서 revoke 하고 새로 발급받으면 된다.

## curl로 메시지 보내기

```bash
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"빌드 완료! :tada:"}' \
  https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXX
```

성공하면 응답으로 `ok`가 온다. 줄바꿈은 `\n`, 이모지는 `:tada:` 같은 코드로 넣을 수 있고, 더 예쁘게 꾸미고 싶으면 `blocks` 페이로드(Block Kit)를 쓰면 된다.

쉘 스크립트에서 쓸 때는 이렇게 함수로 만들어두면 편하다.

```bash
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
notify() {
  curl -s -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"$1\"}" "$SLACK_WEBHOOK_URL"
}
notify "백업 작업 완료: $(date '+%F %T')"
```

## gist 코드

예전에 만들어둔 PowerShell/curl 예제 gist.

{% gist butteryoon/fd438f4d0d316cd20b8f13c14bb69419 %}

## 참고

- [Sending messages using incoming webhooks - Slack Developer Docs](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/){:target="_blank"}
- [Slack API: Your Apps](https://api.slack.com/apps){:target="_blank"}
- [Block Kit Builder](https://app.slack.com/block-kit-builder){:target="_blank"}
