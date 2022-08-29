---
title: "🦕 docker stack visualizer"
description: "visualizer는 2016 dockercon에서 stack기능 소개를 위해 만들어진 툴이다.이를 이용하면 swarm모드에서 노드와 컨테이너의 분포상태를 시각적으로 볼수 있다.  visualizer.yaml컨테이너를 이용해 실습하고 있다면 docker cp명령어로 파"
date: 2022-03-07T02:51:01.607Z
tags: ["docker","server","swarm","visualizer"]
---
## 개요
visualizer는 2016 dockercon에서 stack기능 소개를 위해 만들어진 툴이다.  
이를 이용하면 swarm모드에서 노드와 컨테이너의 분포상태를 시각적으로 볼수 있다.  

## 실습
**visualizer.yaml**
```yaml
version: "3"

services:
    app:
        image: dockersamples/visualizer
        ports:
            - "9000:8080"
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        deploy:
            mode: global
            placement:
                constraints: [node.role == manager]
```
컨테이너를 이용해 실습하고 있다면 docker cp명령어로 파일을 옮겨주자  
```
$ docker cp visualizer.yaml manager:/visualizer.yaml
```
서비스를 실행해주자
```
$ docker exec -it manager \
docker stack deploy -c /visualizer.yaml visualizer
```
localhost:9000에 접속해보면 아름다운 창이 뜰것이다.  

## 참고자료

[스웜(swarm)을 이용한 도커 컨테이너 배포_2](https://cornswrold.tistory.com/515?category=930033)  