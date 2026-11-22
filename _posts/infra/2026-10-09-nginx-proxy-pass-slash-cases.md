---
comments: true
layout: post
title: nginx proxy_pass 슬래시 하나로 달라지는 경로, 6가지 경우 직접 찍어보기
date: 2026-10-09 09:00:00 +0900
category: infra
last_modified_at: 2026-11-22 18:00:00 +0900
---

홈서버에서는 nginx 하나가 443번으로 모든 요청을 받아, 경로에 따라 Docker로 띄운 앱들에게 나눠 준다. `/api`로 시작하면 백엔드 컨테이너로, 나머지는 프론트엔드 컨테이너로 보내는 식이다. 이 구성에서 제일 자주 헷갈리는 게 `proxy_pass` 끝의 슬래시다. 백엔드가 404를 뱉는데 원인이 슬래시 하나인 경우가 많다.

말로 외우면 자꾸 헷갈려서, 백엔드가 받은 경로를 그대로 돌려주게 만들고 경우별로 직접 찍어 봤다. 처음엔 nginx 1.27.5로 시험했고, 11월에 내 서버와 같은 버전(우분투 패키지의 1.18.0)으로 다시 돌려 보고, 실제 서버 설정에도 대 봤다.

## 시험 설정

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

## 결과

| 요청 | location | proxy_pass | 백엔드가 받은 경로 |
|---|---|---|---|
| `/a/users` | `/a/` | `http://host` | `/a/users` |
| `/b/users` | `/b/` | `http://host/` | `/users` |
| `/c/users` | `/c` | `http://host/` | `//users` |
| `/cx/users` | `/c` | `http://host/` | `/x/users` |
| `/d/users` | `/d/` | `http://host/v1` | `/v1users` |
| `/e/users` | `/e/` | `http://host/v1/` | `/v1/users` |
| `/f/users` | `~ ^/f/` | `http://host` | `/f/users` |

1.18.0과 1.27.5의 결과는 한 글자도 다르지 않았다. 몇 년 사이에 바뀐 동작이 아니니, 오래된 우분투 서버에서도 이 표를 그대로 믿어도 된다.

## 규칙은 하나다

[공식 문서](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)의 규칙은 이것 하나다.

- `proxy_pass`에 **URI가 없으면**(`http://host`처럼 호스트에서 끝나면) 요청 경로를 **그대로** 넘긴다. → a
- `proxy_pass`에 **URI가 있으면**(뒤에 `/` 하나만 붙어도 URI다) 요청 경로에서 **location과 일치한 부분을 그 URI로 바꿔서** 넘긴다. → b, e

나머지 이상한 결과는 전부 이 "바꿔치기"가 글자 단위로 일어나서 생긴다.

- **c `//users`** — location `/c`를 `/`로 바꾸면 `/c` + `/users`에서 `/c`만 `/`로 바뀌어 `//users`가 된다. 대부분의 프레임워크는 `//`를 404로 처리하거나 다르게 라우팅한다.
- **cx `/x/users`** — location `/c`는 접두사 매칭이라 `/cx/users`도 잡는다. 의도하지 않은 경로가 백엔드로 새어 들어간다. location 끝의 슬래시는 이걸 막는 역할도 한다.
- **d `/v1users`** — `/d/`를 `/v1`로 바꾸니 슬래시가 사라졌다. 로그에서 가장 찾기 어려운 경우다. 에러 메시지는 그냥 404일 뿐이다.

**location과 proxy_pass의 끝 슬래시를 맞추자.** 둘 다 붙이거나(b, e) 둘 다 안 붙이면 된다.

## 하나 더: URI를 붙이면 경로가 "정리"되어 넘어간다

다시 시험하면서 인코딩된 경로도 넣어 봤다. URI가 없는 `/a/`와 URI가 있는 `/h/`(`proxy_pass http://127.0.0.1:8081/;`)를 비교했다.

| 요청 | URI 없음 (`/a/`) | URI 있음 (`/h/`) |
|---|---|---|
| `…/x%2Fy` | `/a/x%2Fy` | `/x/y` |
| `…/%ED%95%9C` (한글 '한') | `/a/%ED%95%9C` | `/%ED%95%9C` |
| `…/a%20b` | — | `/a%20b` |
| `/h/users?id=1&q=%ED%95%9C` | — | `/users?id=1&q=%ED%95%9C` |

