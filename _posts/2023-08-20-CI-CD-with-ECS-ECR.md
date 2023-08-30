---
title: 롤링 업데이트, AWS ECR, ECS로 쉽게
layout: post
author: 남권우
subtitle: 'AWS ECR, ECS로 쉽게 CI/CD 구성하기'
date: 2021-08-20 00:00:00 +0900
categories:
- Infrastructure
tags:
- Infrastructure
- AWS Cloud
published: true
---

안녕하세요,
업브렐라 팀의 백엔드 남권우입니다.

이번 포스팅에서는 [컨테이너 관리를 더 쉽게, AWS ECR, ECS로 서비스 구축하기](https://upbrella.github.io/infrastructure/2023/08/18/ECR-ECS-Infra.html)에서 구축한 인프라를 바탕으로 CI/CD 환경을 구축한 이야기를 소개하고자 합니다.

## 1. 문제 정의

초기에 개발용으로 CI/CD 환경을 구축해놓았었는데요, 간단하게 정리하면 Github Actions와 AWS CodeDeploy, S3를 이용한 인플레이스 배포전략이었습니다.

1. Github Actions에서 빌드를 진행한다.
2. 빌드 완료된 파일을 압축하여 AWS S3에 업로드한다.
3. AWS CodeDeploy가 트리거되어 S3에 업로드 된 파일을 EC2 인스턴스에 배포한다. 이 때, 실행 중인 인스턴스를 종료시키고 새로운 인스턴스를 배포한다.

위 배포 전략은 간단하고 빠르게 구성할 수 있다는 장점이 있었지만 몇 가지 문제점이 있었습니다.

#### 1. 배포 중 서비스가 중단된다.
이 문제는 초기에 저희가 인플레이스 배포 전략을 선택했기 때문에 발생했습니다. Codedeploy는 다양한 배포 전략을 지원하기 때문에 ECS 배포를 사용하지 않더라도 애플리케이션이 중단되지 않고 배포할 수 있는 여러 전략이 있습니다. 

따라서 ECS, ECR 기반의 도커 이미지 배포 방식으로 전환하면서 배포 전략 또한 변경했습니다.

#### 2. 배포 파일을 도커 이미지 단위로 관리하지 못한다.
ECS, ECR 배포를 사용하면 ECR에서 제공하는 이미지 관리, 버전관리를 다양한 편의 기능을 이용할 수 있으며, 도커 컨테이너 단위로 배포하기 때문에 모든 환경에서 동일하게 실행환경을 구성할 수 있습니다.

## 2. 배포 전략

### 2.1. In-place

<img data-action="zoom" src='{{ "/resources/2023-08-20-03.png" | relative_url }}' alt='absolute'>

인플레이스 배포는 배포 그룹의 인스턴스의 대상 애플리케이션을 일시정지한 후 새로운 버전을 배포하는 방법을 의미합니다. 

### 2.2. Rolling Update

<img data-action="zoom" src='{{ "/resources/2023-08-20-02.png" | relative_url }}' alt='absolute'>

롤링 배포는 여러 개의 가동 중인 인스턴스를 갖춘 환경에서 점진적으로 서버를 업데이트하며 트래픽을 전환하는 방법입니다. 이 때, 배포 중에 가용 처리 용량이 감소한다는 단점이 있을 수 있습니다.

AWS에서는 Elastic Beanstalk에서는 롤링과 추가 배치를 이용한 롤링을 제공합니다. 추가 배치의 롤링은 새 버전의 인스턴스를 배포한 후 바로 시작하여, 모든 용량이 유지되도록 한다는 점이 다릅니다.

롤링 배포는 서버를 한번에 많이 생성하지 않고 점진적으로 전환하기 때문에 리소스를 많이 차지하지 않지만 대신 배포 시간이 오래 걸린다는 단점이 있을 수 있습니다.

### 2.3. Blue/Green

<img data-action="zoom" src='{{ "/resources/2023-08-20-04.png" | relative_url }}' alt='absolute'>

구버전의 환경과 새 버전의 환경이 동시에 존재하도록 구축한 후, 한 번에 혹은 점진적으로 전환하는 방법입니다. 구버전의 모든 인스턴스와 새 버전의 모든 인스턴스가 공존하기 때문에 롤백이 빠르다는 점이 장점이라고 할 수 있습니다. 다만 인스턴스 수가 많을 수록 두 환경을 동시에 띄우기 위한 서버 리소스가 2배로 필요합니다.

### 2.4. Canary

<img data-action="zoom" src='{{ "/resources/2023-08-20-05.png" | relative_url }}' alt='absolute'>

카나리 배포는 블루 그린과 유사하지만 한 번에 전환하는 것이 아닌 일부 트래픽을 새 환경으로 분산시켜 오류율과 성능을 비교한 후 배포하는 방법입니다.

### 2.5. 어떤 전략이 좋을까?

위 전략들과 업브렐라 서버의 서버환경을 종합적으로 고려해서 배포 전략을 선택했습니다. 업브렐라의 서버 환경은 비용 절감의 문제로 micro~small 사이즈의 인스턴스를 사용합니다. 따라서 컨테이너 수가 늘어났을 때 Blue/Green과 같은 배포 전략은 서버 자원의 측면에서 부담이 된다고 판단하여 일반적으로 사용하는 Rolling Update 전략을 사용했습니다. ECS에서 기본 배포 옵션으로 Rolling Update를 제공하기도 합니다.

## 3. 업브렐라의 CI/CD Flow

업브렐라의 CI/CD 흐름을 정리하면 아래 이미지와 같습니다.

<img data-action="zoom" src='{{ "/resources/2023-08-20-01.png" | relative_url }}' alt='absolute'>

1. Github Actions에서 AssumeRole로 AWS의 권한을 얻는다.
2. Github Actions에서 DockerFile로 이미지를 빌드한다.
3. AWS ECR에 빌드한 이미지를 업로드한다.
4. AWS ECS를 통해서 롤링 배포된다.

각 단계에 대해서 살펴보겠습니다.

### 3.1. AWS 권한 - AssumeRole, STS 사용하기

기존 스크립트에서는 AWS Configure 과정에서 Github Actions 시크릿에 Key와 Secret Key를 보관했었는데요, 아무리 Github에서 Secret으로 관리를 한다고 하지만 Github에 비밀번호를 등록하지 않는 것이 좋을 것입니다. 남에게 집안일을 시키는데 집의 비밀번호를 알려주고 일을 시키는 것과, 집을 일시로 출입할 수 있는 출입 카드를 빌려주고 일을 시키는 것에 비유할 수 있습니다. 후자의 방식이 AWS IAM의 AssumeRole을 이용하는 방법입니다.

AssumeRole은 다른 AWS 계정 또는 리소스에 액세스할 수 있게 허가하는 역할과 같습니다. 정리하면 AssumeRole의 장점은 아래와 같습니다.

- 하나의 IAM 사용자 또는 외부 자격 증명을 여러 AWS 계정 또는 리소스에 대해 공유할 수 있습니다.
- Assume Role을 사용하여 AWS 계정 또는 리소스에 대한 액세스 권한을 조정할 수 있습니다.
- Assume Role을 사용하면 권한 부여를 세밀하게 할 수 있습니다.

```
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-region: ap-northeast-2
        role-session-name: GitHubActions
        role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

```

STS는 보안 토큰을 생성하는 서비스입니다. AWS STS를 사용하면 AWS IAM 사용자나 AWS 외부 자격 증명을 사용하여 액세스 권한을 부여할 수 있습니다. 유효기간이 있는 토큰이 발급되기 때문에 보안성이 더 뛰어납니다. Github Actions의 경우 OIDC(Open ID Connect) provider에서 STS 역할을 수행합니다. AWS의 Assume-Role의 신뢰 관계에 등록해둔 레퍼지토리에 대해 요청이 들어왔을 때, 보안 토큰을 발급해주고 일시적으로 서비스를 이용할 수 있게 권한을 부여할 수 있습니다. 

[**참고 자료 : Configuring OpenID Connect**](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

### 3.2. Github Actions CI

Github Actions 스크립트는 아래와 같습니다. Docker Image를 스크립트에 따라 빌드한 후, ECR에 Push합니다. 그리고 ECS에서 Task Definition을 가져오고, 이를 지정 서비스에 배포하는 흐름입니다.

```
...

    - name: Set up Docker Buildx
      id: buildx
      uses: docker/setup-buildx-action@v1
    
    - name: Build & Push image
      uses: docker/build-push-action@v2
      id: build-image
      with:
        platforms: linux/amd64
        tags: ${{ steps.login-ecr.outputs.registry }}/upbrella-ecr:latest
        push: true
        file: ./.github/workflows/Dockerfile
        builder: ${{ steps.buildx.outputs.name }}
        context: .
            
    - name: Get latest ECS task definition
      id: get-latest-task-def
      run: |
        TASK_DEF=$(aws ecs describe-services --cluster ${ECS_CLUSTER} --services ${ECS_SERVICE} --region ${AWS_REGION} --query "services[0].taskDefinition" --output text)
        aws ecs describe-task-definition --task-definition $TASK_DEF --region ${AWS_REGION} --query "taskDefinition" --output json > task-definition.json
        echo "TASK_DEF_JSON=$(pwd)/task-definition.json" >> $GITHUB_ENV

    - name: Fill in the new image ID in the Amazon ECS task definition
      id: task-def
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: ${{ env.TASK_DEF_JSON }}
        container-name: ${{ env.CONTAINER_NAME }}
        image: ${{ steps.login-ecr.outputs.registry }}/upbrella-ecr:latest

    - name: Deploy Amazon ECS task definition
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ env.ECS_SERVICE }}
        cluster: ${{ env.ECS_CLUSTER }}
        wait-for-service-stability: true
```


## 4. 마무리

배포 전략과 이를 토대로 업브렐라에서 Github Actions와 ECS 서비스로 CI/CD를 구축한 과정을 설명드렸는데요, 배포 전략은 각각이 장단점이 분명하니 인프라의 상황에 맞는 적절한 전략을 선택하는게 좋을 것 같습니다. CodeDeploy는 대부분의 배포 전략을 쉽게 적용할 수 있도록 지원한다고 하니 여러 방법을 적용해보고 좋은 방법을 선택해도 좋습니다.


## 참고자료, 출처
#### [매번 헷갈리는 CI/CD 배포 전략 정리해버리기](https://dev.classmethod.jp/articles/ci-cd-deployment-strategies-kr/)
#### [Blue/Green Deployments on AWS](https://d1.awsstatic.com/whitepapers/AWS_Blue_Green_Deployments.pdf)

#### [CanaryRelease](https://martinfowler.com/bliki/CanaryRelease.html?ref=wellarchitected)

#### [Configuring OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
