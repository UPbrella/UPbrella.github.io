---
title: AWS - 슬랙 배포 알림 파이프라인 구축하기
layout: post
author: 남권우
subtitle: '슬랙으로 배포 완료 알림 보내기'
date: 2021-08-30 00:00:00 +0900
categories:
- Infrastructure
tags:
- Infrastructure
- AWS Cloud
- Slack
published: true
---

안녕하세요,
업브렐라 팀의 백엔드 남권우입니다.

이번 포스팅에서는 [컨테이너 관리를 더 쉽게, AWS ECR, ECS로 서비스 구축하기](https://upbrella.github.io/infrastructure/2023/08/19/ECR-ECS-Infra.html)에서 구축한 인프라에 슬랙 봇을 도입하여 알림 파이프라인을 구축한 이야기입니다.

## 1. 문제 정의

인프라와 CI/CD 파이프라인을 구축하였지만 여전히 불편한 점이 있었습니다.
- 배포가 완료되어도 AWS에 접속하여 직접 완료 여부를 점검해야 했습니다.
- 배포가 완료되었음을 알리는 메시지를 팀원들에게 직접 공유해야 했습니다.
- ECS가 갑작스럽게 종료되는 Event가 생겼을 때, 빠르게 팀원들에게 공유되는 알림이 필요했습니다.

따라서 저희는 AWS 슬랙 배포 알림 시스템을 구축하기로 합니다.

업브렐라 개발 CI/CD는 AWS CodeDeploy로 배포되며, 프로덕션의 경우 AWS ECS로 배포됩니다.
따라서, 배포 서비스가 다르기 때문에 각각 알림 파이프라인을 다르게 구축해야했습니다.
먼저, CodeDeploy의 경우부터 살펴보겠습니다.

## 2. CodeDeploy 알림

CodeDeploy는 알림 규칙을 생성하여 쉽게 Slack으로 알림을 보낼 수 있습니다.

알림을 메시지로 보내기 위해서는 AWS Chatbot과 Slack을 연동해야합니다.
Slack 클라이언트를 생성해주고, 채널을 생성합니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-02.png" | relative_url }}' alt='absolute'>

그리고 적절한 권한을 가진 역할을 부여하면 됩니다. 알림 권한을 추가하면, Chatbot이 메시지를 전송할 수 있습니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-03.png" | relative_url }}' alt='absolute'>

챗봇을 생성했으면 CodeDeploy에서 알림 규칙을 생성하여, 챗봇을 통해 알림을 받을 수 있습니다.
CodeDeploy > 애플리케이션 > 알림 규칙 > 알림 규칙 생성을 통해 알림 규칙을 생성할 수 있습니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-01.png" | relative_url }}' alt='absolute'>

알림 트리거 이벤트를 목적에 맞게 선택하고, 아까 생성한 Chatbot을 선택합니다.

마지막으로, Slack에서 선택한 채널에서 `/invite @AWS`로 AWS를 초대하면 알림을 받을 수 있습니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-04.png" | relative_url }}' alt='absolute'>

이렇게 간단하게 CodeDeploy 배포 완료 알림을 구축할 수 있습니다.

## 3. ECS 알림

다음으로, ECS 알림 시스템을 구축해보겠습니다.
ECS는 조금 더 복잡한데, 크게 아래의 흐름으로 동작합니다.

1. **AWS EventBridge**는 AWS 리소스에서 발생하는 이벤트를 모니터링하고, 이벤트를 수신하는 대상을 정의할 수 있는 서비스입니다.
2. ECS를 모니터링하는 EventBridge의 **규칙을 생성**하고, ECS에서 배포의 상태가 변하는 이벤트가 발생한다면 AWS의 **SNS 주제를 호출**합니다.
3. SNS 주제는 AWS의 메시지 큐라고 볼 수 있습니다. **SNS 주제를 구독하는 대상에게 메시지를 전송**합니다.
4. AWS Chatbot이 SNS 주제를 구독하면, **SNS 주제에 메시지가 전송되면 Chatbot이 Slack으로 메시지를 전송**합니다. 

제일 먼저 SNS 주제를 만들어야합니다.

AWS SNS > 주제 > 주제 생성을 통해 주제를 생성합니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-05.png" | relative_url }}' alt='absolute'>

그리고, 주제를 구독할 대상을 추가합니다. 아까 생성했던 Chatbot에 SNS 주제를 구독시킵니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-06.png" | relative_url }}' alt='absolute'>


이제, EventBridge 규칙을 생성합니다.

AWS EventBridge > 규칙 > 규칙 생성을 통해 규칙을 생성합니다.

기본 이벤트 버스로 이벤트 패턴이 있는 규칙을 생성합니다.

이벤트 패턴으로는 ECS의 배포 상태가 변하는 이벤트를 선택합니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-07.png" | relative_url }}' alt='absolute'>

마지막으로, SNS 주제를 호출하도록 설정합니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-08.png" | relative_url }}' alt='absolute'>

이렇게 설정하면 ECS도 배포 알림이 전송됩니다.

<img data-action="zoom" src='{{ "/resources/2023-08-30-09.png" | relative_url }}' alt='absolute'>

## 4. 마무리

이렇게 간단하게 AWS의 배포 알림 시스템을 구축할 수 있습니다.봇
다음 포스팅에서는 서비스와 연동하여 주요 기능 알람을 Slack으로 보낼 수 있도록 슬랙 봇을 추가해보겠습니다.
