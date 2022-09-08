---
title: 🎥 digispark attiny85 usb board firmware upgrade!!
date: 2022-08-31 21:11:34
tags: ["attiny", "digispark"]
---

## 사전 준비물
1. 컴퓨터로 스케치 업로드가 가능한 모든 아두이노 (아마?)
2. 약간의 점퍼케이블 (정확히는 6개)
3. 아래 사진에 나온 Digispark USB Development Board

*글에서 ISP는 In-System Programming를 의미합니다.

## attiny가 뭔데?
Atmel사에서 출시된 마이크로컨트롤러이다.  
AVR microcontroller 중에서 아두이노에서 사용하는 ATmega 칩보다 전반적으로 사양이 낮은 것이 특징이며, 그에 걸맞게 대체로 크기가 작은 것 또한 특징이다.  
이번 글에서 사용할 attiny 칩은 attiny85 칩셋이며 편의를 위해 digispark사에서 설계한 보드를 기반으로 나온 중국산 클론 보드를 이용할 것이다.  
해당 보드는 pcb를 usb A type으로 설계해 컴퓨터에 바로 연결할 수 있다는 특징이 있다.  
![](/images/61e2f14edffc1edfa2685963155b0d33.jpg)

## 이 포스팅에서 수행할 작업
digispark사의 정품, 중국산 copy 둘 다 펌웨어 버전이 제각각이다.  
나중에 설명할 펌웨어 최신버전은 2.6 이지만 정품의 경우 1.02, 중국산의 경우 아예 펌웨어가 없는 경우도 존재한다.  
따라서 정품, 카피 구분없이 펌웨어을 2.6 버전으로 업그레이드 할 것이다.
1. arudion를 ISP로 설정
2. ISP를 이용해 Micronucleus V2.6를 보드에 업로드
4. digispark USB driver Install
3. 예제 프로젝트를 보드에 업로드하여 정상 작동 확인


## 사용할 펌웨어 Micronucleus에 대하여
우선 사용할 저 신기한 보드에 대해 조금 더 알아야한다.  
usb 포트가 존재하지만 usb를 위한 별도의 하드웨어가 없다.  
엥? 그럼 usb는 왜 있을까?  
attiny85 칩셋이 v-usb를 지원... 아니 atmel사의 거의 모든 AVR이 v-usb 기능을 지원하기 떄문에 존재한다.  
무려 디지털포트를 이용해 usb를 소프트웨어적으로 지원한다.  
이 v-usb를 이용해 프로젝트를 ISP 없이 업로드 할 수 있는데, 이를 지원해주는 펌웨어가 Micronucleus이다.  

간단하게 설명하면 usb를 프로그래밍 할 수 있는 방식이 2가지인데,  
1. ISP를 이용하여 보드 메모리에 직접 프로그래밍하는 방식
2. Micronucleus를 부트로더로 업로드하여 보드에 v-usb 지원을 추가하고 usb로 업로드하는 방식

각각 장점과 단점이 있다.  
1번의 경우 ISP를 연결하는 수고를 해야되는 반면 attiny85 프로그래밍 메모리를 전부 이용할 수 있다.  
2번의 경우는 ISP를 연결하는 대신 usb로 간단하게 프로그래밍이 가능해지는 반면 일부 프로그래밍 메모리를 펌웨어인 Micronucleus에게 내줘야한다.  
이렇게하는 경우 편리하지만 큰 프로그램을 업로드하는 경우 제약이 존재한다.  

암튼 펌웨어를 올리기로 결정한 이상 이 포스터는 2번을 수행하는 방법에 대해 설명하겠다. 

아마도 나중에 "attiny85 without microncleus" 같은 글을 쓸 수도 있지 않을까?  

## 아두이노를 ISP로 만들기
사실 이 단계는 AVR ISP mkⅡ 같은 물건이 있으면 필요없다.  
하지만 나는 학생이고 못해도 2만원선인 ISP를 구매하기 어렵ㄷ~~기보단 아직 살 명분을 못 찾은것 뿐~~ㅏ.  
따라서 누구나 집에 하나쯤은 있는 (?) 아두이노를 이용해 임시 ISP로 이용해보자.  
(사실 집에 없어도 왠만한 중고등학교 어딘가엔 굴러다니는게 아두이노 아닌가?)  

1. 컴퓨터에 아두이노를 연결한다.
2. arduino IDE를 열고 `File > Examples > 11.ArduinoISP > ArduinoISP`를 선택한다.
3. 아두이노 보드에 샘플 스케치를 업로드한다.

이제 아두이노는 마치 ISP처럼 작동된다.  

## arduino와 attiny85 간의 와이어링
| ATtiny85 pin | Arduino Pin | Role |
|:----:|:----:|:----:|
| 5V | 5V | VCC |
| GND | GND | GND |
| P5 | 10 | Reset |
| P0 | 11 | MOSI |
| P1 | 12 | MISO |
| P2 | 13 | SCK |

미리 준비해둔 점퍼 케이블로 잘 연결해주면 된다.  