URI가 없으면 nginx는 손님이 보낸 경로를 글자 그대로 넘긴다. URI가 있으면 경로를 한 번 해석(디코딩)한 뒤 바꿔치기하고 다시 인코딩해서 넘긴다. 이 과정에서 **`%2F`(인코딩된 슬래시)가 진짜 `/`로 바뀐다.** 파일 경로나 ID에 슬래시를 인코딩해서 넣는 API라면, URI를 붙이는 순간 백엔드가 받는 경로가 쪼개져 404가 난다. 한글이나 공백은 다시 인코딩되어 원래와 같은 모양으로 넘어갔고, 쿼리 문자열(`?` 뒤)은 손대지 않았다.

## 정규식 location에서는 URI를 못 쓴다

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

## 내 서버 설정에 대 보니

표를 만들고 나서 정작 내 서버 설정을 열어 보니, 앱 하나가 이렇게 되어 있었다.

```nginx
location /api {
    proxy_pass http://127.0.0.1:<API포트>;
}
location / {
    proxy_pass http://127.0.0.1:<프론트포트>;
}
```

location과 proxy_pass 둘 다 끝 슬래시가 없으니 "맞추자" 규칙은 지킨 셈이다. 그런데 위 표의 cx 경우처럼 `/api`는 접두사 매칭이다. 서버 안에서 몇 가지 경로를 직접 보내 봤다.

```bash
for p in /api/__probe /apix /api-docs /ap; do
  curl -sk --resolve "<도메인>:443:127.0.0.1" -o out -w '%{http_code} %{content_type}' "https://<도메인>$p"
done
```

```
/api/__probe   404 application/json  {"status":404,"error":"Not Found","path":"/api/__probe"}
/apix          404 application/json  {"status":404,"error":"Not Found","path":"/apix"}
/api-docs      404 application/json  {"status":404,"error":"Not Found","path":"/api-docs"}
/ap            200 text/html         <!doctype html>...
```

JSON으로 404를 돌려준 건 스프링 백엔드다. `/apix`나 `/api-docs`처럼 `/api`로 **시작하기만 하면** 전부 백엔드로 갔다. `/ap`는 `/api`와 맞지 않으니 프론트엔드가 받았다. 지금은 백엔드에 그런 경로가 없어 404로 끝나지만, 나중에 백엔드에 `/api-internal` 같은 경로가 생기면 의도와 상관없이 밖에서 그대로 닿는다.

바꾸는 방법은 `location /api/`처럼 끝에 슬래시를 붙이는 것이다. 다만 바로 바꾸지는 않았다. nginx는 슬래시로 끝나는 location에 proxy_pass를 쓰면, 슬래시 없는 `/api` 요청에 301 리다이렉트로 `/api/`를 돌려준다. 프론트엔드가 `/api`를 그대로 POST로 부르는 곳이 있다면 리다이렉트 때문에 요청이 깨질 수 있다. 프론트 코드에서 그런 호출이 없는지 먼저 확인하고 바꿀 생각이다.

## 직접 확인하는 법

설정을 바꿀 때마다 헷갈린다면 위처럼 `return 200 "$request_uri"`만 하는 가짜 백엔드를 하나 띄워 두자. 실제 백엔드 로그를 뒤지는 것보다 훨씬 빠르다. 버전을 바꿔 가며 확인하고 싶으면 이미지 태그만 바꾸면 된다.

```bash
docker run -d --rm --name np -v $PWD/default.conf:/etc/nginx/conf.d/default.conf:ro nginx:1.18.0-alpine
docker exec np wget -qO- http://127.0.0.1/d/users
# backend got: /v1users
```

이미 운영 중인 서버라면, 위의 `--resolve` 방법으로 서버 안에서 몇 가지 경로를 보내 보고 응답의 모양(JSON인지 HTML인지)으로 어느 쪽이 받았는지 보는 것도 방법이다. 설정 파일을 읽는 것보다 실제로 어디로 가는지가 더 정확하다.
