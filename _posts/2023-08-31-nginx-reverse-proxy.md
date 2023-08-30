---
title: nginx 리버스 프록시 설정
layout: post
author: 남권우
subtitle: 'nginx 리버스 프록시 설정하기'
date: 2021-08-31 00:00:00 +0900
categories:
- Infrastructure
tags:
- Infrastructure
- nginx
published: true
---

안녕하세요,
업브렐라 팀의 백엔드 남권우입니다.

## 1. 리버스 프록시란?

업브렐라는 React로 클라이언트 사이드 렌더링을 하고 있으며, 최종적으로 아래와 같은 인프라 구조를 갖추었습니다.

<img data-action="zoom" src='{{ "/resources/2023-08-19-01.png" | relative_url }}' alt='absolute'>

기존에는, 클라이언트 사이드 렌더링을 하기 때문에, 클라이언트에서 API 요청을 보낼 때, API 서버로 클라이언트에서 직접 요청을 보내는 구조였습니다.
따라서 보안적인 취약점이 발생하였는데요, 이를 해결하기 위해 nginx를 이용한 리버스 프록시 설정을 하게 되었습니다.

리버스 프록시란, 클라이언트와 서버 사이에서 중계기로서 동작하는 서버를 말합니다.
클라이언트가 서버에 접속할 때, 클라이언트는 중계기에 접속하고 중계기가 서버에 접속하여 클라이언트와 서버를 연결시켜주는 역할을 합니다.

이 뿐만 아니라 nginx로 리버스 프록시를 설정했을 때 많은 장점이 있어 소개하고자 합니다.

## 2. 리버스 프록시의 장점

#### CORS 문제 해결
클라이언트에서 API 요청을 보낼 때, 정적 페이지 소스와 API 요청 소스가 달라져 CORS 문제가 발생합니다.
nginx를 이용한 리버스 프록시 설정을 하면, 정적 페이지 소스와 API 요청이 모두 nginx를 거쳐서 반환되기 때문에 동일 출처로 인식하여 CORS 문제를 해결할 수 있습니다.

#### SSL 인증서 적용

기존의 인프라 구조에서는, 클라이언트에서 API 요청을 보낼 때, API 서버로 클라이언트에서 직접 요청을 보내는 구조였기 때문에, API 서버에도 SSL 인증서를 적용해야 했습니다.
API 서버에 인증서를 적용하지 않으면 **mixed content** 문제가 발생하였는데, nginx로 프록시 포워딩을 하면 같은 지점으로 트래픽이 진입하기 때문에 인증서를 맨 앞단의 로드밸런서에만 적용해주면 되었습니다.
뿐만 아니라 로드밸런서에 적용하지 않더라도 nginx 자체에서 SSL 인증서를 적용할 수도 있다고 합니다.

#### 로드 밸런싱

nginx 자체에서도 로드 밸런싱을 지원합니다. 따라서 nginx를 이용한 리버스 프록시 설정을 하면, 로드 밸런싱을 적용할 수 있습니다.
업브렐라에서는 ALB 2대를 운영하고 있기 때문에 nginx의 로드밸런싱 기능은 적용하지 않았습니다. 

#### 보안

nginx를 거쳐 API 서버에 접속하기 때문에, API 서버의 IP를 외부에 노출시키지 않아 보안적인 측면에서도 좋습니다.

#### 캐싱

nginx는 캐싱 기능을 지원합니다. 따라서 nginx를 이용한 리버스 프록시 설정을 하면, 캐싱 기능을 적용할 수 있습니다.
업브렐라는 redis 캐시 서버를 운영하고 있습니다. nginx에서 redis 옵션을 설정하면 연결된 redis에서 캐시를 설정하게 할 수 있습니다.

#### 라우팅

프록시 포워드가 가능하여, 요청 정보에 따라 API 서버로 보낼 것인지, 정적 리소스를 반환할 것인지 결정할 수 있습니다.

#### 서버 성능 개선

