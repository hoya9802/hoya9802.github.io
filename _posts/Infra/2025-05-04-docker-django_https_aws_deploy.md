---
layout: single
title:  "[Docker 인프라 구축] Django 앱을 AWS EC2에 HTTPS로 배포하기"
typora-root-url: ../
categories: Infra
tag: [Django, HTTPS, NGINX, uWSGI, Let's Encrypt, Docker, AWS, EC2, Route 53]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: false
redirect_from:
  - /infra/docker-django-https-aws-deploy
published: true
---

## HTTP와 HTTPS의 차이

웹사이트 주소를 보면 크게 `http://` 또는 `https://`로 시작하는 걸 볼 수 있습니다. 이 둘은 단순한 차이처럼 보이지만, 보안 측면에서는 아랙 그림과 같이 큰 차이를 갖고 있습니다.

![httpsvshttp](https://drive.google.com/thumbnail?id=1N7ecOsUJjGKt-ZAf32pHBhOhixw-QoH3&sz=w1000){: .align-center}
[이미지 출처](https://certera.com/blog/what-is-https-why-its-important-for-website-and-seo/)

### 🔓 HTTP (HyperText Transfer Protocol)

- **암호화되지 않은 통신 방식**입니다.
- 사용자의 아이디, 비밀번호, 결제 정보 등이 **평문으로 전송**됩니다.
- 공격자가 중간에서 정보를 가로채면 중요한 데이터가 유출될 수 있습니다. (이른바 **중간자 공격**)

### 🔐 HTTPS (HTTP + Secure)

- **SSL/TLS 인증서를 통해 암호화된 통신**입니다.
- 클라이언트와 서버 간의 데이터가 암호화되어 전달되므로 **도청 및 변조 위험이 줄어듭니다**.
- 최근 브라우저는 HTTPS가 없는 사이트에 '주의 요함' 경고를 표시하기도 합니다.

---

## 인프라 구조

이 글에서는 **Docker로 구성한 Django 서비스를 AWS EC2에 배포하고, Nginx와 Let's Encrypt를 이용해 무료 SSL 인증서를 적용하여 HTTPS로 서비스하는 전체 과정**을 아래 그림과 같이 구성합니다.

![httpsvshttp](https://drive.google.com/thumbnail?id=1-Y9azbgPQyUJ_fh7MsuUxkL8kXKtctBF&sz=w1000){: .align-center}

<h3>🛠 사용 기술 스택</h3>

- **Django**: 웹 애플리케이션
- **Docker + docker-compose**: 컨테이너 환경 구성
- **Nginx**: 리버스 프록시 및 SSL 처리
- **uWSGI**: Django와 Nginx 사이 연결
- **Let's Encrypt + Certbot**: 무료 HTTPS 인증서 발급 및 자동화
- **AWS EC2 + Route 53**: 배포 환경 및 도메인 연결

---

## 도메인 구입 (필수)

HTTPS인프라를 구축하기 위해서는 반드시 고유 도메인을 가지고 있어야 합니다. 해당 블로그에서는 가비아에서 구입을 하였습니다.

1. [https://www.gabia.com/](https://www.gabia.com/) 에 접속
2. 원하는 도메인 검색 → 구매 진행

## NGINX 세팅

<h3>⚙️ 최초 실행 시</h3>

- [**DH 파라미터(Diffie-Hellman Parameters)**](https://security.stackexchange.com/questions/94390/whats-the-purpose-of-dh-parameters) 생성<br>
  보안 강화를 위한 암호화 키 교환에 사용되며, 생성된 값은 볼륨에 저장됩니다.

- **Let's Encrypt 인증서 발급 준비 (ACME 챌린지 처리)**  
  HTTPS 인증서 생성을 위해 HTTP 경로로 인증 요청을 처리합니다.

이 작업은 서버에서 프로젝트를 **최초로 배포할 때만** 실행됩니다.

<h3>🔄 이후 실행 시</h3>

초기화가 완료된 이후, NGINX는 다음과 같은 역할을 지속적으로 수행합니다:

- **HTTP 요청을 HTTPS로 리디렉션**
- **Django 정적 파일 제공** (`/static` 경로)
- **uWSGI를 통한 Django 백엔드로 요청 전달**

docker/proxy/nginx/default.conf.tpl에 아래 코드를 넣어줍니다.

```sh
server {
    listen 80;
    server_name ${DOMAIN} www.${DOMAIN};

    location /.well-known/acme-challenge/ {
        root /vol/www/;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

docker/proxy/nginx/default-ssl.conf.tpl에 아래 코드를 넣어줍니다.

```sh
# HTTP 요청 처리용 서버 블록
server {
    listen 80;  # 80번 포트(HTTP)에서 요청을 수신
    server_name ${DOMAIN} www.${DOMAIN};  # 도메인 및 www 서브도메인 설정

    # 인증서 발급 시 필요한 acme-challenge 경로 처리
    location /.well-known/acme-challenge/ {
        root /vol/www/;  # 이 경로에서 인증용 파일 서빙
    }

    # 그 외 모든 요청은 HTTPS로 리디렉션
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS 요청 처리용 서버 블록
server {
    listen 443 ssl;  # 443번 포트(HTTPS)에서 요청 수신
    server_name ${DOMAIN} www.${DOMAIN};  # 도메인 및 www 서브도메인 설정

    # SSL 인증서 파일 경로 (certbot이 생성)
    ssl_certificate     /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;

    # SSL 설정 옵션 파일 포함 (추후 생성)
    include     /etc/nginx/options-ssl-nginx.conf;

    # DH 파라미터 설정 (보안 강화용)
    ssl_dhparam /vol/proxy/ssl-dhparams.pem;

    # HSTS 헤더 설정 (브라우저에게 HTTPS 사용 강제)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 정적 파일 서빙 경로 설정 (Django static files)
    location /static {
        alias /vol/static;
    }

    # 그 외 모든 요청은 uWSGI 서버로 전달 (Django 애플리케이션)
    location / {
        uwsgi_pass           ${APP_HOST}:${APP_PORT};  # uWSGI 서버 주소
        include              /etc/nginx/uwsgi_params;  # uWSGI 설정 포함
        client_max_body_size 10M;  # 업로드 파일 최대 크기 제한
    }
}
```

이후 docker/proxy/nginx/options-ssl-nginx.conf에 아래 코드를 넣어주세요.

 - 원하는 버전의 options-ssl-nginx.conf이 있는 경우 [여기](https://github.com/certbot/certbot)에서 버전을 선택해주세요.

```sh
# Taken from:
# https://github.com/certbot/certbot/blob/1.28.0/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf

ssl_session_cache shared:le_nginx_SSL:10m;
ssl_session_timeout 1440m;
ssl_session_tickets off;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;

ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
```

Django와 Nginx가 서로 통신하기 위한 uWSGI params을 docker/proxy/nginx/uwsgi_params에 만듭니다.

```sh
uwsgi_param QUERY_STRING $query_string;
uwsgi_param REQUEST_METHOD $request_method;
uwsgi_param CONTENT_TYPE $content_type;
uwsgi_param CONTENT_LENGTH $content_length;
uwsgi_param REQUEST_URI $request_uri;
uwsgi_param PATH_INFO $document_uri;
uwsgi_param DOCUMENT_ROOT $document_root;
uwsgi_param SERVER_PROTOCOL $server_protocol;
uwsgi_param REMOTE_ADDR $remote_addr;
uwsgi_param REMOTE_PORT $remote_port;
uwsgi_param SERVER_ADDR $server_addr;
uwsgi_param SERVER_PORT $server_port;
uwsgi_param SERVER_NAME $server_name;
```

이제 Nginx을 실행하기 위한 docker/proxy/run.sh 파일을 생성합니다.

```sh
#!/bin/bash

set -e

echo "Checking for dhparams.pem"
if [ ! -f "/vol/proxy/ssl-dhparams.pem" ]; then
  echo "dhparams.pem does not exist - creating it"
  openssl dhparam -out /vol/proxy/ssl-dhparams.pem 2048
fi

# envsubst이 명령어가 실행될때 .tpl에 있는 아래 변수들이 변경되는 것을 방지
export host=\$host
export request_uri=\$request_uri

echo "Checking for fullchain.pem"
if [ ! -f "/etc/letsencrypt/live/${DOMAIN}/fullchain.pem" ]; then
  echo "No SSL cert, enabling HTTP only..."
  envsubst < /etc/nginx/default.conf.tpl > /etc/nginx/conf.d/default.conf
else
  echo "SSL cert exists, enabling HTTPS..."
  envsubst < /etc/nginx/default-ssl.conf.tpl > /etc/nginx/conf.d/default.conf
fi

nginx -g 'daemon off;'
```

NGINX를 위한 docker/nginx/Dockerfile을 만듭니다.

```sh
FROM nginx:1.23.0-alpine

COPY ./nginx/* /etc/nginx/
COPY ./run.sh /run.sh

ENV APP_HOST=app
ENV APP_PORT=9000

USER root

RUN apk add --no-cache openssl bash
RUN chmod +x /run.sh

VOLUME /vol/static
VOLUME /vol/www

CMD ["/run.sh"]
```

## Certbot 세팅

docker/certbot/certify-init.sh에 아래 코드를 넣어주세요.

```sh
#!/bin/sh

# 에러 발생 시 즉시 스크립트 종료
set -e

# proxy 컨테이너의 80번 포트가 열릴 때까지 대기
until nc -z proxy 80; do
    echo "Waiting for proxy..."
    sleep 5s & wait ${!}
done

echo "Getting certificate..."

# certbot을 실행하여 SSL 인증서 발급 받기
certbot certonly \                             # 인증서만 발급 받고, 웹 서버에 설치는 하지 않음
    --webroot \                                # 웹서버의 루트 디렉토리에 challenge 파일을 생성하는 방식 사용
    --webroot-path "/vol/www/" \               # challenge 파일을 저장할 웹 루트 경로
    -d "$DOMAIN" \                             # 인증서를 발급받을 도메인
    --email $EMAIL \                           # 인증서 갱신 관련 알림을 받을 이메일 주소
    --rsa-key-size 4096 \                      # RSA 키 길이 (4096비트 권장)
    --agree-tos \                              # Let’s Encrypt 사용자 약관에 동의
    --noninteractive                           # 사용자 입력 없이 자동으로 실행
```

이제 certbot 설치를 위한 docker/certbot/Dockerfile 폴더을 다음과 같이 만듭니다.

```sh
FROM certbot/certbot:v1.27.0

COPY certify-init.sh /opt/
RUN chmod +x /opt/certify-init.sh

ENTRYPOINT []
CMD ["certbot", "renew"]
```

## Docker Compose 생성

Django을 배포하기 전에 app/app/settings.py에서 해당 코드로 업데이트해 주세요.

```python
import os
# ...

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "setmeinprod")

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = bool(int(os.environ.get("DJANGO_DEBUG", 0)))

ALLOWED_HOSTS = [] if DEBUG else os.environ.get("DJANGO_ALLOWED_HOSTS").split(',')

# ...
```

이후, 기존 django Dockerfile과 같은 위치에 docker-compose-deploy.yml를 다음과 같이 만듭니다.

```sh
version: "3.9"

services:
  app:
    build:
      context: .
    restart: always
    environment:
      - DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY}
      - DJANGO_ALLOWED_HOSTS=${DOMAIN}

  proxy:
    build:
      context: ./docker/proxy
    restart: always
    depends_on:
      - app
    ports:
      - 80:80
      - 443:443
    volumes:
      - certbot-web:/vol/www
      - proxy-dhparams:/vol/proxy
      - certbot-certs:/etc/letsencrypt
    environment:
      - DOMAIN=${DOMAIN}

  certbot:
    build:
      context: ./docker/certbot
    command: echo "Skipping..."
    environment:
      - EMAIL=${ACME_DEFAULT_EMAIL}
      - DOMAIN=${DOMAIN}
    volumes:
      - certbot-web:/vol/www
      - certbot-certs:/etc/letsencrypt/
    depends_on:
      - proxy

volumes:
  certbot-web:
  proxy-dhparams:
  certbot-certs:
```

 - certbot-web: 이 볼륨은 acme 챌린지 파일을 nginx 웹 서버와 공유하기 위해 사용됩니다. 이를 통해 인증 요청을 처리할 수 있도록 해당 파일을 인터넷에서 접근 가능하게 만듭니다.

 - proxy-dhparams: 이 볼륨은 처음 프록시를 실행할 때 생성되는 dhparams 파일을 저장하는 데 사용됩니다.

 - certbot-certs: 이 볼륨은 certbot이 생성한 인증서를 저장하고, nginx 프록시가 이를 사용할 수 있도록 공유합니다.

또한 변수들을 불러오기 위한 .env 파일과 EC2에서 사용하기 위한 .env.sample도 생성합니다.

```sh
DJANGO_SECRET_KEY=secretkey123
ACME_DEFAULT_EMAIL=email@example.com
DOMAIN=example.com
```

## 서버 생성

인스턴스를 생성할때 네트워크 설정때 아래 그림과 같이 2가지는 <span style="color:red">반드시</span> 체크해 주세요.
 - 인터넷에서 HTTPS 트래픽 허용
 - 인터넷에서 HTTP 트래픽 허용

![보안그룹](https://drive.google.com/thumbnail?id=1MU7usnznxrJaiWCNk18sjQoUzQurZa6f&sz=w1000){: .align-center}

인스턴스가 생성되면 아래 명령어를 통해 Public IPv4 DNS를 통해 EC2 서버에 접속합니다.

```sh
ssh ec2-user@<address>
```

Docker Compose 설치는 [여기](https://github.com/hoya9802/Docker-AWS-Deployment-Setup)를 참고하여 설치를 진행해주세요.

## GitHub에서 EC2 배포

EC2 서버에 접속한 다음 아래 명령어를 실행하여 SSH key를 발급 받습니다.

그리고 cat 명령어를 사용하여 pub key를 확인 후 복사합니다.

```sh
ssh-keygen -t ed25519 -C "GitHub Deploy Key"
cat ~/.ssh/id_ed25519.pub
```

이후 깃허브 프로젝트 > settings > Deploy keys에 접속하여 Title에 원하는 제목을 넣고, key에 앞서 복사한 pub key값을 넣어줍니다.

그리고 아래 코드를 실행하여 깃허브 프로젝트를 불러옵니다.

```sh
git clone <paste url here>
```

## DNS 세팅

AWS에서 route 53 > 호스팅 영역에 들어간 다음, 호스팅 영역 생성 버튼을 누르고 이전에 구매한 도메인 주소를 넣고 호스팅 영역 생성 버튼을 눌러줍니다.

이후 생성된 레코드 중 `ns-`로 시작하는 값들을 가비아 > 내 도메인 > 네임서버에서 호스명에 하나씩 넣어줍니다.

![보안그룹](https://drive.google.com/thumbnail?id=1Bsjb4WUQctug2UA-YOlAwRTM3loyk5n3&sz=w1000){: .align-center}

<h3> 🧐 왜 네임서버(NS)를 AWS로 바꾸는가?</h3>

도메인을 구입한 곳(예: 가비아)에서는 기본적으로 그 도메인의 **네임서버(NS)**를 자신들이 운영하는 네임서버로 설정합니다.  
하지만 우리가 **AWS의 Route 53**을 통해 DNS 설정을 관리하고 싶다면, **네임서버를 AWS에서 제공하는 값으로 변경해야** 합니다.

<h3>🔧 네임서버란</h3>

네임서버는 도메인에 대한 트래픽이 어디로 가야 하는지를 알려주는 시스템입니다.  
쉽게 말해, **"이 도메인의 주소 설정은 어디서 확인하면 되나요?"**를 결정하는 역할을 합니다.

<h3>✅ AWS 네임서버로 변경하면?</h3>

- **DNS 관리 권한을 AWS Route 53이 가지게 됩니다.**
- 즉, 도메인의 A레코드, CNAME 등 **세부적인 설정을 AWS에서 직접 할 수 있게 되는 것**입니다.

<h3>⚠️ 주의할 점</h3>

- 도메인의 **소유권**은 여전히 가비아에 있습니다.
- 단지, **DNS(네임서버) 관리 권한만 AWS에 위임**하는 것입니다.

> 이 작업을 통해, EC2에 배포한 Django 앱을 사용자가 구입한 도메인(example.com)으로 접근할 수 있게 됩니다.  

Route 53에서 해당 도메인을 누르고 레코드 생성 버튼을 눌러줍니다.

 - 레코드 이름에 서브도메인 이름 삽입
 - 레코드 유형은 CNAME을 선택
 - 값에는 생성한 EC2 인스턴스에 퍼블릭 IPv4 DNS을 삽입

![보안그룹](https://drive.google.com/thumbnail?id=1EtCNO5Et3xW5-J0dZWAW7m43z4rHQ8GU&sz=w1000){: .align-center}

이후 오른쪽 하단에 레코드 생성 버튼을 누르고 잠시 기다리면 EC2와 route 53이 연결됩니다.

## 앱 구성

이제 EC2 서버에서 명령어를 실행하여 .env 파일을 만듭니다.

이후 .env파일을 수정하여 실제값들로 변경합니다.

```sh
cp .env.sample .env
nano .env
```

```sh
DJANGO_SECRET_KEY=secretkey123
ACME_DEFAULT_EMAIL=email@example.com
DOMAIN=example.com
```

## 최초 인증서 발급

docker-compose-deploy.yml 파일이 있는 경로로 가서 아래 코드를 실행합니다.

```sh
docker-compose -f docker-compose-deploy.yml run --rm certbot /opt/certify-init.sh
```

<h3>프록시 대기 시간에 대한 안내</h3>

서비스를 처음 실행할 때 프록시가 준비될 때까지 시간이 다소 걸릴 수 있습니다.  
그 이유는 최초 실행 시, 프록시가 `dhparams` 파일을 생성하기 때문입니다.  
이 과정은 몇 분 정도 소요될 수 있으며, 생성된 파일은 볼륨에 저장되어 이후에는 다시 생성할 필요가 없습니다.

아래와 비슷한 문구가 출력되면 정상적으로 인증서를 발급 받은 것입니다.

```sh
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/yourdomain.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/yourdomain.com/privkey.pem
This certificate expires on 2025-08-01.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

아래 명령어를 통해 기존 docker service를 모두 종료 후, 다시 실행하면 됩니다.

```sh
docker-compose -f docker-compose-deploy.yml down
docker-compose -f docker-compose-deploy.yml up
```

그러면 이제 정상적으로 Django app이 HTTPS로 배포되는 것을 확인할 수 있습니다.

## 인증서 자동 갱신

Let's Encrypt에서 발급한 인증서는 <span style="color:red">90일</span>의 유효기간을 가지므로, 자동 갱신 설정이 중요합니다.

따라서 /home/ec2-user/renew.sh을 생성합니다.

```sh
#!/bin/sh
set -e

cd /home/ec2-user/django-docker-deployment-with-https-using-letsencrypt
/usr/local/bin/docker-compose -f docker-compose.deploy.yml run --rm certbot certbot renew
```

그런 다음, `chmod +x renew.sh` 명령어를 실행하여 실행 권한을 부여합니다. 

이후 다음 명령어를 실행합니다.

```sh
crontab -e
```

만약, `-bash: crontab: command not found` 에러가 Amazon Linux에서 발생하면 아래 명령어를 차례대로 실행합니다.

```sh
sudo yum install cronie
sudo systemctl enable crond
sudo systemctl start crond
```

이후 다시 crontab 명령어를 사용 후, 아래 명령어를 입력합니다.

```sh
0 0 * * 6 sh /home/ec2-user/renew.sh
```

이를 통해서 매주 토요일 00:00에 인증서를 갱신하는 renew.sh이 실행됩니다.

명령어와 관련된 자세한 설명은 [여기](https://crontab.guru/)를 확인하면 됩니다.

## 마치며

이제 도메인을 직접 구매하고, AWS와 연동한 뒤, Docker를 활용해 HTTPS까지 적용된 **인프라 환경**을 구축할 수 있게 되었습니다. 

한 번 구축해두면 이후부터는 자동 인증서 갱신과 안정적인 배포가 가능하므로,  
실전 프로젝트나 포트폴리오에도 매우 유용하게 사용할 수 있습니다.