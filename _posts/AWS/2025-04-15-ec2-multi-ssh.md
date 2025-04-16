---
layout: single
title:  "기존 키페어로 생성한 EC2에 맥북 SSH 키 추가 등록"
typora-root-url: ../
categories: AWS
tag: [AWS, EC2, SSH, authorized_keys]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: false
redirect_from:
  - /AWS/multissh
published: false
---

AWS EC2 인스턴스를 사용하다 보면 **노트북과 데스크탑을 오가며 접속해야 하는 경우**가 많습니다.  
EC2 인스턴스는 기본적으로 <span style="color:red">**하나의 key pair**</span>만 등록하게 되어 있어,  
새로운 컴퓨터에서 접속하려면 적절한 설정이 필요합니다.

이 글에서는 새로운 장비(맥북)에서 기존 EC2 인스턴스에 접속하기 위한 공개키 설정 방법을 단계별로 정리합니다.

---

## 1. EC2에서 SSH 인증 방식 이해하기

만약 ssh에 대해서 잘 모른다면 [해당](https://hoya9802.github.io/network/ssh/){:target="_blank"} 블로그를 참고하세요.

 - EC2 인스턴스는 `~/.ssh/authorized_keys` 파일에 등록된 퍼블릭 키 목록을 기준으로 SSH 인증합니다.

 - 각 클라이언트(로컬 컴퓨터)는 여기에 등록된 퍼블릭 키에 대응되는 개인 키를 가지고 있어야 EC2에 접속할 수 있습니다.


---

## 2. 맥북(노트북)에서 SSH 키 생성

터미널에서 .ssh 폴더로 이동한 후 해당 코드를 실행(윈도우는 따로 .ssh를 설치해야함)

```bash
ssh-keygen -t rsa -b 4096
```

이 명령어를 실행하면 아래와 같은 프롬프트가 나타납니다.

```bash
Enter file in which to save the key (/Users/yourname/.ssh/id_rsa):
```
 - 여기서 기본값 그대로 쓰면 ~/.ssh/id_rsa에 저장됩니다.
 - 하지만 직접 경로와 파일명 지정하고 싶다면 이렇게 입력하면 됩니다.


