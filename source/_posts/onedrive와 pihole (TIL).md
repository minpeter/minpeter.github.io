---
title: "onedrive와 pihole (TIL)"
date: 2022-09-25 11:01:00 +0900
tags: ["DNS", "TIL", "homelab", "onedirve", "pihole", "server"]
canonical_url: "https://minpeter.xyz/blog/onedrive-pihole"
---

### update log

| 날짜       | 변경점               |
| ---------- | -------------------- |
| 2021/09/12 | 🎺 처음 글 작성한 날 |
| 2022/09/25 | 현재 상황 업데이트   |

## 사건의 발단

별것도 아닌걸로 2일을 날렸다.  
windows 11 DEV 채널이 BETA 채널과 완전히 분리되어 다른 빌드번호를 가지게 되었다.  
뭔 뜻이냐면 이제 더 이상 DEV에서 BETA로 넘어갈 수 없다는 뜻이고, 더이상 DEV에서는 실사용으로 사용하기 어려울꺼라는 뜻이다.  
또 마침 22454 빌드로 넘어가고 전 빌드에 존재했던 sandbox 작동 불능은 고쳐지지도 않고 taskbar에 아이콘이 왼쪽으로 치우쳐지는 이상한 버그가 생겼다.  
솔직히 이렇게 버그가 많은데 DEV 채널 플라이트를 계속할 이유도 없었고,  
따라서 BETA채널로 내려가기를 결정했다.

## onedrive와 pihole을 궁합

BETA채널로 내려가기 위해선 전부 재설치를 해야했다.  
최근 winget 패키지관리자로 프로그램 설치도 간단해졌고 원도우 설치도 얼마 안걸리는 마당에 그냥 밀기로했다.  
설치 usb만들고 BETA설치를 한뒤 onedrive 로컬폴더 백업을 할때 문제가 생겼다.  
옵션이 없는 것이다.  
혹시 윈도우설치가 잘못됬나싶어서 재설치도 해보고, 다시 DEV로 올라가서 설정도 해봤다.  
전부 실패였다.  
뭐가 문제였을까?  
생각보다 간단했다.  
[about onedrive](https://support.microsoft.com/ko-kr/office/onedrive%eb%a6%b4%eb%a6%ac%ec%8a%a4-%ec%a0%95%eb%b3%b4-%ec%b0%b8%ec%a1%b0-845dcf18-f921-435e-bf28-4e24b95e5fc0?ui=ko-kr&rs=ko-kr&ad=kr)
에 들어가보면 아래와 같이 나와있다.

```
참고 사항:

표의 빈칸은 현재 해당 링에 배포 중인 빌드가 없음을 의미합니다.

OneDrive 동기화 앱 업데이트를 적용하려면 컴퓨터가 다음에 연결될 수 있어야 합니다: "oneclient.sfx.ms" 및 "g.live.com." 이들 도메인을 차단하고 있지 않은지 확인합니다. 이들은 또한 기능을 활성화/비활성홯하고 버그 수정을 적용하는데 사용됩니다. Microsoft 365에서 사용되는 URL 및 IP 주소에 대한 자세한 정보

OneDrive 동기화 앱 업데이트 프로세스에 대해 자세히 알아보십시오.

지연된 링 릴리스를 완료한 후 빌드가 프로덕션 링에 릴리스되기를 기다린 후 지연된 링의 다음 릴리스로 선택합니다. 이 경우 정확한 빌드 번호와 대상 날짜를 게시하기 전에 지연된 열을 "다음 릴리스: 19.222.x"을 업데이트하여 고객의 계획에 도움을 제공합니다.
```

2번째 줄을 유심히 보자  
"oneclient.sfx.ms"와 "g.live.com"에 연결할 수 있어야한다.  
근데 ping test를 해보니 "g.live.com"에 접속이 되지 않았다.  
정확히는 아예 ip주소를 못찾았으니 pihole이 막고 있는 것이다.  
DNS서버를 기본상태로 돌리고 다시 백업 설정을 해보니 거짓말같이 잘 작동하였다.  
아놔 ㅅㅂ....  
솔직히 마소 잘못은 아니다 ㅋㅋ g.live.com 이 광고이미지서버, msc뉴스서버, 원드라이브의 일부 기능까지 너무 광범위하게 사용될뿐이지...  
pihole에 화이트리스트를 주고 windows 11 beta 체널로 이주를 완료했다.

글에 언급된거 말고도 별 뻘짓을 다해서 2일이나 날려먹었다.  
~~내소중한주말..~~

## 2022/09/25 업데이트

위 사건 이후 거의 2주 정도 pihole를 사용하지 않고 클라우드플레어의 DNS를 사용했었다.  
하지만 어느 시점에 pihole의 기본 Adlist가 업데이트 되었고 별도의 화이트리스트 없이 다음과 같이 정상적으로 쿼리가 가능해지게 되었다.

```paintext
C:\Users\minpeter>nslookup
Default Server:  pi.hole
Address:  192.168.0.120

> oneclient.sfx.ms
Server:  pi.hole
Address:  192.168.0.120

Non-authoritative answer:
Name:    e9659.dspg.akamaiedge.net
Addresses:  2600:1410:1000:185::25bb
          2600:1410:1000:18d::25bb
          104.74.21.118
Aliases:  oneclient.sfx.ms
          oneclient.sfx.ms.edgekey.net

> g.live.com
Server:  pi.hole
Address:  192.168.0.120

Non-authoritative answer:
Name:    g-msn-com-nsatc.trafficmanager.net
Address:  52.231.199.126
Aliases:  g.live.com
          g.msn.com
```

따라서 pihole를 dns 서버로 사용해도 원드라이브 백업오류는 발생하지 않는다.  
다행인 것 같다.
