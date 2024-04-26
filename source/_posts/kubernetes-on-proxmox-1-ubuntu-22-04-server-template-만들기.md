---
title: kubernetes on proxmox (1) - ubuntu 22.04 server template 만들기
date: 2022-10-03 20:19:00
tags:
categories: "home lab"
canonical_url: "https://minpeter.xyz/blog/k8s-via-proxmox-1"
---

# template이란?

그 동안 proxmox를 사용하면서 template이라는 개념을 처음 접했다.
template을 만들면 vm을 만들 때 매번 설치 과정을 거치지 않고 template을 복제해서 만들 수 있다.

kubernetes를 설치하려면 master node와 worker node가 필요하다.
못해도 3개 이상의 vm을 설정해야하는데 한번은 (아니 사실 두번이나..) 몰라서 vm을 만들 때 매번 설치 과정을 거쳤다.
3개 vm의 설치 프로세서를 동시에 진행하니 업데이트로 인한 네트워크 트래픽 부화가 심해져서 느려졌다.
그래서 template을 만들어서 vm을 만들 때 매번 설치 과정을 거치지 않고 template을 복제해서 만들 수 있도록 해보자.

# ubuntu 22.04 server template 만들기

## 1. vm 생성

proxmox에서 vm을 생성한다.
template으로 만들 vm의 이름은 ubuntu-22-04-server-template로 한다.
가능하다면 혼동을 피하기 위해 vm-id도 900으로 설정해주자.
media에서 Do not use any media를 선택한다.
뒤에서 언급하겠지만 클라우드 이미지를 사용할 예정이기 떄문이다.  
System 탭에서는 Qemu Agent를 체크한다.
이것은 vm이 실행되는 동안에 vm의 상태를 확인할 수 있게 해준다.
뭐 안해도 상관없긴 한데 나중에 설정하게 되면 켜져있는 편이 편하다.
디스크는 제거해주고 cpu는 1core, memory는 1GB로 설정한다.
네트워크는 일반적으로 vm을 만들 때와 동일하게 설정한다.

## 2. cloud init 설정

이제 생성된 vm의 hardward 탭에서 Add > CloudInit Drive를 클릭한다.
이렇게 하면 vm의 하드디스크에 cloud-init drive가 추가된다.
vm의 Cloud-Init 탭에서 User Data를 설정한다.
이때 네트워크 설정은 꼭 dhcp로 해야한다.
static으로 설정하면 복제한 vm에서 ip가 중복되어서 문제가 발생할 수가 있다.
도메인은 작성해주지 않아도 되고 SSH Public Key는 아래 과정을 따라해서 생성해주자.

### 2.1 ssh key 생성

ssh key를 생성한다.

```bash
ssh-keygen -t <key-type> -C <comment>
```

key-type은 rsa, dsa, ecdsa, ed25519 등이 있다.
comment는 key에 대한 설명이다.

`Enter file in which to save the key (C:\Users\minpeter/.ssh/id_rsa): ` 라는 질문이 나오면 proxmox_key.. 같은 적절한 이름을 입력한다.
그리고 `Enter passphrase (empty for no passphrase): ` 라는 질문이 나오면 비밀번호를 입력한다. (없어도 상관없다.) (없는게 좋다.)

생성된 public key를 복사한다.

```bash
cat ~/.ssh/<key-name>.pub
```

방금 proxmox_key라고 입력했다면 `cat ~/.ssh/proxmox_key.pub` 이렇게 입력하면 된다.

## 3. cloud image 다운로드

이미지는 약간의 작업을 위해 pve 쉘에서 진행한다.

### 3.1 pve 쉘 접속

web gui 또는 ssh root@<proxmox server ip>로 접속한다.

### 3.2 cloud image 다운로드

[](https://cloud-images.ubuntu.com/minimal/releases/jammy/release/)에서 다운로드 할 수 있다.
ubuntu-22.04-minimal-cloudimg-amd64.img를 다운 받으면 되기 때문에 링크를 복사하고 wget으로 다운받는다.

```bash
wget https://cloud-images.ubuntu.com/minimal/releases/jammy/release/ubuntu-22.04-minimal-cloudimg-amd64.img
```

이후 `qm set 900 -serial0 socket -vga serial0` 명령어를 입력한다.
이때 900은 vm의 id이다.

### 3.3 cloud image 변환

이미지를 변환한다.

```bash
mv ubuntu-22.04-minimal-cloudimg-amd64.img ubuntu-22.04.qcow2
qemu-img resize ubuntu-22.04.qcow2 32G
```

### 3.4 cloud image 추가

이미지를 추가한다.

```bash
qm importdisk 900 ubuntu-22.04.qcow2 local-lvm
```

물론 이때 900 또한 vm의 id이다.

### 3.5 cloud image 설정

여기까지 진행한 뒤 web gui에 접해보면 vm의 hardward에 Unuse disk 0이 추가된 것을 확인할 수 있다.
edit를 눌러 `Discard` 항목에 체크해주고 `Apply`를 눌러준다.

다음로 options 탭에서 `Boot Order`를 2번재로 설정해준다.  
Start at boot 항목은 Yes로 변경해준다.

이제 모든 설정이 끝났다.
모든 설정을 잘 했는지 확인한 뒤에 vm list에서 900(ubuntu-22-04-server-template)을 우클릭해서 Convert to Template을 눌러준다.
잠깐의 작업 끝에 template이 생성될 것이다.

## 2. template을 이용한 vm 생성

방금 전환된 template을 이용해 vm을 생성해보자.

이전과 동일하게 우클릭을 하게 되면 메뉴가 변경되었을 것이다.
이제는 `Clone`을 눌러준다.

이후 `Clone VM`이라는 창이 뜨는데 여기서 `Name`을 `ubuntu-test`로 변경해주고 `Mode`를 `Full Clone`으로 변경해준다.
Clone 클릭 이후 전원을 켜고 기다리면 끝이다.

# 다음편에선..?

지금 만든 template을 이용해 k8s-ctrlr와 k8s-node를 만들고, k8s-node 탬플릿을 추가로 생성, 총 k8s-ctrlr + 2개의 k8s-node를 만들고 설정해보자.

## 참고자료

- [youtube/Proxmox VE - How to build an Ubuntu 22.04 Template (Updated Method)](https://youtu.be/MJgIm03Jxdo)
- [blog/Proxmox VE - How to build an Ubuntu 22.04 Template (Updated Method)](https://www.learnlinux.tv/proxmox-ve-how-to-build-an-ubuntu-22-04-template-updated-method/)
