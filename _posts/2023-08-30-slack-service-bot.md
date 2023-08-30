---
title: 슬랙 봇으로 서비스 알림 받기
layout: post
author: 남권우
subtitle: '슬랙 API 활용'
date: 2021-08-30 00:00:00 +0900
categories:
- Infrastructure
tags:
- Infrastructure
- Slack
published: true
---

안녕하세요,
업브렐라 팀의 백엔드 남권우입니다.

이번 포스팅에서는 [AWS 슬랙 배포 알림 파이프라인 구축하기](https://upbrella.github.io/infrastructure/2021/08/29/slack-alarm-bot.html)에서 이어지는 내용으로, 슬랙 봇을 통해 서비스 알림을 받는 방법에 대해 알아보겠습니다.

## 1. 문제 정의

슬랙 봇을 도입하게 된 이야기에 앞서, 업브렐라 서비스에 대해서 설명해보겠습니다.
업브렐라는 공유 우산 서비스로 사용자가 우산의 QR을 촬영하여 우산을 빌릴 수 있습니다.
우산을 빌릴 때, 보증금 10,000원을 입금하며 14일 이내 반납 시 보증금을 돌려받을 수 있습니다.
업브렐라는 결제 API를 사용할 수 없었기 때문에, 반납을 관리자가 수동으로 해야하는 시스템입니다.
그러나 관리자가 반납 여부를 알림 받지 못하기 때문에 어드민 페이지를 계속 트래킹해야하는 불편함이 있었습니다.
즉, 문제점은 다음과 같았습니다.

- 반납이 완료되었는지 실시간으로 알지 못합니다.
- 여러 명의 관리자에게 알림을 보내는 기능이 필요했습니다.

이러한 문제점을 해결하기 위해, 반납이 완료되면 알림을 보내주는 슬랙 봇을 도입하게 되었습니다.

## 2. 슬랙 봇 생성하기

[Slack API](https://api.slack.com)에서 우선 슬랙 App을 생성해야합니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-10.png" | relative_url }}' alt='absolute'>

App을 생성하고, App의 옵션은 원하는 옵션을 선택합니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-13.png" | relative_url }}' alt='absolute'>

간단한 자기소개도 작성해줍니다.

슬랙 봇을 생성했으니, 서비스에서 이벤트가 발생하면 슬랙 봇의 Webhook URL로 전송하고, 이를 슬랙 봇이 지정 채널에 보내주는 방식으로 알림을 보낼 수 있습니다.


<img data-action="zoom" src='{{ "/resources/2023-08-30-11.png" | relative_url }}' alt='absolute'>

기능 추가에서 Incoming Webhook을 활성화해줍니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-14.png" | relative_url }}' alt='absolute'>

Webhook을 생성하고, Add New Webhook to Workspace로 Webhook을 원하는 채널에 생성합니다.
여기서 생성된 Webhook URL로 요청을 보내면 슬랙 봇이 지정한 채널로 메시지를 전송합니다.

## 3. 서버에서 Webhook으로 알림 보내기

이제 서버에서 Webhook으로 알림을 보내는 방법에 대해 알아보겠습니다.

메시지를 생성하고, 위에서 생성한 Webhook URL으로 전송하면 됩니다. 간단한 프로토타입을 아래와 같이 구현했습니다.

```java

@Service
@RequiredArgsConstructor
public class SlackAlarmService {
    private final SlackBotConfig slackBotConfig;
    private final RestTemplate restTemplate;

    public void notifyReturn(long unrefundedCount) {

        StringBuilder sb = new StringBuilder();
        sb.append("*우산이 반납되었습니다. 보증금을 환급해주세요.*\n\n")
                .append("*환급 대기 건수* : ")
                .append(unrefundedCount);

        send(sb.toString());
    }

    private void send(String message) {

        Map<String, Object> request = new HashMap<>();
        request.put("text", message);

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(request);

        restTemplate.exchange(slackBotConfig.getWebHookUrl(), POST, entity, String.class);
    }
}
```

알림을 보내는 김에, 미환급 우산 개수까지 같이 세서 보내는 기능을 추가했습니다.

```java
    @PatchMapping
    public ResponseEntity<CustomResponse> returnUmbrellaByUser(@RequestBody ReturnUmbrellaByUserRequest returnUmbrellaByUserRequest, HttpSession httpSession) throws JsonProcessingException 
    {
        // ... 비즈니스 로직

        rentService.returnUmbrellaByUser(userToReturn, returnUmbrellaByUserRequest);
        // 미환급 우산 개수 조회
        long unrefundedRentCount = rentService.countUnrefundedRent();
        // 반납 완료 및 미환급 우산 개수를 슬랙 알림 전송
        slackAlarmService.notifyReturn(unrefundedRentCount);
        return ResponseEntity
                .ok()
                .body(new CustomResponse(
                        "success",
                        200,
                        "우산 반납 성공"
                ));
    }
```

반납을 했을 때, 아래와 같이 알림이 오는 것을 확인할 수 있었습니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-12.png" | relative_url }}' alt='absolute'>

간단한 프로토 타입이며, 앞으로 비즈니스 로직에서 운영 편의를 위한 다양한 편의 기능을 슬랙 봇을 통해 구현해볼 예정입니다.

