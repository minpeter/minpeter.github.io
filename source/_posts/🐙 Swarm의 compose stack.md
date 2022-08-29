---
title: "🐙 Swarm의 compose stack"
description: "전에서 run명령어와 비슷한 service 명령어를 이용해 swarm에 컨테이너를 띄우고 스케일을 조정해보았다.이번엔 compose명령어와 비슷한 stack에 대해 알아보자.스택이란 하나 이상의 서비스를 그룹으로 묶은 단위로, 애플리케이션 저체 구성을 정의한다.그냥 간"
date: 2022-03-07T02:50:11.702Z
tags: ["docker","docker-swarn","server"]
---

## swarm에서 compose
전에서 run명령어와 비슷한 service 명령어를 이용해 swarm에 컨테이너를 띄우고 스케일을 조정해보았다.  
이번엔 compose명령어와 비슷한 stack에 대해 알아보자.  
스택이란 하나 이상의 서비스를 그룹으로 묶은 단위로, 애플리케이션 저체 구성을 정의한다.  
그냥 간단하게 swarm에서 작동하는 (스케일기능이 포함된) compose라고 생각하자  
또 특징으로는 compose와 동일하게 같은 네트워크에 포함되게된다.  

일단 네트워크을 하나 생성해주자
```
$ docker exec -it manager \
docker network create --driver=overlay --attachable test
```
test라는 이름의 네트워크를 생성해주었다.  
이제 stack-compose 파일을 생성해주자  
**docker-stack.yaml**
```yaml
version: '3'
services:
    flask-echo:
        image: minpeter/flask-echo
        deploy:
            replicas: 3
            placement:
                constraints: [node.role != manager]
        ports:
            - "5000:5000"
        networks:
            - test
    nginx-echo:
        image: minpeter/nginx-echo
        deploy:
            replicas: 3
            placement:
                constraints: [node.role != manager]
        depends_on:
            - flask-echo
        ports:
            - "8080:80"
        networks:
            - test

networks:
    test:
        external: true
```
이제 파일을 manager 컨테이너로 보내야되는데 docker cp 명령어를 사용하자  
```
$ docker cp docker-stack.yaml manager:/docker-stack.yaml
```
이제 stack을 배포할 차례이다.  
```
$ docker exec -it manager \
docker stack deploy -c /docker-stack.yaml echo
```
이제 `hello, flask!`를 출력하는 간단한 페이지가 8080포트에 올라갔다.  
배포된 서비스를 확인해보자  
```
$ docker exec -it manager \
docker stack services echo
```
스택이 컨테이너를 배포한 방식을 확인하려면 다음 명령어를 사용하자.  
```
$ docker exec -it manager \
docker stack ps echo
```


## 참고자료

[스웜(swarm)을 이용한 도커 컨테이너 배포_2](https://cornswrold.tistory.com/515?category=930033)  