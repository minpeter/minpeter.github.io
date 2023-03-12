---
title: 총정리 - traefik proxy
date: 2023-3-12 22:12:8 +0900
tags: ["traefik", " homelab", " server"]
categories: "home lab"
---

글을 시작하기 앞서 [https://doc.traefik.io/traefik/](https://doc.traefik.io/traefik/) 의 내용 중 일부를 재구성했음을 알림미다.

2023년 3월 기준 v3.9.8이 최신버전이고 이를 기준으로 작성하였다.

## traefik란?

본인의 인프라에 서비스 게시를 편하게 해주는 Edge Router이다.

Edge Router는 외부네트워크와 내부네트워크를 연결할 수 있도록 그 경계에 존재하는 특수한 라우터이다.

들어오는 모든 트래픽을 우선 받은 후 규칙에 따라 적절한 서비스로 연결해주는 역활을 담당한다.

## 글에서 다루는 범위

traefik proxy는 생각보다 훨씬 다양한 기능을 가지고 있다.

서비스를 연결하기 위해 [사용하는 수단과 방법 또한 다양하게 제공](https://doc.traefik.io/traefik/providers/overview/)한다.

HTTP는 물론이고 4계층인 TCP와 UDP도 지원한다.

하지만 이 글에서는 도커 및 파일 방식을 이용해 배포되는 HTTP 및 HTTPS 서비스 연결에 대한 일부를 이야기 해보려한다.

## 개념

앞에서 말했듯 traefik proxy는 에지 라우터이다.

서버의 문이며 들어오는 모든 요청을 가로채서 어떤 서비스가 어떤 요청을 처리하는지 결정하는 규칙(경로, 호스트, 헤더)을 통해 라우팅한다.

이때 서비스는 파일 또는 도커를 이용한 자동 감지 외 다양한 방법을 이용할 수 있고 이를 [providers](https://doc.traefik.io/traefik/providers/overview/)라고 부른다.

또 그저 규칙에 따라 서비스에 라우팅 하는 것이 아닌 미들웨어를 통해 접근 제어나 요청을 서비스에 전달하기 전에 요청을 업데이트 할 수 있다.

## 로컬에서 동작 이해

글의 마지막에서는 실제 서버에 (\*라즈베리파이) 클라우드플레어를 연결하여 설정하게 될 것이다.

하지만 서버에 올리기 앞서 지금 이 글을 보고 있는 노트북에서 간단한 실험을 통해 traefik의 동작 방식을 설명해보려 한다.

노트북에 도커가 설치돼 있다는 전재하에 설명을 진행하도록 하겠다.

우선 traefik-test 폴더를 만들어 해당 폴더 아래에 docker-compose.yaml 파일을 생성해주자.

```yaml
version: "3"

services:
  traefik-proxy:
    # traefik v2 공식 도커 이미지
    image: traefik:v2.9
    # 웹 UI를 활성화하고 Traefik에게 도커를 수신하도록 한다.
    command: --api.insecure=true --providers.docker
    ports:
      # HTTP로 들어오는 요청을 받기 위한 포트
      - "80:80"
      # Traefik WEB UI 포트 (--api.insecure=true 옵션에 의해 활성화됨)
      - "8080:8080"
    volumes:
      # Traefik가 Docker 이벤트를 수신할 수 있도록 설정
      - /var/run/docker.sock:/var/run/docker.sock
```

이제 다음 명령으로 traefik를 실행시킬 수 있다.

```bash
docker-compose up -d traefik-proxy
```

브라우저를 열고 [http://localhost:8080/dashboard](http://localhost:8080/dashboard) 에 접속하면 traefik의 웹 UI를 확인할 수 있다.

![Untitled](/images/traefik-1/image.png)

현재는 traefik만 있고 서비스가 없는 상태이다.

docker-compose.yaml 파일을 다음과 같이 수정해보자.

```yaml
version: "3"

services:
  traefik-proxy:
    image: traefik:v2.9
    command: --api.insecure=true --providers.docker
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  iplogger:
    #접속 IP등의 주소를 보여주는 컨테이너 노출(서비스 추가)
    image: minpeter/iplogger
    labels:
      - "traefik.http.routers.iplogger.rule=Host(`ip.docker.localhost`, `ip.traefik.me`)"
```

다음 명령으로 iplogger 서비스를 시작해주자.

```yaml
docker-compose up -d iplogger
```

이제 traefik를 통해 iplogger 서비스에 접속해보도록 하겠다.

```bash
curl -H Host:ip.docker.localhost 127.0.0.1
```

위의 명령어는 http 요청을 localhost:80으로 하나 전송하는데 http 헤더에 Host 필드를 추가하고 이를 ip.docker.localhost로 요청한다.

아마 다음과 같은 화면이 보일 것이다.

![Untitled](/images/traefik-1/image1.png)

80포트는 분명 traefik 컨테이너에 바인딩 되어 있다.

하지만 요청은 Host 헤더에 의해 iplogger 컨테이너로 라우팅되어 정상적으로 응답이 돌아왔다.

- 번외 : traefik.me
  Host 헤더에 대한 이해를 돕기 위해 설명하려 한다.
  위에서는 Host 헤더를 직접 수정하여 요청을 전송하였지만 유저가 이런 요청을 보내기에는 다소 무리가 있다.
  따라서 DNS 설정을 하여 도메인으로 접속을 하게 될 텐데 이렇게 하면 Host 헤더가 자동으로 생성된다.
  이걸 연습해볼 수 있는 방법이 바로 traefik.me이다.
  traefik.me는 무조건 127.0.0.1라고 답하는 도메인이다.
  ![Untitled](/images/traefik-1/image2.png)
  [ip.traefik.me](http://ip.traefik.me)의 경우에도 dns 조회 결과는 127.0.0.1 이다.
  이를 이용하면 다음과 같이 요청을 보내 볼 수 있다.
  ```bash
  curl ip.traefik.me
  ```
  ![Untitled](/images/traefik-1/image3.png)
  브라우저로 [http://ip.traefik.me/](http://ip.traefik.me/) 에 접속하면 다음과 같은 웹 페이지도 볼 수 있다.
  ![Untitled](/images/traefik-1/image4.png)

이제 로컬에서 해볼 수 있는 기초 과정을 마쳤다.

## 실제 서버에서는?

약간 더 복잡하다.

방금 이용한 docker provider를 이용한 서비스 자동 감지는 같은 네트워크 도커 네트워크에서 컨테이너가 실행 중 일때만 가능하고 소소하게 신경 써야 되는 Access logging, Dashboard 접근 제어 미들웨어, file provider 설정, HTTPS 인증서 자동 발급등의 설정이 있다.

하나씩 알아볼테니 놓치지 않고 따라오도록 하자.

## 외부에서 서버로!

우선 서버에는 운영체제와 도커가 설치되어 있어야 한다.

또한 SSH나 기본적인 설정은 완료가 됬다고 가정한다.

DNS는 무엇을 사용해도 상관없지만 글에서는 클라우드플레어를 이용하도록 하겠다.

도메인이 example.com이라면 레코드에 다음을 추가해주자.

| A record | \*.example.com | YOUR SERVER IP |
| -------- | -------------- | -------------- |
| A record | example.com    | YOUR SERVER IP |

[YOUR SERVER IP]를 알아내는 방법은 서버 터미널에서 다음명령어를 이용할 수 있다.

```bash
curl ifconfig.me
# or
curl ip.minpeter.cf -L
```

또 traefik로 들어가는 트래픽을 위해 공유기나 방화벽을 사용 중이라면 80, 443 포트를 개방시켜주어야 한다. ~~(알아서 하자)~~

만약 클플을 사용한다면 Proxied는 켜두고 \***\*Your SSL/TLS encryption mode is Full (strict) 옵션은 Full (strict)으로 설정하자.\*\***

![Untitled](/images/traefik-1/image5.png)

~~뭐 사실 그냥 Flexible 박고 https 인증서 발급 설정을 무시해도 된다. ㅋㅋㅋㅋ~~

마지막으로 나중에 인증서 발급을 위해 http challenge를 사용하는데 이를 위해 페이지 옵션을 주자.

설정은 Rules > Page Rules > Create Page Rule에서 할 수 있다.

URL에는 `*.example.com//.well-known/acme-challenge/*` 를 입력해주고 Pick a Setting (required)은 SSL > Off 로 설정해주자.

![Untitled](/images/traefik-1/image6.png)

이제 나름 안전하게 80, 443 포트로 외부에서 접근이 가능해졌고 또한 이후 진행할 인증서 관련 설정도 마쳤다.

물론 클플 이외의 DNS를 이용한다면 레코드만 설정해주면 된다. ~~(대신 공격은 못막죠)~~

## 제로부터 특정 서비스 접근 제한까지

서버에 SSH나 원하는 방법으로 접속해준다.

traefik 폴더를 만들고 docker-compose.yaml 파일을 생성해준다.

이후 설정파일에서 나오는 minpeter.cf는 모두 본인의 도메인으로 대체하면 된다.

```bash
version: '3.8'
services:
  traefik:
    image: traefik:v2.9
    container_name: traefik
    restart: unless-stopped
    labels:
      - traefik.enable=true
      - traefik.http.routers.traefik.rule=Host(`traefik.minpeter.cf`) && PathPrefix(`/api`, `/dashboard`)
      - traefik.http.routers.traefik.entrypoints=websecure
      - traefik.http.routers.traefik.tls.certresolver=myresolver
      - traefik.http.routers.traefik.service=api@internal
    ports:
      - 80:80
      - 443:443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yaml:/etc/traefik/traefik.yaml
      - ./external_services:/external_services
      - traefik-letsencrypt:/letsencrypt
    networks: [traefik]

  iplogger:
    image: minpeter/iplogger
    restart: unless-stopped
    container_name: iplogger-service
    labels:
      - traefik.enable=true
      - traefik.http.routers.iplogger.rule=Host(`ip.minpeter.cf`)
      - traefik.http.routers.iplogger.entrypoints=websecure
      - traefik.http.routers.iplogger.tls.certresolver=myresolver
    networks: [traefik]

networks:
  traefik:
    external: true

volumes:
  traefik-letsencrypt:
```

처음 보는 설정이 많아져서 당황할 수 도 있겠지만 기본적으로 처음 했던 실습에 HTTPS 설정과 인증서 발급 관련, WEB UI 접근 방식을 바꾸었다.

우선 기존에 8080 포트로 접근하였던 WEB UI를 traefik.example.com을 통해 접근하도록 변경하였다.

```bash
# docker-compose.yaml 중 traefik service의 labels
- traefik.http.routers.traefik.rule=Host(`traefik.minpeter.cf`) && PathPrefix(`/api`, `/dashboard`)
- traefik.http.routers.traefik.service=api@internal
```

또 SSL 인증서를 위해 다음과 같은 설정이 추가 되었다.

```bash
version: '3.8'
services:
  traefik:

...

labels:
..
    - traefik.http.routers.traefik.entrypoints=websecure
    - traefik.http.routers.traefik.tls.certresolver=myresolver

...

volumes:
  traefik-letsencrypt:
```

볼륨 부분을 보면 3가지가 추가된 것을 볼 수 있다.

```bash
traefik:
  volumes:
	...
	      - ./traefik.yaml:/etc/traefik/traefik.yaml
	      - ./external_services:/external_services
	      - traefik-letsencrypt:/letsencrypt

...

volumes:
  traefik-letsencrypt:
```

traefik.yaml은 기존의 command 를 대신해 파일로 정적 설정을 하는 용도이고 곧 설정할 것이다.

external_services 폴더는 file provider를 위해 미리 연결해두었고 traefik-letsencrypt 볼륨은 발급받은 인증서를 저장하기 위해 사용한다.

마지막으로 네트워크 관련된 부분이다.

```bash
#서비스 마다
networks: [traefik]

#이 컴포즈 스택에서는 외부 도커 네트워크를 사용하겠습니다~
networks:
  traefik:
    external: true
```

라는 의미이다.

볼륨으로 연결해준 traefik.yaml 파일과 external_services 폴더를 생성해주자.

후 traefik.yaml 파일을 수정하자.

```bash
accessLog: {}

api:
  insecure: false
  dashboard: true

providers:
  docker:
    network: traefik
    exposedByDefault: false
  file:
    directory: /external_services
    watch: true

entryPoints:
  web:
    address: :80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: :443
    forwardedHeaders:
      trustedIPs:
        - 173.245.48.0/20
        - 103.21.244.0/22
        - 103.22.200.0/22
        - 103.31.4.0/22
        - 141.101.64.0/18
        - 108.162.192.0/18
        - 190.93.240.0/20
        - 188.114.96.0/20
        - 197.234.240.0/22
        - 198.41.128.0/17
        - 162.158.0.0/15
        - 104.16.0.0/13
        - 104.24.0.0/14
        - 172.64.0.0/13
        - 131.0.72.0/22
        - 2400:cb00::/32
        - 2606:4700::/32
        - 2803:f800::/32
        - 2405:b500::/32
        - 2405:8100::/32
        - 2a06:98c0::/29
        - 2c0f:f248::/32

certificatesResolvers:
  myresolver:
    acme:
      email: kali2005611@gmail.com
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web
```

읽어보면 이해가 될 것이다.

http to https 리다이렉트, 인증서 발급 관련 설정, 신뢰가능한 IP 대역 (클플) provider 설정이 있다.

따로 건들건 없고 아래의 myresolver에서 email를 본인의 이메일로 바꿔서 저장하자.

서버를 켜기 앞서 external로 명시해둔 traefik 네트워크를 생성한다.

```yaml
docker network create traefik
```

이제 이 상태에서 서버를 시작해보도록 하겠다.

```bash
docker compose up -d
```

서버가 실행되고 [ip.example.com](http://ip.example.com) 에 대한 인증서가 자동적으로 발급되고 얼마 뒤 https로 접속할 수 있게 된다.

또한 [traefik.example.com/dashboard/](http://traefik.example.com/dashboard/) 로 접속해 WEB UI도 이용할 수 있다.

## 추가 서비스 구성

여기까지 따라왔다면 traefik proxy 뒤에는 iplogger 서비스가 돌아가고 있을 것이다.

여기서 서비스를 추가하기 위해선 어떻게 해야될까?

현재 설정으로는 2가지 provider가 있다.

1. 도커를 이용한 자동 설정
2. file를 이용한 수동 설정

우선 도커를 이용한 방식은 labels을 이용해 동일 네트워크에 있는 컨테이너를 자동을 찾아 연결한다.

예시는 다음과 같다.

우선 kuma 폴더를 만들고 그 안에 docker-compose.yaml 파일을 만든다.

```yaml
version: "3.8"
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - uptime-kuma:/app/data
    labels:
      - traefik.enable=true
      - traefik.http.services.uptime-kuma.loadbalancer.server.port=3001
      - traefik.http.routers.uptime-kuma.entrypoints=websecure
      - traefik.http.routers.uptime-kuma.tls.certresolver=myresolver
      - traefik.http.routers.uptime-kuma.rule=Host(`uptime.minpeter.cf`)
    networks:
      - traefik

networks:
  traefik:
    external: true

volumes:
  uptime-kuma:
    driver: local
```

볼륨을 제외한다면 항상 동일한 패턴이다.

해당 서비스에 필요한 compose 파일을 작성하고 traefik를 위한 labels과 networks를 traefik 네트워크로 설정한다.

그럼 나머지는 traefik가 알아서 설정한다.

다음으로 file를 이용하는 방식은 다음과 같다.

먼저 같은 서버에서 서비스가 돌아가는 경우와 같은 네트워크의 다른서버에서 돌아가는 경우로 나뉘는데 후자의 경우엔 그냥 약간 응용하면 해결할 수 있다.

docker에서 host의 localhost에 접근하기 위해서 traefik/docker-compose.yaml에 다음 내용을 추가한다.

```yaml
traefik:
...

	extra_hosts:
	  - host.docker.internal:host-gateway
```

이렇게 하면 traefik 컨테이너 내부에서 host.docker.internal을 이용하여 호스트의 localhost에 접근 할 수 있다.

이후 앞에서 생성해두었던 external_services 폴더에 서비스를 정의하는 파일을 작성해준다.

traefik가 올라간 서버의 localhost의 10000 포트에 서비스가 올라가 있다고 가정하자.

```yaml
[http.routers]
  [http.routers.iplogger]
    entryPoints = ["websecure"]
    rule = "Host(`ip.minpeter.cf`)"
    service = "iplogger-ext-srv"
    [http.routers.iplogger.tls]
      certResolver = "myresolver"
[[http.services.iplogger-ext-srv.loadBalancer.servers]]
  url = "http://host.docker.internal:10000"
```

이렇게 파일을 만들면 파일 저장과 동시에 ip.example.com으로 들어오는 모든 트래픽은 [localhost](http://localhost):10000 으로 보내진다.

이제 기본적인 설정은 완료되었다.

traefik 서비스에 아래 2개 라벨를 추가하고 volume에도 한줄을 추가해주자.

```bash
# labels
- traefik.http.routers.traefik.middlewares=traefik-auth
- traefik.http.middlewares.traefik-auth.basicauth.usersfile=/usersfile

#volume
- ./usersfile:/usersfile
```

그리고 다음 명령어로 usersfile를 생성해주자.

```yaml
echo "<username>:<htpassword>" >> traefik/usersfile
```

위에서 <, >은 제거하고 htpassword는 온라인이나 htpasswd 명령어로 생성해주자.

그럼 접근할때 traefik가 인증 과정을 거치고 인증에 성공하면 서비스로 연결시켜준다.

와 샌즈 쓰다보니까 글 쓰기 싫어서 대충쓰고 있네

이제 그만쓸래
