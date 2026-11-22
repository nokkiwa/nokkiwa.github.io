---
comments: true
layout: post
title: nginx 취약점 5개가 한꺼번에 나왔을 때 내 서버 확인한 방법, 설정 조건 점검부터 우분투 패치 적용 확인까지
date: 2026-12-08 09:00:00 +0900
category: security
---

11월 19일, nginx 취약점 다섯 개가 한꺼번에 공개됐다는 소식을 봤다. 묶어서 [nginx2root](https://nginx2root.com/ko/)라고 부르고 있었다. 홈서버 앞단에서 443번을 받는 게 nginx라 그냥 넘길 수 없었다.

이런 소식이 나오면 "일단 업데이트"가 정답처럼 들린다. 그런데 우분투 패키지는 nginx.org보다 늦게 나오고, 그사이 내 서버가 실제로 위험한지는 따로 판단해야 한다. 이 글은 그날 설정 조건을 하나씩 대조한 과정과, 이후 우분투 패치가 실제로 적용됐는지 확인한 기록이다.

## 취약점마다 조건이 다르다

공개 페이지를 읽어 보니 다섯 개가 각각 다른 기능에서 나온 것이었다. 정리하면 이렇다.

| CVE | 취약해지는 설정 | 영향 시작 버전 |
|---|---|---|
| CVE-2026-123007 | HTTP/2 사용 | 1.9.5 |
| CVE-2026-123633 | HTTP/2 + 업스트림으로 요청 전달 | 1.19.7 |
| CVE-2026-121810 | HTTP/3 + 업스트림으로 요청 전달 | 1.25.0 |
| CVE-2026-123634 | `resolver` + 요청 값으로 정한 업스트림 + `mirror` | 1.13.4 |
| CVE-2026-113425 | 메일 프록시(`auth_http`) | 0.5.15 |

다섯 개가 서로 독립된 공격 경로라서, HTTP/2를 끈다고 나머지가 해결되지는 않는다. 그래서 내 서버가 **어떤 기능을 쓰고 있는지**부터 확인해야 했다.

## 1단계: 버전과 빌드 옵션

```
$ nginx -v
nginx version: nginx/1.18.0 (Ubuntu)

$ nginx -V 2>&1 | grep -o -- '--with-http_v2_module\|--with-http_v3_module\|--with-mail[^ ]*' | sort -u
--with-http_v2_module
```

우분투 22.04 기본 패키지라 1.18.0이다. HTTP/3 모듈과 메일 모듈은 빌드에 아예 들어 있지 않다. 그러니 HTTP/3(121810)과 메일 프록시(113425)는 설정을 볼 것도 없이 해당이 없다. HTTP/2 모듈은 들어 있으니, 실제로 켜져 있는지가 다음 확인 대상이다.

## 2단계: 설정에서 조건 찾기

`nginx -T`는 include된 파일까지 합쳐서 실제로 적용되는 설정 전체를 보여준다. 주석을 지우고 관련 지시어만 찾았다.

```bash
nginx -T 2>/dev/null | sed 's/#.*//' \
  | grep -nE 'listen[[:space:]].*http2|^[[:space:]]*http2[[:space:]]+on|quic|http3|resolver|mirror|^[[:space:]]*mail|auth_http'
```

나온 줄은 `video/quicktime`(MIME 타입 목록에 든 단어) 하나뿐이었다. `listen 443 ssl http2`도, `http2 on`도, `resolver`도, `mirror`도 없다. 설정 파일상으로는 HTTP/2를 쓰지 않는다.

## 3단계: 설정 말고 실제 동작으로 확인

설정을 잘못 읽었을 수도 있으니, 실제로 HTTP/2로 협상이 되는지 직접 붙어 봤다. 브라우저는 TLS 연결을 맺을 때 ALPN이라는 확장으로 "h2(HTTP/2) 되나요?"를 묻는다. `openssl`로 같은 질문을 던졌다.

```
$ openssl s_client -connect 127.0.0.1:443 -servername <도메인> -alpn h2 </dev/null 2>&1 | grep ALPN
No ALPN negotiated
```

이 서버에 붙어 있는 도메인 세 개 모두 같은 결과였다. 서버가 h2를 받아 주지 않으니 HTTP/2 경로(123007, 123633)도 해당이 없다.

## 함정: nginx가 하나가 아니었다

여기서 끝내려다 하나가 떠올랐다. 같은 서버의 다른 프로젝트가 **Docker 컨테이너 안에 자기 nginx**를 따로 띄우고 있었다. 호스트 nginx만 보면 놓친다.

```
$ docker exec <프로젝트-nginx> nginx -T 2>/dev/null | sed 's/#.*//' | grep -nE 'listen|http2|http3|resolver|mirror|proxy_pass|auth_http'
144:    listen 80 default_server;
158:    resolver 127.0.0.11 valid=10s ipv6=off;
164:        proxy_pass http://$chat_upstream;
174:        proxy_pass http://$app_upstream:3000;
```

이쪽은 1.30.6 버전이고, `resolver`와 변수로 정하는 업스트림(`proxy_pass http://$...`)을 쓰고 있었다. 123634의 조건 세 개 중 두 개가 맞는다. 컨테이너가 다시 만들어져 IP가 바뀌어도 따라가도록 Docker 내장 DNS를 주기적으로 다시 찾게 한 설정이다. 다만 마지막 조건인 `mirror`가 없어서 해당하지 않았다. HTTP/2도 쓰지 않는다(80번만 듣고, 앞단의 터널이 TLS를 처리한다).

컨테이너 안 nginx는 우분투 패키지 업데이트로 고쳐지지 않는다. 이미지를 새 버전으로 바꿔 다시 띄워야 한다. 지금은 조건이 맞지 않아 급하지 않지만, 나중에 `mirror`를 쓰게 되면 이 이미지부터 올려야 한다는 걸 메모해 뒀다.

그날의 결론은 "호스트 nginx도, 컨테이너 nginx도 다섯 개 모두 조건에 해당하지 않음. 우분투 패치가 나오면 적용"이었다.

## 우분투 패치는 언제, 어떻게 들어왔나

며칠 뒤 확인해 보니 패치는 이미 들어와 있었다. 패키지 변경 기록부터 봤다.

```
$ zcat /usr/share/doc/nginx-core/changelog.Debian.gz | sed -n '20,60p' | grep -E 'CVE-2026-[0-9]+$|^ --'
    - CVE-2026-123007
    - CVE-2026-123633
    - CVE-2026-113425
    - CVE-2026-123634
 -- (우분투 패키지 담당자)  Wed, 18 Nov 2026 10:46:35 -0600
```

`1.18.0-6ubuntu14.23` 판에 네 개가 들어 있었다. 우분투 쪽 시간으로 11월 18일 오전에 나온 패치다. 빠진 하나(121810)는 HTTP/3 문제라 1.18에는 해당 코드가 없다. 버전 번호는 1.18.0 그대로지만, 우분투는 이렇게 **버전을 올리지 않고 수정만 가져다 붙이는**(백포트) 방식으로 고친다. "1.31.9로 올려라"라는 안내만 보고 버전 숫자로 판단하면 안 되는 이유다.

설치 기록은 이렇다.

```
$ grep ' upgrade nginx-core' /var/log/dpkg.log
2026-11-20 06:14:47 upgrade nginx-core:amd64 1.18.0-6ubuntu14.22 1.18.0-6ubuntu14.23
2026-11-21 06:27:38 upgrade nginx-core:amd64 1.18.0-6ubuntu14.23 1.18.0-6ubuntu14.24
```

내가 손대지 않았는데 11월 20일 새벽 6시 14분에 설치됐다. 우분투의 자동 보안 업데이트(unattended-upgrades)가 깔아 준 것이다. `/var/log/unattended-upgrades/unattended-upgrades.log`에도 같은 시각에 nginx 패키지들을 올린다는 줄이 남아 있었다. 다음 날 같은 시각에는 또 다른 보안 수정이 담긴 14.24가 들어왔다.

## 마지막 확인: 실행 중인 nginx가 새 것인가

패키지를 깔았다고 끝이 아니다. nginx는 실행 중인 프로세스가 옛 실행 파일을 계속 쓸 수 있다. `nginx -s reload`는 설정만 다시 읽고 실행 파일은 바꾸지 않는다는 점은 공개 페이지에서도 강조하고 있었다. 그래서 실행 파일이 바뀐 시각과 서비스가 시작된 시각을 비교했다.

```
$ systemctl show nginx -p ActiveEnterTimestamp
ActiveEnterTimestamp=Sat 2026-11-21 06:27:43 KST

$ grep ' upgrade nginx-core' /var/log/dpkg.log | tail -1
2026-11-21 06:27:38 upgrade nginx-core:amd64 ... 1.18.0-6ubuntu14.24
```

패키지가 06:27:38에 설치되고, 5초 뒤 06:27:43에 nginx가 다시 시작됐다. 우분투 패키지는 업그레이드할 때 서비스를 알아서 재시작한다. 지금 돌고 있는 nginx는 패치가 들어간 14.24다.

## 이런 소식이 나오면 이 순서로

1. **취약점마다 조건을 표로 만든다.** "nginx 취약점"이 아니라 "어떤 기능을 쓸 때 취약한가"로 읽는다.
2. **빌드 옵션**(`nginx -V`)으로 아예 없는 기능부터 지운다.
3. **실제 적용 설정**(`nginx -T`)에서 조건이 되는 지시어를 찾는다.
4. **실제 동작**으로 한 번 더 확인한다. HTTP/2라면 `openssl s_client -alpn h2`.
5. **컨테이너 안 nginx도** 같은 순서로 본다. 이건 OS 업데이트로 안 고쳐진다.
6. 패치가 나오면 **변경 기록에 CVE 번호가 있는지**, **서비스가 패치 설치 뒤에 재시작됐는지**까지 본다.

버전 숫자만 보고 "1.18이니 위험하다"거나 "업데이트했으니 끝"이라고 판단했다면 둘 다 틀렸을 것이다.
