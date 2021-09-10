---
title: 🐋 Docker swarm Orchestration
author: minpeter
date: 2021-09-09 11:12:00 +0900
categories: [server]
tags: [docker, docker-swarm, server]
pin: no
---

# 소개
해당 포스트는 도커안에 도커 (dind)기능을 이용해 로컬에서 클러스터와 유사한 환경을 구성하고,  
docker swarm을 이용해 클러스터를 제어하는 Orchestration을 실습하는 방법에 대한 포스팅이다.  

## docker swarm, Orchestration이란?
docker swarm은 kubernetes와 같은 Orchestration tool이다.  
예를 들어 5대의 서버가 있다고 생각해보자  
Orchestration tool을 사용하지 않을 경우는 각각 서버에 동일한 기능을 하도록 설정하거나, 각자 특정기능을 담당하는 식으로 설정해야될 것이다.  
하지만 이 경우 특정 기능이나 서비스에 많은 요청이 들어올경우, 직접 수동으로 해당 서비스를 더 많이 만들어주어야 한다.  
하지만 Orchestration tool을 이용하면 설정값에 따라 특정 서비스를 생성하고 유지하기 때문에 부하가 많이 걸릴시 설정만 바꿔주면 되고 해당 작업 또한 자동화 할 수 있게된다.  
또 서비스를 업데이트 할때 서비스를 종료하지않고 배포하는 것이 가능하다.  
(특정 서버에 서비스를 몰아서 배치하는 것이 아닌 분산하여 배치하기 때문에 순차적으로 배포를 진행하면 중단하지 않고도 가능하다.)  
배포를 하였을때 해당 배포에 문제가 생긴 경우 배포를 중단하고 원래 배포전 상태로 돌려주고, 서비스에 문제가 생겨 종료된 경우 자동으로 재시작도 진행해준다.  
이처럼 서비스가 배포중에도 중단되지않고 꺼지지 않아야 하는 경우 사용하면 좋은 도구이다.  

## docker swarm에서 사용하는 용어 설명
|이름|역활|
|:---:|:---:|
|compose|여러 컨테이너로 구성된 도커 애플리케션을 관리함|
|swarm|클러스터 구축 및 관리|
|service|기본적인 배포단위, swarm에서 클러스터 안의 서비스을 관리, 기본 docker에서 run과 비슷하다|
|stack|swarm에서 여러 개의 서비스를 합함 전체 애플리케이션을 관리|
|node|swarm 클러스터에 속한 도커 서버의 단위|
|manager node|스윔 클러스터의 상태를 관리하는 노드, 동시에 worker node가 될 수 있고, 여기에서만 swarm명령어를 실행할수 있다.|
|worker node|manager node의 명령을 받아 컨테이너를 생성, 상태를 체크하는 노드|
|task|컨테이너 배포 단위, 서비스는 여러개의 task을 실행, task는 컨테이너를 관리한다|

# docker swarm 실습
docker swarm은 기본적으로 여러개의 서버를 관리하는 툴이다.  
하지만 우리가 실습할때는 대부분 노트북 하나만을 가지고 실습을 할 것이다.  
따라서 실제 환경과는 조금 다른 가상 실습환경을 만들것이다.  
다양한 방법이 있지만 dind (docker in docker)기능을 활용해서 도커 컨테이너를 각각 노드로 사용하여 실습하도록하겠다.  
(다른 방법으로는 vagrant을 이용해 가상머신을 생성해주거나 클라우드서비스를 이용해 실제 클러스터를 구현할수도)  
docker compose을 이용해서 1개의 manager node(dind)와 3개의 worker node(dind)을 생성해줄것이다.  
참고한 기존 블로그 글에서는 registy컨테어너가 별도로 존재했지만 실습결과 딱히 필요성을 느끼지 못해 삭제하였다.  
(docker registy는 docker hub를 대체하는 로컬서버정도로 이해하면 될 것 같다.)

## 가상클러스터 생성
docker-swarm 디렉토리를 생성하고 그 안에 아래의 코드를 참조해 파일을 만들어주자.  
**docker-compose.yaml**
```yaml
version: "3"
services:
    manager:
        container_name: manager
        image: docker:dind
        privileged: true
        tty: true
        ports:
            - 8000:80
            - 9000:9000
        depends_on:
            - registry
        expose:
            - 3375
        volumes:
            - "./stack:/stack"
    worker01:
        container_name: worker01
        image: docker:dind
        privileged: true
        tty: true
        depends_on:
            - manager
            - registry
        expose:
            - 7946
            - 7946/udp
            - 4789/udp
    worker02:
        container_name: worker02
        image: docker:dind
        privileged: true
        tty: true
        depends_on:
            - manager
            - registry
        expose:
            - 7946
            - 7946/udp
            - 4789/udp
    worker03:
        container_name: worker03
        image: docker:dind
        privileged: true
        tty: true
        depends_on:
            - manager
            - registry
        expose:
            - 7946
            - 7946/udp
            - 4789/udp
```

