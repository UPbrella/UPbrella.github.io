---
title: nGrinder 자동화
layout: post
author: 임동현
subtitle: 'nGrinder 자동화'
date: 2023-09-21 00:00:00 +0900
categories:
- nGrinder
tags:
- nGrinder
---

## 1. 문제 정의

nGrinder를 사용해서 부하테스트를 하는 이유는 지난 게시글를 통해 알아보았습니다.

하지만 매번 배포를 할때마다 개발자가 nGrinder 서버를 띄워서 테스트 하는건 자원의 낭비라고 생각해서 자동화하기로 결정하였습니다.

## 2. nGrinder 자동화 도입

### 2-1 설정하기

java 11 과 docker가 설치되었다고 가정하고 진행하겠습니다.

도커로 ngrinder를 pull 하고 실행해줍니다.

```
sudo docker pull ngrinder/controller:3.5.5-p1
```

latest 버전이 아닌 3.5.5-p1을 명시해준 이유는 3.5.6 이상부터는 Script Error가 발생하고,

3.5.4 이하 버전은 agent Error가 빈번히 발생하여서 선택하였습니다.

https://github.com/naver/ngrinder/issues/940

```
sudo docker run -d -p 80:80 ngrinder/controller:3.5.5-p1
```

ec2의 public ip로 접속해보면 ngrinder가 실행중인 것을 볼 수 있습니다.

