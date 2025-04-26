---
layout: single
title:  "EC2에서 Gmail API를 쓰기 위한 OAuth 2.0 인증 처리 방법"
typora-root-url: ../
categories: API
tag: [API, Google, Gmail, EC2]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: false
redirect_from:
  - /API/Gmail_ec2
published: true
---

## 문제 발생

로컬에서 개발 중이던 웹사이트에 Gmail API를 OAuth 2.0 인증 과정을 거쳐 토큰을 발급받고, 이를 사용해 이메일을 보내는 흐름은 문제없이 잘 작동했습니다.

그런데 서비스를 `EC2`에 배포한 뒤 문제가 발생했습니다.  

로컬에서 사용하던 토큰을 그대로 EC2에 복사하여 사용한 경우:

```bash
google.auth.exceptions.RefreshError: ('invalid_grant: Bad Request', '{ "error": "invalid_grant", "error_description": "Bad Request" }')
```

ec2서버에서 OAuth 2.0 인증을 시도하는 경우:

```bash
webbrowser.error: could not locate runnable browser
```

## 원인 분석

첫번째 에러는 OAuth 인증 과정에서는 리디렉션 URI(redirect URI) 설정이 필수적인데,
초기 설정 당시 Google Cloud Console에서 등록한 리디렉션 URI가 http://localhost 같은 로컬 개발용 주소로만 되어 있었습니다.
이로 인해 EC2 서버에서는 OAuth 인증 요청이 정상적으로 완료될 수 없었습니다.

두번째 에러는 Gmail API를 사용하기 위해서는 OAuth 2.0 인증 과정을 거쳐야 합니다.  
이 과정에서 사용자는 구글 로그인 화면을 통해 권한을 부여해야 하고, 이를 위해 기본적으로 웹 브라우저가 열려야 합니다.

하지만 `EC2` 서버는 일반적으로 CLI(Command Line Interface) 환경만 제공하며, 웹 브라우저가 실행될 수 있는 GUI 환경이 없습니다.  
이 때문에 Gmail API 클라이언트 라이브러리는 브라우저를 띄우려다 실패하고, 다음과 같은 에러를 발생시킵니다.

## 해결 방법

 * 만약, OAuth 2.0을 통한 Gmail API 인증정보를 발급 받는 법을 알고 싶다면 [**해당 블로그**](https://hoya9802.github.io/api/GmailAPI/)를 참고하세요.

1. Google Cloud Console에 로그인

2. Gmail API를 사용하는 프로젝트를 선택하거나, 새로운 프로젝트를 생성(새로 생성하는 경우, 애플리케이션 유형 -> <span style="color:red">**웹 애플리케이션**</span> 선택)

3. 왼쪽 메뉴에서 API & 서비스 → 사용자 인증 정보를 선택

4. OAuth 2.0 클라이언트 ID 항목에서 수정하고자 하는 클라이언트 ID를 찾고 클릭

5. 승인된 리디렉션 URL에 `localhost 주소`와 `ec2 도메인 주소`를 추가 후, 저장 버튼 클릭

6. Json 다운로드 버튼 클릭

7. <span style="color:red">웹 브라우저가 실행 가능한 **로컬 개발 환경**</span>에서 토큰을 발급받은 뒤, 해당 token.json 파일을 EC2 서버로 옮김

위 과정을 통해 EC2에 배포된 웹 애플리케이션에서도 Gmail API를 안정적으로 사용할 수 있습니다. 인증 과정은 다소 번거롭지만, 한 번만 잘 세팅해두면 이후엔 큰 문제 없이 동작하니 참고하세요.