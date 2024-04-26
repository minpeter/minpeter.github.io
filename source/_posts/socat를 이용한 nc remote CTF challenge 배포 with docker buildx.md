---
title: socat를 이용한 nc remote CTF challenge 배포
date: 2022-09-10 00:11:18
tags: ["socat", "nc", "netcat", "remote", "buildx"]
canonical_url: "https://minpeter.xyz/blog/use-socat-nc-ctf-challenge"
---

## 개요

각종 CTF와 드림핵등 문제를 풀다보면 이런식의 주소를 알려주는 경우가 많다.  
`nc pwnable.kr 9000`
\*pwnable.kr의 BOF 문제

netcat이라는 유틸리티로 해당 서버의 프로그램을 원격 실행시킬 수 있는데 이런 생각이 들었다.  
"이런 형태의 문제는 어떻게 만들고 배포할까?"

그래서 알아보았다.

## 그래서 어케하는데요?

**SOCAT를 사용하면 된다**

처음 알게된 방식은 socket 서버를 직접 개발하는거였다.  
이게 생각보다 괜찮았는데 딱 단순 메세지 출력용으로는 좋았다.  
근데 또 생각해보면 당장 위에 BOF 문제만 하더라도 입/출력을 다양하게 여러번 수행하는데 이를 일일히 소켓서버로 구현했을리가 없었다.  
심지어 nc 주소와 함께 제공되는 소스코드를 보면 더욱 확신이 들었다.  
**소켓관련 코드는 전혀 없었기 때문**이다.

그런데 GoogleCTF에서 정답이 보였다.  
문제 Dockerfile 중 socat이라는 프로그램을 CMD에 적어둔게 보였기 때문이다.  
이게 그동안 몰랐던 nc 서버관련 유틸이라는 직감이 들었다.

### Multipurpose relay (SOcket CAT)

```
두 개의 양방향 바이트 스트림을 설정하고 이들 간에 데이터를 전송하는 명령줄 기반 유틸리티입니다.
스트림은 다양한 유형의 데이터 싱크 및 소스(주소 유형 참조)로 구성될 수 있고 많은 주소 옵션이 스트림에 적용될 수 있기 때문에 socat은 다양한 목적으로 사용될 수 있습니다.
```

이라고 한다.  
그냥 netcat의 고급버전이라고 생각해도 될 것 같다.

## socat 설치

우분투에서는 다음과 같이 설치할 수 있다.  
`apt-get install socat`
맥에서는 brew가 설치되있다는 가정하에 다음과 같다.  
`brew install socat`

이번 글은 socat을 이용한 문제 배포이기 때문에 socat을 직접 사용하지도 깊게 다루지도 않겠다.

## socat 사용 챌린지 예제

다음은 socat를 사용하는 예제이다.

우선 일반적인 입출력을 수행하는 프로그램을 작성한다.

```py
# main.py
userInput = input("Enter a number: ")

try:
    userInput = int(userInput)
    if userInput == 1395:
        print("flag{you_entered_the_correct_number}")
    else:
        print("You did not enter the correct number")
except ValueError:
    print("You did not enter a number!")


```

위의 프로그램은 보다시피 1395를 입력할 경우 flag를 제공하는 간단한 프로그램이다.
만약 이 글을 보고 CTF 문제를 낼 생각이라면 해당 문제 파일로 대체하면 되겠다.

이제 배포를 위한 Dockerfile를 작성한다.

```Dockerfile
FROM python:3.10-alpine

RUN apk add socat

WORKDIR /app

ENV PORT 1337
ENV FILE_NAEM main.py

COPY $FILE_NAEM .


CMD ["socat" , "-T60" , "-dd" , "-v" , "-v" , "TCP-LISTEN:"+$PORT+",reuseaddr,fork" , "EXEC:python3 "+$FILE_NAEM+",pty,stderr,setsid,sigint,sane"]
```

이제 도커 이미지를 빌드한다.

`docker build -t socattest .`

하고 문제 없이 빌드되면 실행한다.

`docker run --rm -p 1337:1337 socattest`

이제 새로운 터미널 창을 실행해서 다음 명령어로 서버에 접속한다.

`nc localhost 1337`

그러면 다음과 같은 화면이 나온다.

```
Enter a number:
```

이제 1395를 입력하면 다음과 같은 화면이 나온다.

```
Enter a number: 1395
flag{you_entered_the_correct_number}
```

이제 이 문제를 CTF에 올려보자.

## CTF에 문제 올리기

CTF에 문제를 올리기 위해서는 다음과 같은 파일들이 필요하다.

- Dockerfile
- 문제 파일
- 문제 설명 파일

이제 문제를 올리기 위한 폴더를 만들고 다음과 같이 파일들을 작성한다.

```
.
├── Dockerfile
├── README.md
└── main.py
```

Dockerfile은 위에서 작성한 것과 같다.

README.md는 다음과 같이 작성한다.

```
# socat challenge

## Description

This is a socat challenge.

## How to solve

1. Connect to the server.
2. Enter the correct number.
```

main.py는 위에서 작성한 것과 같다.

이제 이 폴더를 압축해서 CTF에 올리면 된다.

이 글을 보는 누구든 CTF 문제를 만들때 도움이 되었으면 좋겠다.  
여기까지 글을 읽어주신 모든 분들께 감사드린다.

## 참고

위에 CTF에 문제를 올리는 방법은 Github Copilot이 전부 작성했다.  
솔직히 맥락이 너무 잘 맞고 문장이 너무 자연스러워서 놀랐다.  
AI 만세?
