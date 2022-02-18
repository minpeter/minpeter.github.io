---
title: "🔮 WSL 개발 환경 설정 (with asdf)"
date: 2022-02-18T21:09:48+09:00
categories: [terminal settings]
tags: [terminal, logo-ls, bat, asdf, linux]
author: minpeter
pin: no
--- 

windows에서 개발환경을 구축하는 경우 몇가지 불편한 점이 있었다.

존재하긴 하지만 아직은 부족한 패키지 관리자 (winget)

기본을 무시해 버리는 비 POSIX 터미널

따라오는 수많은 cli기반 툴들의 미지원

한번 꼬이면 어디부터 해결해야될지 모르겠는 PATH 

사실 몇가지는 windows에 대해 몰라서 생기는 불편함과 문제점일 수도 있다.

그런데 마침 마소에서 WSL이라는 거희 완벽한 시스템을 제공하니 당연히 써야겠지

아무튼 WSL를 잘 설치해보자

보통 `wsl --install` 이나 windows store에서 다운 받을 수 있다.

이후 ubuntu 앱을 실행시키면 계정설정을 도와준다.

이후 설명을 따라하자

## 소프트웨어 업데이트

```bash
sudo apt update && sudo apt upgrade
```

## vscode  remote 설정

```bash
sudo apt-get install wget ca-certificates
```

vscode에서 ****[Remote - WSL](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)**** 플러그인 설치

터미널에서 `code .` 입력

## Git 설정

```bash
sudo apt-get install git
git config --global user.name "Your Name"
git config --global user.email "youremail@domain.com"
git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/libexec/git-core/git-credential-manager-core.exe"
```

[windows의 crlf와 wsl의 lf의 충돌로 인한 문제 해결](https://code.visualstudio.com/docs/remote/troubleshooting#_resolving-git-line-ending-issues-in-containers-resulting-in-many-modified-files)

## docker 설치

```bash
// cmd or pwsh

winget install docker.desktop
```

docker desktop >> settings

General >> Use the WSL 2 based engine

Resources >> WSL INTEGRATION >> (turn on)

## zsh, p10k theme 설치

```bash
sudo apt install zsh
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
```

p10k 태마의 아름다움을 위해 [Nerd Fonts](https://github.com/ryanoasis/nerd-fonts), [Source Code Pro](https://github.com/adobe-fonts/source-code-pro), [Font Awesome](https://fontawesome.com/), [Powerline](https://github.com/powerline/fonts) 폰트 중 하나를 선택해 windows에 설치 후 터미널의 폰트를 변경해주자

** Nerd Fonts를 이용할 때만  p10k 테마의 모든 설정을 활성화할 수 있다.

(Meslo Nerd Font 추천)

~/.zshrc  파일의

`ZSH_THEME="robbyrussell"`
 -> `ZSH_THEME="powerlevel10k/powerlevel10k"`
 로 수정

`source ~/.zshrc`

로 p10k 테마를 적용시킨다.

설정마법사 같은 창이 나오는데 대충 원하는 테마를 설정해주자.

만약 다시 설정하고 싶다면 `p10k configure` 명령어를 이용해 가능하다.

## oh-my-zsh 플러그인

```bash
# zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

~/.zshrc 파일에 plugins 항목에 다음과 같이 추가한다.

```bash
plugins=(
  zsh-syntax-highlighting
  zsh-autosuggestions
)
```

추가로 추천하는 플러그인은 git, asdf, docker 정도

`source .zshrc`

## 터미널 입력 문제 해결

windows terminal과 oh-my-zsh의 뭔지 모를 문제로 인해

터미널에 붙혀넣기를 수행할때 한글자씩 천천히 입력되는 문제가 있다.

~~없어졌다면 다행이고..~~

이걸 해결하기 위해선 ~/.zshrc에 다음 한줄을 추가한다.

```bash
DISABLE_MAGIC_FUNCTIONS="true"
```

이후 `source ~/.zshrc` 나 터미널을 재시작 해보면 해당 문제가 사라졌음을 알 수 있다.

# 터미널 추천 툴

## bat 설치 (clone cat)

[Releases · sharkdp/bat](https://github.com/sharkdp/bat/releases)

```bash
bat_version_amd64.deb <- 다운로드
sudo dpkg -i bat_version_amd64.deb
```

.zshrc에 추가

```bash
alias cat="bat"
```

이제 예쁜 cat으로 대체되었다

## logo-ls 설치

[Releases · Yash-Handa/logo-ls](https://github.com/Yash-Handa/logo-ls/releases)

```bash
logo-ls_amd64.deb <- 다운로드
sudo dpkg -i logo-ls_amd64.deb
```

zshrc에 다음 줄 추가

```bash
alias ls="logo-ls"
```

ls 명령어에 아이콘이 생겼다.

## alias explorer="explorer.exe” 설정

## asdf 설치 (version manager)

python, go, hugo, node, yarn 버전 문제를 한큐에

```bash
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.9.0
```

add to `~/.zshrc`

```bash
. $HOME/.asdf/asdf.sh

# append completions to fpath
fpath=(${ASDF_DIR}/completions $fpath)
# initialise completions with ZSH's compinit
autoload -Uz compinit && compinit
```

[ohmyzsh/plugins/asdf at master · ohmyzsh/ohmyzsh](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/asdf)

`source ~/.zshrc`

이제 asdf 명령어를 실행할 수 있습니다.

## asdf를 이용해 node 설치

node 플러그인 종속성 설치

```bash
apt-get install dirmngr gpg curl gawk
```

플러그인 설치

```bash
asdf plugin add nodejs https://github.com/asdf-vm/asdf-nodejs.git
```

node 최신 버전 설치

```bash
asdf install nodejs latest
```

node 시스템 전역 사용 선언

```bash
asdf global nodejs latest
```

yarn 설치 ← 이렇게 설치하는 것보단 asdf install yarn 하자

```bash
npm i -g yarn
```

zshrc에 yarn bin 폴더 추가 (yarn global add 시)

```bash
# FOR YARN BIN PATH #
export PATH="$(yarn global bin):$PATH"
```

## asdf를 이용해 python 설치

python 플러그인 종속성 설치

```bash
sudo apt-get update; sudo apt-get install make build-essential libssl-dev zlib1g-dev \
libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm \
libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
```

플러그인 설치

```bash
asdf plugin add python
```

node 최신 버전 설치

```bash
asdf install python latest
```

node 시스템 전역 사용 선언

```bash
asdf global python latest
```

## rustup 설치

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

1번 입력

깔끔하게 설치된다.