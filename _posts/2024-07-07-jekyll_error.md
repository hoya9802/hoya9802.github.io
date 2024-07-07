---
layout: single
title:  "[Git Blog] Jekyll 실행시 Binding Error"
typora-root-url: ../
categories: Blog
tag: [Blog, Jekyll]
author_profile: false
sidebar:
    nav: 'counts'
search: true
use_math: false
redirect_from:
  - /Blog/Git_Blog
published: true
---

## 문제 발생

Jekyll local server을 열기 위해 아래 코드를 실행시켰는데 다음과 같은 에러가 발생했습니다.
```php
$ bundle exec jekyll server

            ------------------------------------------------
Jekyll 4.3.2 Please append --trace to the build command 
             for any additional information or backtrace.
            ------------------------------------------------
```

## 문제 원인

에러를 자세히 확인해 보기 위해 에러문에 쓰여있는데로 아래 명령어를 실행하였습니다.
```php
$ jekyll serve --trace

Address already in use - bind(2) for 127.0.0.1:4000 (Errno:: EADDRINUSE)
```
해당 문제는 4000번 포트를 다른 프로세스가 이미 사용중이기 때문에 발생하는 에러였습니다.

## 문제 해결

해당 문제를 해결하기 위해서 몇가지 방법을 알려드리겠습니다.

1.현재 실행 중인 프로세스 확인 및 종료:

```php
$ lsof -i :4000 
```

위 코드를 사용하면 4000번 포트를 사용 중인 프로세스가 나오게 됩니다.

```php
$ kill -9 <PID>
```

이후 기존에 4000번 포트를 사용중인 PID 값을 넣어주면 기존 프로세스가 종료되고 jekyll 서버가 정상 작동합니다.

2.다른 포트를 사용하여 Jekyll 실행:

```php
$ jekyll serve --port 4001
```

위 코드는 local server을 다른 포트(4001)로 연결하여 jekyll 서버를 실행시키는 방법입니다.