nginx는 비동기 이벤트 기반으로 동작하기 때문에, 서버의 성능을 개선할 수 있습니다.
뿐만 아니라, 위에서 언급한 캐싱 기능을 적용하면, API 서버의 부하를 줄여 전체적인 인프라 성능 개선에 도움이 됩니다.

## 3. 리버스 프록시 설정

nginx를 이용한 리버스 프록시 설정을 해보겠습니다.

먼저, 도메인 이름을 기준으로 정적 리소스를 반환하도록 설정하였습니다.

```nginx
server {
    listen 80;
    server_name www.upbrella.co.kr upbrella.co.kr;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    location / {

        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```

만약 도메인 이름을 기준으로 API 서버로 요청을 보내고 싶다면, 다음과 같이 설정하면 됩니다.

```nginx
server {
    listen 80;
    server_name api.upbrella.co.kr;

    # 로그 설정
    access_log /var/log/nginx/access_api.log;
    error_log /var/log/nginx/error_api.log;
    
    location / {
        # 로드 밸런서로 요청 전달
        proxy_pass {{ Private Load Balancer DNS Name }};
        
        # Host 헤더를 설정합니다.
        proxy_set_header Host $host;
        
        # nginx에서 프록시 된 요청임을 나타냅니다.
        proxy_set_header X-Nginx-Proxy true;
        
        # 실제 클라이언트의 IP를 기록하는 헤더
        proxy_set_header X-Real-IP $remote_addr;
        
        # 다중으로 Proxy되는 경우 실제 IP를 기록하는 헤더
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # 캐시 옵션을 설정합니다.
        add_header 'Cache-Control' 'no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0';
        
        charset utf-8;
    }
}
```

위의 설정을 통해, api.upbrella.co.kr로 요청을 보내면, nginx는 로드 밸런서에 요청을 보내고, 로드 밸런서는 API 서버로 요청을 보내게 됩니다.

참고로, 위 옵션 중에서, `X-Real-IP` 헤더와 `X-Forwarded-For` 헤더가 있습니다.
이는, nginx를 거쳐 API 서버로 요청이 전달되는 경우, API 서버에서는 nginx의 IP를 클라이언트의 IP로 인식하기 때문에, 실제 클라이언트의 IP를 기록하기 위해 설정하는 것입니다.
`X-Forwarded-For`는 다중으로 프록시가 되는 경우, 실제 클라이언트의 IP를 기록하기 위해 설정하는 것입니다.
클라이언트와 모든 중간 프록시 서버의 IP 주소를 컴마로 구분하여 나열한 목록을 가지고 있습니다.
클라이언트의 IP 주소가 첫 번째로 오며, 그 뒤로는 요청이 거쳐간 각 프록시 서버의 IP 주소가 순서대로 나옵니다. 이를 통해 웹 서버는 요청이 어떤 경로를 거치며 왔는지를 파악할 수 있습니다.

예를 들면, 다음 순서로 요청이 흘러간다고 생각해보겠습니다.
```
111.111.111.111(Client) ->

222.222.222.222(Proxy 1) ->

333.333.333.333(Proxy 2) ->

444.444.444.444(WAS) 
```
이 경우, WAS가 인식하는 IP는 최종 리버스 프록시인 333.333.333.333일 것입니다.
그러나, `X-Forwarded-For`를 확인하면, 클라이언트부터 최종 프록시 까지의 IP 주소가 순서대로 나열된 111.111.111.111 222.222.222.222 333.333.333.333을 받게 됩니다.
그리고, `X-Real-IP`에서는 최초의 Client IP인 111.111.111.111을 확인할 수 있습니다.

이 외에도 nginx 설정은 다양합니다. 설정에 따라 성능적으로 큰 차이를 낼 수 있다고 하니, 지속적으로 모니터링하면서 최적화가 필요합니다.

[nginx 문서](https://nginx.org/en/docs/)를 참고하시면 더 자세한 내용을 확인하실 수 있습니다.
