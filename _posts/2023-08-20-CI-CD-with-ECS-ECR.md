---
title: 롤링 업데이트, AWS ECR, ECS로 쉽게
layout: post
author: 남권우
subtitle: 'AWS ECR, ECS로 쉽게 CI/CD 구성하기'
categories:
- Infrastructure
tags:
- Infrastructure
- AWS Cloud
published: false
---

안녕하세요,
업브렐라 팀의 백엔드 남권우입니다.

이번 포스팅에서는 [컨테이너 관리를 더 쉽게, AWS ECR, ECS로 서비스 구축하기](https://upbrella.github.io/infrastructure/2023/08/19/ECR-ECS-Infra.html)에서 구축한 인프라를 바탕으로 CI/CD 환경을 구축한 이야기를 소개하고자 합니다.

## 1. 문제 정의

초기에 개발용으로 CI/CD 환경을 구축해놓았었는데요, 간단하게 정리하면 Github Actions와 AWS CodeDeploy, S3를 이용한 배포전략이었습니다.

1. Github Actions에서 빌드를 진행한다.
2. 빌드 완료된 파일을 압축하여 AWS S3에 업로드한다.
3. AWS CodeDeploy가 트리거되어 S3에 업로드 된 파일을 EC2 인스턴스에 배포한다.

위 방법은 간단하고 빠르게 구성할 수 있다는 장점이 있었지만 몇 가지 문제점이 있었습니다.

1. 배포 시간이 오래 걸린다.
2. 배포 중 서비스가 중단된다.
3. 배포 중 서비스가 중단되어도 복구가 느리다.
4. 배포 중 서비스가 중단되어도 사용자는 에러 페이지를 보지 않는다.

## 2. 배포 전략

### 2.1. Rolling Update

### 2.2. Blue/Green

### 2.3. Canary

## 3. 업브렐라의 CI/CD Flow

## 4. 사용해보니..
