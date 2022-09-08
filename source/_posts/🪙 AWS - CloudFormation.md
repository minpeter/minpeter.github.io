---
title: "🪙 AWS - CloudFormation"
description: "AWS 인프라에 대해 코드로 선언하는 방법이다.미리 코드로 적어둔 AWS 인프라 자원을 생성하거나 삭제할 수 있다.비슷한 서비스로는 Terraform이 존재하고 이를 통틀어 IaC (Infrastructure as Code) 라고 부른다.생성할 인프라 자원을 코드로 정"
date: 2022-06-15T23:50:15.412Z
tags: ["aws","cloudformation"]
categories: "aws"
---
## 이론
### CloudFormat이란?
AWS 인프라에 대해 코드로 선언하는 방법이다.
미리 코드로 적어둔 AWS 인프라 자원을 생성하거나 삭제할 수 있다.
비슷한 서비스로는 Terraform이 존재하고 이를 통틀어 IaC (Infrastructure as Code) 라고 부른다.

### 템플릿
생성할 인프라 자원을 코드로 정의한 파일
JSON, YAML 형식을 사용할 수 있다.

### 생성 순서
1. 템플릿을 CloudFormation에 업로드
2. CloudFormation에서 스택 생성 명령
3. AWS가 자동으로 템플릿에 작성된 순서대로 자동으로 생성

### 삭제 순서
1. CloudFormation에서 스택 삭제 명령
2. AWS가 자동으로 인프라 자원을 삭제

## 실습

### 스택 생성
```
Parameters:
  KeyName:
    Description: Name of an existing EC2 KeyPair to enable SSH access to the instances. Linked to AWS Parameter
    Type: AWS::EC2::KeyPair::KeyName
    ConstraintDescription: must be the name of an existing EC2 KeyPair.
  LatestAmiId:
    Description: (DO NOT CHANGE)
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'
    AllowedValues:
      - /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: t2.micro
      KeyName: !Ref KeyName
      Tags:
        - Key: Name
          Value: WebServer
      SecurityGroups:
        - !Ref MySG
      UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash
            yum install httpd -y
            systemctl start httpd && systemctl enable httpd
            echo "<h1>Test Web Server</h1>" > /var/www/html/index.html

  MySG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable HTTP access via port 80 and SSH access via port 22
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 0.0.0.0/0
```

위에 코드를 test_lab1.yaml 형식으로 저장하고 다음 단계를 진행

1. [CloudFormation Console](https://ap-northeast-2.console.aws.amazon.com/cloudformation/home?region=ap-northeast-2)에 접속해 스택 생성 클릭
2. 준비된 템플릿 선택 -> 템플릿 파일 업로드
3. 임의의 스택 이름 지정 (CF-TEST)
4. KeyName: 사용하고 있는 키 페어 선택

### 생성된 자원 확인

1. EC2 Console 접속
2. 인스턴스 -> 인스턴스
3. WebServer 인스턴스 IP 확인
4. ssh 및 http 접속 확인

### 스택 삭제

1. CloudFormation Console 접속
2. 스택 탭
3. 생성한 스택 선택 (CF-TEST)
4. 삭제 클릭

### 삭제된 자원 확인

1. EC2 Console 접속
2. 인스턴스 -> 인스턴스
3. WebServer 인스턴스 상태 확인 (Terminated)