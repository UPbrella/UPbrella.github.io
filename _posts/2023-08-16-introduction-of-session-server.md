---
title: 분산 서버 환경에서 세션 로그인
layout: post
author: 임동현
subtitle: 'Redis를 이용한 세션 서버'
categories:
- Login
tags:
- Session
- Redis
published: true
---

안녕하세요,
저는 업브렐라에서 백엔드 개발을 담당하고 있습니다.
이번 글에서는 분산 서버 환경에서 세션 로그인을 구현하는 방법에 대해 알아보겠습니다.

## 1. 문제 정의

업브렐라 서비스의 특성 상 비가 올 경우 사용자가 급증하기 때문에 분산 서버 환경을 구성하고 있습니다. 하지만 분산 서버 환경에서 세션 로그인 방법을 사용한다면 한 가지 문제점을 맞이하게 됩니다.

<img src="https://user-images.githubusercontent.com/115435784/260866282-d61f4945-798b-4376-83d3-e7eaad9a7f9a.png" width="600" height="600"/>

각 Spring Application은 자신만 접근할 수 있는 장소(일반적으로 서버의 메모리)에 세션을 저장하고 있지만, 분산 환경에서는 문제가 될 수 있습니다.

Spring Application #2 가 세션 #3의 요청을 받는 상황을 상상해보면, 세션 데이터가 Spring Application #1의 메모리에 저장되어 있기 때문에, 애플리케이션이 세션 데이터를 읽을수가 없게 됩니다.  

## 2. Session Storage

이 문제를 해결하기 위해서는 Redis와 같은 일종의 공유 세션 스토리지를 구현해야 합니다.

<img src="https://user-images.githubusercontent.com/115435784/260866385-ab6e3b53-acba-4e61-b388-90a16233c3fc.png" width="600" height="600"/>


이렇게 공유 세션 저장소를 사용하게 된다면, 분산 서버 환경에서의 문제점을 해결할 수 있습니다. 

## 3. Spring Session Data Redis

이제 Spring Redis Session 을 적용해서 분산 서버 환경에서의 로그인 문제를 해결하는 방법에 대해 알아보겠습니다.

### 3-1. Redis 설치

Redis를 이용할 서버에 접속해 다음 과정을 거칩니다.

```
sudo apt-get update 
sudo apt install redis-server
```

Redis를 기본적으로 설치하면, localhost만 접속가능하도록 설정되어 있습니다. spring 서버에서 접근할 수 있도록 설정을 변경 하겠습니다.

```
sudo nano /etc/redis/redis.conf
```

설정파일에 들어간 후 bind 부분을 찾아본다면 다음과 같이 되어있을 것입니다.

```
bind 127.0.0.1 ::1
```

**`::1`**은 IPv6에서 "localhost" 또는 "loopback" 주소를 나타냅니다. IPv4에서의 **`127.0.0.1`**과 동일한 역할을 합니다.

위의 주소를 spring 서버 주소로 변경합니다. (여러개의 IP 주소를 허용하고 싶은 경우 IP 주소 사이에 공백을 추가합니다.

### 3-2. build.gralde 설정

```
		// Spring Data Redis
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    // Spring Session Data Redis
    implementation 'org.springframework.session:spring-session-data-redis'
```

build.gradle 에 다음과 같이 의존성을 추가해줍니다.

### 3-3. Redis Session 설정

서버에서 사용하고 있는 레디스에 접근하기 위해 application.yml 파일에 설정을 추가합니다.

```
spring:
  redis:
    host: your-redis-hostname-or-ip
    port: 6379 # Redis의 기본 포트입니다. 변경한 경우 여기를 수정하세요.
    password: your-redis-password # 비밀번호를 설정한 경우 입력하세요.
    timeout: 60000 # 필요에 따라 타임아웃을 설정할 수 있습니다.
  session:
    store-type: redis
    timeout: 3600s # 세션의 타임아웃을 설정합니다. 예를 들어, 1시간 동안 유효합니다.
```

다음으로 Redis를 세션 서버로 사용하기 위한 설정 클래스를 Bean으로 등록합니다.

```java
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
@Configuration
public class HttpSessionConfig {

}
```

이제 기존과 동일하게 세션을 사용한다면 Redis Server에 저장되게 됩니다.

```
session.setAttribute("key", "value");
```

### 3-4. Redis Session 테스트

위의 설정이 잘 적용되었는지 직접 확인해보겠습니다.

```java
@GetMapping("/session")
    public String sessionTest(HttpSession session) {
        
        session.setAttribute("test", 123);
        return "test";
    }
```

위의 임시 코드를 활용해서 session에 값을 넣어보겠습니다.

값을 넣은 후 redis server에서 session을 조회해보면 다음의 결과를 얻을 수 있습니다.

```
KEYS spring:session:*
1) "spring:session:sessions:495e1547-ea28-4262-b288-ab76e342f427"
2) "spring:session:expirations:1692144180000"
3) "spring:session:sessions:expires:495e1547-ea28-4262-b288-ab76e342f427"
```

1. **"spring:session:expirations:TIMESTAMP"**:
  - 이 키는 세션의 만료 시간과 관련이 있습니다. TIMESTAMP는 해당 시간에 만료될 세션의 ID를 포함하고 있습니다.
  - 세션이 만료될 때 Redis에서 해당 세션 데이터를 자동으로 삭제하는 데 사용됩니다.

2. **"spring:session:sessions:SESSION_ID"**:
  - 이 키는 실제 세션 데이터를 포함하는 해시입니다.
  - **`HGETALL`** 명령을 사용하면 해당 세션의 모든 데이터를 조회할 수 있습니다.
  - 예:

      ```
      HGETALL spring:session:sessions:06e5030c-679b-47bb-9d1e-5ef6a0e1bda4
      ```


1. **"spring:session:sessions:expires:SESSION_ID"**:
  - 이 키는 세션의 TTL(Time To Live)을 나타냅니다.
  - **`TTL`** 명령을 사용하면 해당 세션의 남은 생명 시간을 초 단위로 확인할 수 있습니다.
  - 예:

      ```
      TTL spring:session:sessions:expires:06e5030c-679b-47bb-9d1e-5ef6a0e1bda4
      ```


HGETALL 명령을 이용하면 세션 키의 값을 조회할 수 있지만, 데부분의 세션 데이터는 직렬화된 형태로 저장하므로, 직접적인 값의 이해가 어려울 수 있습니다. 이러한 값들은 Spring Session 을 이용하게 되면 다시 보기 쉬운 형태로 값이 조회됩니다.

세션에 저장된 “test” 라는 키의 값을 보려면, 아래와 같은 명령을 실행하면 조회할 수 있습니다.

```
hget spring:session:sessions:495e1547-ea28-4262-b288-ab76e342f427 sessionAttr:test
```