<details>
<summary>registry을 이용한 예제</summary>
<div markdown="1">

```yaml
version: "3"
services:
    registry:
        container_name: registry
        image: registry:latest
        ports:
            - 5000:5000
        volumes:
            - "./registry-data:/var/lib/registry"
    manager:
        container_name: manager
        image: docker:dind
        privileged: true
        tty: true
        ports:
            - 8000:80
            - 9000:9000
        depends_on:
            - registry
        expose:
            - 3375
        command: "--insecure-registry registry:5000"
        volumes:
            - "./stack:/stack"
    worker01:
        container_name: worker01
        image: docker:dind
        privileged: true
        tty: true
        depends_on:
            - manager
            - registry
        expose:
            - 7946
            - 7946/udp
            - 4789/udp
        command: "--insecure-registry registry:5000"
    worker02:
        container_name: worker02
        image: docker:dind
        privileged: true
        tty: true
        depends_on:
            - manager
            - registry
        expose:
            - 7946
            - 7946/udp
            - 4789/udp
        command: "--insecure-registry registry:5000"
    worker03:
        container_name: worker03
        image: docker:dind
        privileged: true
        tty: true
        depends_on:
            - manager
            - registry
        expose:
            - 7946
            - 7946/udp
            - 4789/udp
        command: "--insecure-registry registry:5000"
```
</div>
</details></br>

다음 아래의 명령어를 입력해주자  
```
$ docker compose up -d
$ docker ps
```
이제 클러스터의 역활을 할 컨테이너가 4개 또는 5개가 생성된 것을 확인 할 수 있을 것이다.  

## manager 등록
원래 `docker swarm init` 명령어를 이용해 등록하면 되지만 현제 컨테이너 안에 가상클러스터를 실행했기 때문에 명령어 앞에 다음 명령어를 추가로 입력해주어야한다.  
```
$ docker exec -it (실행할 타겟, manager or worker) \
(실행할 명령어)
```
따라서 메니저에서 docker swarm init 명령어를 실행하기 위해서는 다음 명령어를 실행하면 된다.  
```
$ docker exec -it manager \
docker swarm init
```
명령어를 입력하면 아래에  
```
$ docker swarm join --token (토큰값) (메니저IP):(port)
```
형태로 출력될텐데 다음 명령어를 복사해주자.  

## worker 등록
위에서 복사한 명령어를 worker에서 실행하면 우리가 만든 swarm 클러스터에 등록할 수 있다.  
다음과 같은 방식으로 worker01 ~ worker 03까지 등록을 진행해주자.  
```
$ docker exec -it worker01 \
docker swarm join --token (토큰값) (메니저IP):(port)
```
```
$ docker exec -it worker02 \
docker swarm join --token (토큰값) (메니저IP):(port)
```
```
$ docker exec -it worker03 \
docker swarm join --token (토큰값) (메니저IP):(port)
```
이제 아래 명령어로 노드가 잘 등록됬는지 확인해보자
```
$ docker exec -it manager \
docker node ls
```
만약 worker node을 포함해 4개의 node가 나온다면 클러스터를 만드는데 성공한 것이다.  

<details>
<summary>docker registry 이미지 등록</summary>
<div markdown="1">

## 이미지 태그 생성
```
$ docker tag (원본이미지) localhost:5000/(원본이미지)
```
## registry 컨테이너에 이미지 등록
```
$ docker push localhost:5000/(원본이미지)
```
## worker 컨테이너에서 이미지 다운
```
$ docker exec -it worker01 \
docker pull registry:5000/(원본이미지)
```
## 이미지확인
```
$ docker exec -it worker01 \
docker images
```
</div>
</details></br>

## 서비스 생성


## 참고자료

[스웜(swarm)을 이용한 도커 컨테이너 배포_1](https://cornswrold.tistory.com/512?category=930033)  
[스웜(swarm)을 이용한 도커 컨테이너 배포_2](https://cornswrold.tistory.com/515?category=930033)  
[스웜(swarm)을 이용한 도커 컨테이너 배포_3 (스웜 클러스터 외부에서 서비스 사용하기)](https://cornswrold.tistory.com/516?category=930033)  
[Docker Swarm을 이용한 쉽고 빠른 분산 서버 관리](https://subicura.com/2017/02/25/container-orchestration-with-docker-swarm.html)  
[docker docs](https://docs.docker.com/engine/swarm)  
