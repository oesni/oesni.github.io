---
title:  "Docker로 간단하게 reverse proxy 설치하기"
excerpt: "GitHub Blog 서비스인 github.io 블로그 시작하기로 했다."

categories:
  - Network
tags:
  - Network
last_modified_at: 2022-05-23T21:26:00+0900
published: false
---

Nginx Proxy Manager를 사용하면 간단하게 reverse proxy 서버를 구성할 수 있습니다. <br>
그러면 reverse proxy는 무엇이고 forward proxy랑 어떻게 다를까요? <br>
[CLOUDFLARE - What is a reverse proxy? | Proxy servers explained](https://www.cloudflare.com/ko-kr/learning/cdn/glossary/reverse-proxy/)

CLOUDFALRE는 proxy를 다음과 같이 설명하고 있습니다.

> **What is a reverse proxy?** <br><br>
리버스 프록시는 웹 서버 앞에 위치하여 클라이언트(웹 브라우저 등)의 요청을 해당 웹 서버로 전달하는 서버입니다. 일반적으로 리버스 프록시는 보안, 성능, 안정성을 향상시키기 위해 구현되었습니다.

> **What's a proxy server?** <br><br>
proxy, proxy server 또는 web proxy라 불리는 forward proxy는 다수의 클라이언트 기기 앞에 존재합니다. 해당 컴퓨터들이 인터넷 상의 사이트와 서비스들에 요청을 보낼때, 프록시 서버는 요청을 가로채서 클라이언트를 대신해 서버와 통신합니다.

proxy 서버는 요청을 가로채서 대신 통신하는 역할을 하는 서버입니다. forward proxy와 reverse proxy 모두 클라이언트와 서버 사이에 위치합니다. 그렇다면 forward proxy와 reverse proxy는 어떻게 다를까요?? 차이는 누구를 대신해서 통신을 하느냐입니다. forward proxy는 클라이언트의 대신 서버와 통신하는 역할을 하고, reverse proxy는 서버 대신 클라이언트와 통신을 합니다.

따라서 reverse proxy를 통해 구성된 서버는 클라이언트와 직접 통신을 하지 않고, 클라이언트는 proxy(reverse proxy)서버와만 직접 통신을 하게 됩니다. reverse proxy를 사용하면 다음과 같은 장점이 있습니다.
- 로드밸런싱 (Load balancing)
- 공격으로부터 보호 (Proection from attacks)
- 글로벌 서버 로드 밸런싱(Global Server Load Balancing, GSLB)
- 캐싱 (Caching)
- SSL 암호화 (SSL encryption)

Nginx Proxy Manager를