---
title: "AWS - IAM"
description: "CLI, API 또는 codedoploy... AWS에서 뭔가를 해야할때 IAM가 자주 등장한다. 한번 간단하게 알아보자!!"
date: 2022-05-09T00:50:58.192Z
tags: []
categories: "aws"
---
## IAM 소개

사용자들의 계정에 대한 관리가 주된 목적이며,
유저를 관리하고 접근 레벨 및 권한에 대한 관리가 가능한 권한 관리 솔루션

> 모든 AWS에 대해 세분화된 액세스 제어를 제공합니다.
> IAM을 사용하면 서비스 및 리소스에 액세스할 수 있는 사용자와 액세스할 수 있는 조건을 지정할 수 있습니다.
> IAM 정책으로 인력 및 시스템에 대한 권한을 관리하여 최소 권한을 보장할 수 있습니다.

AWS IAM 대시보드에 적혀있는 설명이다.
IAM, Identity and Access Management의 약자이며 키 모양의 로고를 사용한다.
![](/images/a4b4483d-f6e3-4cc0-8fa1-c78a8b440f49-image.png)

IAM은 루트 계정보다 낮은 혹은 낮은 권한을 가진 하위 계정을 관리하는 서비스(?)이다.
이용 요금은 무제한 무료이며 계정 생성에 제한 또한 없다.

## IAM의 필요성
이제 IAM이 AWS에 접근할 수 있는 사용자를 관리하는 서비스라는 사실을 알았다.
그렇다면 왜 여러 계정을 추가로 생성해야하며 굳이 **최소 권한**으로 유지, 관리 해야 하는 걸까?

정확하진 않지만 내가 이해한걸 바탕으로 한가지 가정을 해보겠다.
만약 당신이 회사을 운영한다. 음... 그냥 CTO라고 가정하자.
그리고 그 회사가 클라우드를 이용한 서비스를, 특히 AWS을 이용해 서비스를 한다고 했을 때
회사의 AWS Root 계정을 생성할 것이다.
아직은 모르겠지만? 아마도 법인카드 같은걸 사용하는 계정을 생성할 것 같다.
회사에는 다양한 개발 부서가 있을 것이다.
각각 다른 부분의 개발을 담당하는 부서에서 관리해야되는 AWS 리소스는 다를 것이다.
어떤 부서가 필요 이상의 관리 권한을 받게 되면 그럴필요도 없고, 괜히 잘못된 접근으로 다른 부서의 서비스를 망가뜨릴 수도 있을거 같다.

또 당신이 어떤 서비스를 개발하고 있는데 모종의 이유로 .env 파일에 AWS 액세스키를 적어두었다.
그리고 github에 올릴때 .gitignore 파일 작성을 까먹어 그대로 .env 파일을 업로드 했다면...
또 IAM을 사용하지 않아 Root 계정의 액세스키가 노출됬다면 이런 글의 주인공이 될 수 있다.
![](/images/bdf34e08-c4e2-4dc0-9a1d-8015fa1aefb5-image.png)

물론 상황이 다른 경우도 있지만 해킹이나 실수로 키가 누출됬다는건 공통점이다.
이런 상황을 예방하거나 피해를 줄이기 위해 IAM이 존재한다.

방금까지 IAM 사용자 계정이 왜 필요한지에 대한 가정이였다.

## IAM 자격 증명 유형
![](/images/8eab1eb3-bc5e-479b-92b6-6fed61a9d6ce-image.png)
IAM AWS 자격 증명 유형에는 2가지가 있다.
액세스 키를 사용하는 프로그래밍 방식 액세스, 암호를 사용하는 AWS 관리 콘솔 액세스
앞으로 각각 키 방식, 암호 방식이라고 부르겠다.

2가지 유형을 동시에 선택할 수도 있는데 사용하는 이유와 환경에 따라 선택해주면 된다.

### IAM - 암호 방식

AWS 관리 콘솔에 접근하기 위해 **비밀번호**를 인증 수단으로 사용한다.

IAM 사용자 계정은 마치 Root 계정처럼 웹 대시보드에 접근하여 서비스를 컨트롤 하지만
계정에 대한 모든 권한을 가지고 있는 Root 계정과는 다르게 Root 계정 또는 상위 권한 그룹에서 적용시킨 제한된 권한을 가지고 관리할 수 있다.


### IAM - 키 방식

AWS API, AWS CLI, AWS SDK에 접근하기 위해 **액세스 ID**와 **비밀 엑세스 키**을 인증수단으로 사용한다.

프로그래밍 방식만 활성화된 경우 관리 콘솔에 접근하진 못하지만, API/CLI/SDK에서 주어진 권한의 작업을 수행할 수 있게 된다.
보통 IAM을 처음 접하게 되는 방식일꺼 같다.
(AWS CLI 입문, AWS SDK를 이용한 프로그래밍... 암호 방식에 비해 사용이 거의 강제되기 때문)

## IAM 계정 관리을 편하게 해주는 것들

IAM에는 수많은 권한이 존재하고 조직이 커질수록 특정 그룹에서 공유하는 권한, 특정 사용자에게 필요한 특수한 권한 등이 생겨날 것이다.
차칫 복잡해질 수 있는 IAM 계정관리를 도와주는 기능을 알아보자!!

### 사용자 그룹
> 사용자 그룹은 IAM 사용자의 컬렉션입니다. 그룹을 사용하여 사용자 컬렉션에 대한 권한을 지정할 수 있습니다.

사용자를 모아둔 그룹
말그대로 비슷한 역활을 공유하는 사용자들을 하나로 뭉쳐두고 그룹에 공유하는 정책을 설정해두면 정책을 그룹단위로 관리할 수 있게된다.

### 사용자
> IAM 사용자는 계정에서 AWS와 상호 작용하는 데 사용되는 장기 자격 증명을 가진 자격 증명입니다.

실제로 접근할 수단을 제공하는 것
나중에 설명할 역활과 대조되게 장기 자격 증명을 가지고 있음
자신이 사용자 그룹의 정책과 사용자 고유의 권한을 적용 받음


### 역활
> IAM 역할은 단기간 동안 유효한 자격 증명을 가진 특정 권한이 있는 자격 증명입니다. 신뢰할 수 있는 개체가 역할을 맡을 수 있습니다.

사실 잘 모르겠다.
사용자가 존재하고, 해당 사용자가 충분한 권한이 있다면 작업에 필요한 역활를 위임 받아서 필요한 작업을 수행하고 작업이 끝나면 역활을 돌려주는... 그런 임시 권한을 말하는 것 같다.
나중에 IAM 역활만 따로 포스팅 해보도록 하겠다.



### 정책
> 정책은 권한을 정의하는 AWS의 객체입니다.

정책은 AWS 리소스에 액세스할 수 있는 사용자와 해당 리소스에서 수행할 수 있는 작업을 지정 해둔 JSON 문서이다.
자격 증명 또는 리소스에 정책을 연결하여 권한을 정의할 수 있다고 한다.
음... 이것도 정책 작성법으로 따로 포스팅 해보겠다.