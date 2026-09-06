---
comments: true
layout: post
title: nginx proxy_next_upstream 단일 서버 트러블슈팅
date: 2026-06-20 22:32:10 +0900
category: error
---

회사에서 프론트엔드 분에게 한가지 요청을 받았었다.

> 이상하게 개발 스웨거에서 api 요청할 때 서버 에러가 한번 나면 잠시동안 모든 api가 502 가 리턴돼요.

개발기에서 확인해보니 실제로 한 api에서 에러가 나면 다른 요청들이 잠시동안 모두 502를 리턴받으면서 cors를 뱉고 있었다.
아래는 해결 과정이다.

### 1. curl -I http://localhost 요청 확인
의도적으로 개발기에서 서버 에러를 발생시킨 후 dev 서버에 접근해서 ```curl -I http://localhost``` 을 날려보았다. 문제없이 200을 리턴했다.

### 2. nginx 설정 확인
서버 내에서 호출 시 문제가 없다면 외부에서 호출 시 문제가 생긴다고 판단했다. 그래서 nginx 설정을 확인해보았다.

```nginx
  location / {
      proxy_pass http://localhost:8080;
      proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
      
      ~ 생략 ~
      
      proxy_read_timeout 600s;
  }
```

처음에는 nginx 설정을 봤을때는 이상이 없어보였다. 그런데 ```proxy_next_upstream``` 이 눈에 들어왔다. 

만약 단일 서버 환경에서 해당 설정을 사용하게 된다면 서버 에러가 발생할 경우 다음 서버를 찾게되는 과정에서 다음 서버가 없기애 후에 오는 요청들은 요청 경로를 못찾아 502 bad gateway를 발생시키게 된다. 

현재 개발하고 있는 서비스는 단일 어플리케이션을 사용하고 있어서 해당 설정이 필요가 없었다. 해당 설정을 주석처리하고 nginx 를 restart 해주니 에러가 발생해도 다른 api들은 정상적으로 200을 리턴했다. 

해결 완료 !

```
# proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
```

### 3. 여러 서버일 경우
만약 여러 서버를 사용한다면 다음과 같이 바꿔주면 된다.

```nginx
upstream backend {
    server localhost:8080;
    server localhost:8081;  # 추가 서버
}
  location / {
      proxy_pass http://backend;
      proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
      
      ~ 생략 ~
      
      proxy_read_timeout 600s;
  }
```


끝