## 필수 파일 다운로드
https://github.com/micronucleus/micronucleus의 `firmware/releases/t85_default.hex` 파일이 필요하다.  
다음 [링크](https://github.com/micronucleus/micronucleus/blob/master/firmware/releases/t85_default.hex)에서 다운 받을 수 있다.  

## avrdude로 firmware 굽기
```powershell
"C:\Program Files (x86)\Arduino\hardware\tools\avr/bin/avrdude" ^
-C "C:\Program Files (x86)\Arduino\hardware\tools\avr/etc/avrdude.conf" ^
-v -pattiny85 -cstk500v1 -PCOM6 -b19200 ^
-Uflash:w:"%USERPROFILE%\Downloads\micronucleus\firmware\releases\t85_default.hex":i ^
-U lfuse:w:0xe1:m -U hfuse:w:0xdd:m -U efuse:w:0xfe:m
```
*cmd 명령어 줄바꿈은 ^(caret)를 이용한다. 여담으로 pwsh은 `(backtick)

arduino IDE 설치할때 같이 다운되는 avrdude를 사용하는데 위의 명령어에서 `"%USERPROFILE%\Downloads\micronucleus\firmware\releases\t85_default.hex"` 부분은 다운받은 `t85_default.hex` 파일이 위치한 경로로 대체하여 주자  
또한 `-PCOM6` 옵션은 ISP로 설정한 아두이노의 포트 숫자로 바꿔주면 된다.  
나의 경우 COM6가 아두이노이기 때문에 -PCOM6로 설정했다.  

이제 전부 끝났다.  
약간 수정한 명령어를 **CMD**에서 실행하면 firmware 업로드가 진행되고 다음과 같은 글씨가 나타나면 성공이다.  
```paintext
.
.
avrdude: input file 0xfe contains 1 bytes
avrdude: reading on-chip efuse data:

Reading | ################################################## | 100% 0.00s

avrdude: verifying ...
avrdude: 1 bytes of efuse verified

avrdude: safemode: lfuse reads as E1
avrdude: safemode: hfuse reads as DD
avrdude: safemode: efuse reads as FE
avrdude: safemode: Fuses OK (E:FE, H:DD, L:E1)

avrdude done.  Thank you.
```

여기까지 성공했다면 2.6 버전 설치에 성공한 것이다.  
또는 위에서 펌웨어 파일만 수정해 다양한 옵션이 활성화된 https://github.com/ArminJo/micronucleus-firmware의 스피드로더 펌웨어를 시도할 수도 있고, digispark에서 공식지원하는 마지막 버전인 micronucleus 1.02로 다운그레이드 할 수도 있다.  


## digispark USB driver install

1. https://github.com/digistump/DigistumpArduino/releases에서 `Digistump.Driver.zip` 다운로드
2. 압축해제
3. `Install Drivers.exe` 실행 및 설치

이제 드라이버는 설치됬다.  
arduino IDE에 약간의 설정을 하면 코드를 v-usb을 이용해 업로드할 수 있게된다.  

1. arduino IDE 실행
2. File > Preferences 
3. Additional Boards Manager URLs에 `http://digistump.com/package_digistump_index.json` 추가
4. OK 클릭
5. Tools > Board > Boards Manager...
6. digistump 검색 후 `Digistump AVR Boards` 설치

설정은 끝이다.  
태스트 코드를 업로드하여 보자.  

## Keyboard로 "HelloWorld" 적는 프로그램 업로드
스케치를 하나 만들고 다음 코드를 입력하자.

```c
#include "DigiKeyboard.h"

void setup() {
  pinMode(1, OUTPUT);
}

void loop() {
   
  DigiKeyboard.update();
  DigiKeyboard.sendKeyStroke(0); //구형장치 호환성 유지용
  DigiKeyboard.delay(500);
  DigiKeyboard.println("HelloWorld");
  DigiKeyboard.delay(1000);
  DigiKeyboard.println("HelloWorld");

  for(int i=0; i<=10; i++) {
    digitalWrite(1, HIGH);
    DigiKeyboard.delay(100);
    digitalWrite(1, LOW); 
    DigiKeyboard.delay(100);
  }
}
```

HelloWorld를 두번 입력하고 led를 10번 깜빡이는 코드다.
업로드를 진행해보자.

1. Tools > Board > Digistump AVR Boards > Digispark (Default - 16.5mhz)
2. Ctrl + U // 코드 업로드 단축키

다음과 같은 글자가 출력되면 USB 포트에 attiny85 board를 연결해주자

```paintext
Sketch uses 3194 bytes (53%) of program storage space. Maximum is 6012 bytes.
Global variables use 98 bytes of dynamic memory.
Running Digispark Uploader...
Plug in device now... (will timeout in 60 seconds)
```

연결 후 다음과 같이 출력되면 성공적으로 업로드된 것이다.

```paintext
> Device is found!
connecting: 16% complete
connecting: 22% complete
connecting: 28% complete
connecting: 33% complete
> Device has firmware version 2.6
> Device signature: 0x1e930b 
> Available space for user applications: 6650 bytes
> Suggested sleep time between sending pages: 7ms
> Whole page count: 104  page size: 64
> Erase function sleep duration: 728ms
parsing: 50% complete
> Erasing the memory ...
erasing: 55% complete
erasing: 60% complete
erasing: 65% complete
> Starting to upload ...
writing: 70% complete
writing: 75% complete
writing: 80% complete
> Starting the user app ...
running: 100% complete
>> Micronucleus done. Thank you!
```

업로드 직후 보드가 재연결되고 6초동안 코드업로드를 대기한다.  
그 이후 다시 재연결되고 HID 모드로 동작한다.  
마치 사람이 직업 입력하는 것으로 받아드리고 커서의 위치에 HelloWorld를 두번 적을 것이다.  
그 후 보드에 led가 10번 깜빡거린다면 이 포스트의 모든 과정이 성공적이라는 의미이다.  
또한 코드를 업로드할때 출력중 다음과 같은 부분이 있다.  

```paintext
> Device is found!
connecting: 16% complete
connecting: 22% complete
connecting: 28% complete
connecting: 33% complete
> Device has firmware version 2.6
```

직관적으로 firware version 2.6!! 이라고 알려준다.  
다음 attiny 포스트는 연결즉시 프로그램이 실행되는 스피드로더를 이용해 usb rubber ducky를 만들어 볼까 한다.  