![image](https://user-images.githubusercontent.com/115435784/269597182-24acbe9c-8b03-4dfc-a607-b3f1e1539204.png)

이제 controller의 쉡 스크립트를 작성해보겠습니다.

```
#!/bin/bash
export EC2_SERVER_IP=52.79.180.25
source ~/.bashrc
sudo systemctl restart docker
sudo docker rm $(sudo docker ps -a -q)
sudo docker run -d -e EC2_SERVER_IP=$EC2_SERVER_IP -v ~/ngrinder-controller:/opt/ngrinder-controller -p 80:80 -p 16001:16001 -p 12000-12009:12000-12009 ngrinder/controller:3.5.5-p1
```

- 쉘 스크립트 설명
  1. **`#!/bin/bash`**
    - 이 줄은 스크립트가 Bash 쉘을 사용하여 실행되어야 함을 지정합니다.
  2. **`source ~/.bashrc`**
    - 사용자의 **`.bashrc`** 파일을 소스화하여, 해당 파일에 정의된 환경 변수와 함수를 현재 쉘 세션에 로드합니다.
  3. **`sudo systemctl restart docker`**
    - **`sudo`** 권한을 사용하여 Docker 시스템 서비스를 재시작합니다.
  4. **`sudo docker rm $(sudo docker ps -a -q)`**
    - 현재 모든 Docker 컨테이너의 ID를 가져와(**`sudo docker ps -a -q`**) 이를 사용하여 모든 Docker 컨테이너를 삭제합니다(**`sudo docker rm`**).
  5. **`sudo docker run -d -e EC2_SERVER_IP=$EC2_SERVER_IP -v ~/ngrinder-controller:/opt/ngrinder-controller -p 80:80 -p 16001:16001 -p 12000-12009:12000-12009 ngrinder/controller`**
    - **`sudo`** 권한으로 새 Docker 컨테이너를 실행합니다. 여기에서 사용하는 옵션은 다음과 같습니다:
      - **`d`**: Docker 컨테이너를 백그라운드에서 실행합니다 (데몬 모드).
      - **`e EC2_SERVER_IP=$EC2_SERVER_IP`**: **`EC2_SERVER_IP`** 환경 변수를 컨테이너 내부의 환경 변수로 설정합니다. **`EC2_SERVER_IP`**는 스크립트 실행 전에 쉘 환경에 설정되어 있어야 합니다.
      - **`v ~/ngrinder-controller:/opt/ngrinder-controller`**: 호스트 시스템의 **`~/ngrinder-controller`** 디렉터리를 컨테이너의 **`/opt/ngrinder-controller`**에 마운트합니다.
      - **`p 80:80`**: 호스트 시스템의 80번 포트를 컨테이너의 80번 포트에 매핑합니다.
      - **`p 16001:16001`**: 호스트 시스템의 16001번 포트를 컨테이너의 16001번 포트에 매핑합니다.
      - **`p 12000-12009:12000-12009`**: 호스트 시스템의 12000번에서 12009번 포트를 컨테이너의 동일한 포트 범위에 매핑합니다.
      - **`ngrinder/controller`**: 사용할 Docker 이미지를 지정합니다.

agent 설치

```
sudo docker pull ngrinder/agent:3.5.5-p1
```

agent도 controller와 동일한 이유로 3.5.5-p1을 선택하였습니다.

```
#!/bin/bash
source ~/.bashrc
export CONTROLLER_IP=52.79.180.25
sudo systemctl restart docker
sudo docker rm $(sudo docker ps -a -q)
sudo docker run -v ~/ngrinder-agent:/opt/ngrinder-agent -d ngrinder/agent:3.5.5-p1 $CONTROLLER_IP:80
```

agent를 실행한 후 agent 관리로 들어가면 agent가 등록된 것을 확인할 수 있습니다.

![image](https://user-images.githubusercontent.com/115435784/269597308-1897261e-2ea7-4452-b747-c840dd3990f9.png)

- nGrinder controller 와 agent 분리 이유
  1. **리소스 경쟁**: **`controller`**와 **`agent`**가 같은 시스템 리소스(예: CPU, 메모리)를 사용하므로, 특히 테스트 중에 성능 저하가 발생할 수 있습니다.
  2. **고장 격리**: **`controller`** 또는 **`agent`** 중 하나에 문제가 발생하면, 다른 컴포넌트도 영향을 받을 수 있습니다. 두 컴포넌트를 분리하여 실행하면, 하나의 컴포넌트에 문제가 발생해도 다른 컴포넌트에는 영향을 미치지 않습니다.

### 2 - 2 GitHub Action

upbrella 개발팀은 dev 서버는 code deploy를 활용해서 배포하고 있습니다.

nGrinder에 대한 설명은 CodeDeploy 이후 부분부터 입니다.

```
name: Upbrella DEV CI

on:
  push:
    branches: [ "release-dev" ]

env:
  WORKING_DIRECTORY: ./
  CODE_DEPLOY_APPLICATION_NAME: upbrella-dev-deploy
  CODE_DEPLOY_DEPLOYMENT_GROUP_NAME: UpbrellaServerDev
  S3_BUCKET_NAME: upbrella-storage

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:

      - name: checkout
        uses: actions/checkout@v3

      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'adopt'

      - name: Set application properties
        run: |
          touch src/main/resources/application.properties
          echo "${{ secrets.APPLICATION_PROPERTIES_DEV }}" > src/main/resources/application.properties
          echo "${{ secrets.APPLICATION_PROPERTIES_TEST }}" > src/test/resources/application.properties

      - name: Build with Gradle
        run: |
          chmod +x gradlew
          ./gradlew clean build
        env:
          WORKING_DIRECTORY: ${{ env.WORKING_DIRECTORY }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Make zip file
        run: zip -r ./$GITHUB_SHA.zip .
        shell: bash

      - name: Upload to S3
        run:
          aws s3 cp $GITHUB_SHA.zip s3://${{ env.S3_BUCKET_NAME }}/server-dev-deploy/$GITHUB_SHA.zip --region ${{ secrets.AWS_REGION }}

      - name: Code Deploy
        run: |
          aws deploy create-deployment \
          --deployment-config-name CodeDeployDefault.AllAtOnce \
          --application-name ${{ env.CODE_DEPLOY_APPLICATION_NAME }} \
          --deployment-group-name ${{ env.CODE_DEPLOY_DEPLOYMENT_GROUP_NAME }} \
          --s3-location bucket=${{ env.S3_BUCKET_NAME }},bundleType=zip,key=server-dev-deploy/$GITHUB_SHA.zip

			# 여기서부터 nGrinder 설정입니다. 
      # test에 필요한 ec2 인스턴스를 aws cli를 통해 실행
      - name: Start nGrinder EC2 - (Controller, Agent)
        run: aws ec2 start-instances --instance-ids ${{ secrets.AWS_EC2_NGRINDER_CONTROLLER }} ${{ secrets.AWS_EC2_NGRINDER_AGENT }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}

      # Dev server 완전히 실행될 때까지 기다림
      - name: Sleep for 30 seconds - waiting controller,agent run
        run: sleep 30s
        shell: bash

      # EC2- Ngrinder Controller를 실행
      - name: nGrinder Controller Start
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.AWS_EC2_CONTROLLER_IP }}
          username: ubuntu
          key: ${{ secrets.AWS_SECRET_PEM }}
          script: |
            bash ./controller.sh

			# nGrinder Agent 실행
      - name: nGrinder Agent Start
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.AWS_EC2_AGENT_IP }}
          username: ubuntu
          key: ${{ secrets.AWS_SECRET_PEM }}
          script: |
            bash ./agent.sh

      - name: waiting controller,agent run
        run: sleep 30s
        shell: bash

			# 업브렐라 서비스의 방문자 통계를 따라 최대 30명의 사용자가 동시접속한다고 가정하였습니다. 
      - name: nGrinder Test
        uses: fjogeleit/http-request-action@v1
        with:
          url: 'http://${{ secrets.AWS_EC2_CONTROLLER_IP }}/perftest/api'
          method: 'POST'
          username: ${{ secrets.NGRINDER_ADMIN_USERNAME }}
          password: ${{ secrets.NGRINDER_ADMIN_PASSWORD }}
          customHeaders: '{"Content-Type": "application/json"}'
          data: '{"param" : "${{ secrets.AWS_EC2_TEST_SERVER_IP }}", "testName" : "upbrella-test", "tagString" : "upbrella-test", "description" : "upbrella-test", "scheduledTime" : "", "useRampUp": false, "rampUpType" : "PROCESS", "threshold" : "D", "scriptName" : "upbrella-test.groovy", "duration" : 240000, "runCount" : 1, "agentCount" : 1, "vuserPerAgent" : 30, "processes" : 2, "rampUpInitCount" : 0, "rampUpInitSleepTime" : 0, "rampUpStep" : 1, "rampUpIncrementInterval" : 1000, "threads": 15, "testComment" : "upbrella-test", "samplingInterval" : 2, "ignoreSampleCount" : 0, "status" : "READY"}'
          timeout: '60000'

      - name: waiting test for 300 seconds
        run: sleep 300s
        shell: bash

      # Ngrinder Rest Api 를 통해 테스트 결과 조회
      - name: Get nGrinder test result ...
        uses: satak/webrequest-action@master
        id: NgrinderTestResult
        with:
          url: 'http://${{ secrets.AWS_EC2_CONTROLLER_IP }}/perftest/api?page=0'
          method: GET
          username: ${{ secrets.NGRINDER_ADMIN_USERNAME }}
          password: ${{ secrets.NGRINDER_ADMIN_PASSWORD }}

      - name: send test result to slack
        uses: 8398a7/action-slack@v3
        with:
          text: '${{ steps.NgrinderTestResult.outputs.output }}'
          status: ${{ job.status }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always() # Pick up events even if the job fails or is canceled.

      - name: Stop Ngrinder EC2 - (Controller, Agent, Test Server)
        run: aws ec2 stop-instances --instance-ids ${{ secrets.AWS_EC2_NGRINDER_CONTROLLER }} ${{ secrets.AWS_EC2_NGRINDER_AGENT }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}
```

이제 배포가 완료되면 nGrinder를 위한 EC2 2대를 실행하고 부하테스트를 실행하게 되었습니다.

![image](https://user-images.githubusercontent.com/115435784/269597500-8a5890d6-8583-47c8-b4e5-ebadeaecc0d7.png)

테스트의 결과는 Slack 알림을 통해서 받아볼 수 있도록 설정하였습니다.
![image](https://user-images.githubusercontent.com/115435784/269597560-eb15c332-9a31-42ef-9d12-1539093d334f.png)

## 3. 마무리

이번 게시글을 통해 nGrinder 자동화하는 방법에 대해서 알아보았습니다. 현재 플로우는 업브렐라 api중 복잡한 쿼리만을 테스트하고 있습니다.

다음 게시글을 통해 실제 유저 플로우대로 진행하는 방법에 대해 알아보도록 하겠습니다.
