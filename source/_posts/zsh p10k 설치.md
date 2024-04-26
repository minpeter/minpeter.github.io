---
title: "zsh p10k 설치"
description: "요즘 뜨는 zsh과 그를 꾸며주는 oh-my-zsh, powerlevel10k 설정 가이드"
date: 2021-08-29 05:05:05 +0900
tags: ["linux", "ohmyzsh", "p10k", "shell", "zsh"]
canonical_url: "https://minpeter.xyz/blog/zsh-p10k-install"
---

## zsh과 oh-my-zsh, p10k의 사이

zsh은 bash같은 친구를 대체하는 심미적인 부분을 더해주는 쉘이다.  
~~(적어도 내가 느끼기에는 그랬다)~~  
여기서 oh-my-zsh이 나오는데 zsh에 theme와 plugin을 설정할 수 있게 해준다.  
그리고 oh-my-zsh의 테마로 등장하는 친구가 p10k이다.  
장점으로는 ~~멋있고!~~ 커스터마이징이 가능하다!!!  
공식 글꼴을 깔면 배포판 로고나 깃관련된 이모지나 폴더 아이콘이 터미널에 표시되는등 신기한 기능도 있다.

## zsh 설치하기

`sudo apt-get install zsh`  
대부분 다른 배포판도 패키지 관리자만 변경하면 동작한다.

예를 들면 arch에서는  
`sudo pacman -S zsh`

fedora에서는  
`sudo dnf install zsh`

## oh-my-zsh 설치하기

`sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`

그렇다 모든 베포판이 동일할 것이라고 예상이 된다.  
중간에 zsh을 기본쉘로 변경한다고 패스워드를 입력하라고 하는데.. 당연히 입력해주자

## power level 10k (p10k) 설치

`git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k`

이 친구도 oh-my-zsh만 설치되있다면 정상적으로 작동한다.

## p10k theme 적용

`vim ~/.zshrc`  
다른 편집기가 편하면 그거 쓰면 된다.

`ZSH_THEME="robbyrussell"` -> `ZSH_THEME="powerlevel10k/powerlevel10k"`
로 수정해주면 된다

## font 설치

여기서 폰트를 선택해야된다.  
기본적으로 powerline 폰트만 포함된 고정폭폰트면 뭐든 상관없으나 `D2coding`, `MesloLGS NF`, `Fira Code` 중에서 선택하는 걸 추천한다.

참고로 `MesloLGS NF`가 p10k 공식 폰트며 해당 폰트 설치시 아이콘을 표시하는 옵션이 생성된다.

이부분은 운영체제, 설치환경 등에 따라 다르니 각자 폰트 설치법, 터미널 폰트 설정법을 찾아 설정하기 바란다

## p10k setting

`p10k configure` 명령어로 p10k 테마 커스터마이징을 할 수 있다.

잘 읽고 선호 하는 설정으로 설정해주면 자신만의 p10k theme가 완성된다.

## 참고자료

[abdfnx/oh-my-zsh-powerlevel10k-cool-terminal-1no0](https://dev.to/abdfnx/oh-my-zsh-powerlevel10k-cool-terminal-1no0)  
[ruddms936/zsh-설치](https://velog.io/@ruddms936/zsh-%EC%84%A4%EC%B9%98)  
[FiraCode](https://github.com/tonsky/FiraCode)  
[D2coding](https://github.com/naver/d2codingfont)  
[MesloLGS NF](https://github.com/romkatv/powerlevel10k/#user-content-fonts)
