---
comments: true
layout: post
title: nginx proxy_pass 슬래시 하나로 달라지는 경로, 6가지 경우 직접 찍어보기
date: 2026-10-09 09:00:00 +0900
category: infra
---

예전에 `proxy_next_upstream` 때문에 502가 나던 문제를 정리한 적이 있다. nginx 리버스 프록시 설정에서 그만큼 자주 헷갈리는 게 `proxy_pass` 끝의 슬래시다. 백엔드가 404를 뱉는데 원인이 슬래시 하나인 경우가 많다.

말로 외우면 자꾸 헷갈려서, 백엔드가 받은 경로를 그대로 돌려주게 만들고 경우별로 직접 찍어봤다. nginx 1.27.5 도커 이미지로 테스트했다.

### 1. 테스트 설정

```nginx
# 백엔드 역할: 받은 요청 경로를 그대로 응답
server {
    listen 8081;
    location / { return 200 "backend got: $request_uri\n"; }
}

server {
    listen 80;
    location /a/    { proxy_pass http://127.0.0.1:8081; }
    location /b/    { proxy_pass http://127.0.0.1:8081/; }
    location /c     { proxy_pass http://127.0.0.1:8081/; }
    location /d/    { proxy_pass http://127.0.0.1:8081/v1; }
    location /e/    { proxy_pass http://127.0.0.1:8081/v1/; }
    location ~ ^/f/ { proxy_pass http://127.0.0.1:8081; }
}
```

### 2. 결과

| 요청 | location | proxy_pass | 백엔드가 받은 경로 |
|---|---|---|---|
| `/a/users` | `/a/` | `http://host` | `/a/users` |
| `/b/users` | `/b/` | `http://host/` | `/users` |
| `/c/users` | `/c` | `http://host/` | `//users` |
| `/cx/users` | `/c` | `http://host/` | `/x/users` |
| `/d/users` | `/d/` | `http://host/v1` | `/v1users` |
| `/e/users` | `/e/` | `http://host/v1/` | `/v1/users` |
| `/f/users` | `~ ^/f/` | `http://host` | `/f/users` |

### 3. 규칙은 하나다

[공식 문서](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)의 규칙은 이것 하나다.

- `proxy_pass`에 **URI가 없으면**(`http://host`처럼 호스트에서 끝나면) 요청 경로를 **그대로** 넘긴다. → a
- `proxy_pass`에 **URI가 있으면**(뒤에 `/` 하나만 붙어도 URI다) 요청 경로에서 **location과 일치한 부분을 그 URI로 바꿔서** 넘긴다. → b, e

나머지 이상한 결과는 전부 이 "바꿔치기"가 글자 단위로 일어나서 생긴다.

- **c `//users`**: location `/c`를 `/`로 바꾸면 `/c` + `/users`에서 `/c`만 `/`로 바뀌어 `//users`가 된다. 대부분의 프레임워크는 `//`를 404로 처리하거나 다르게 라우팅한다.
- **cx `/x/users`**: location `/c`는 접두사 매칭이라 `/cx/users`도 잡는다. 의도하지 않은 경로가 백엔드로 새어 들어간다. location 끝의 슬래시는 이걸 막는 역할도 한다.
- **d `/v1users`**: `/d/`를 `/v1`로 바꾸니 슬래시가 사라졌다. 로그에서 가장 찾기 어려운 경우다. 에러 메시지는 그냥 404일 뿐이다.

**location과 proxy_pass의 끝 슬래시를 맞추자.** 둘 다 붙이거나(b, e) 둘 다 안 붙이면 된다.

### 4. 정규식 location에서는 URI를 못 쓴다

f처럼 `~` 정규식 location에서는 "일치한 부분"을 뭘로 바꿀지 정할 수 없다. 그래서 URI를 붙이면 설정 검사부터 실패한다.

```nginx
location ~ ^/g/ { proxy_pass http://127.0.0.1:8081/; }
```

```
$ nginx -t
nginx: [emerg] "proxy_pass" cannot have URI part in location given by regular expression,
or inside named location, or inside "if" statement, or inside "limit_except" block
```

정규식 location에서 경로를 바꾸고 싶으면 `rewrite`로 먼저 경로를 고친 뒤 URI 없는 `proxy_pass`를 쓴다.

```nginx
location ~ ^/g/ {
    rewrite ^/g/(.*)$ /$1 break;
    proxy_pass http://127.0.0.1:8081;
}
```

### 5. 직접 확인하는 법

설정을 바꿀 때마다 헷갈린다면 위처럼 `return 200 "$request_uri"`만 하는 가짜 백엔드를 하나 띄워두자. 실제 백엔드 로그를 뒤지는 것보다 훨씬 빠르다.

```bash
docker run -d --rm --name np -v $PWD/default.conf:/etc/nginx/conf.d/default.conf:ro nginx:1.27-alpine
docker exec np wget -qO- http://127.0.0.1/d/users
# backend got: /v1users
```

끝
