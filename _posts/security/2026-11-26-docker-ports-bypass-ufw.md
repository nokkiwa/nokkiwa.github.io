---
comments: true
layout: post
title: ufw는 켜져 있었는데 왜 뚫렸나, Docker가 연 포트가 방화벽을 건너뛰는 길과 DOCKER-USER로 막기
date: 2026-11-26 09:00:00 +0900
category: security
---

[지난 글](/security/2026/11/24/redis-replicaof-intrusion.html)에서 홈서버 Redis가 외부에 열려 있다가 공격자 서버의 복제본이 된 일을 적었다. 정리하고 나서 제일 찜찜했던 건 따로 있었다. 이 서버에는 ufw가 켜져 있었고, 기본 정책도 "들어오는 연결은 거부"였다. 그런데 왜 6379와 5672가 밖에서 열려 있었을까.

결론부터 말하면 **Docker가 `-p`로 공개한 포트는 ufw 규칙을 아예 지나가지 않는다.** 이 글은 그걸 내 서버의 실제 규칙으로 확인한 과정과, 그때 넣은 차단 규칙이 무슨 뜻인지 한 줄씩 풀어 쓴 기록이다.

## 증거: 허용한 적 없는 포트가 열려 있었다

먼저 ufw 상태다. 기본 정책 줄만 옮긴다.

```
$ sudo ufw status verbose
상태: 활성
기본 설정: deny (내부로 들어옴), allow (외부로 나감), deny (라우팅 된)
```

허용 규칙 목록에 5672는 없었다. 그러니 ufw만 보면 5672는 막혀 있어야 한다. 하지만 같은 날 서버 안에서 공인 IP로 두드려 본 결과는 이랬다.

```
5672 open
6379 open
```

컨테이너 목록을 보면 이유가 보인다.

```
$ docker ps --format '{% raw %}{{.Names}}|{{.Ports}}{% endraw %}'
rabbitmq|0.0.0.0:5672->5672/tcp, [::]:5672->5672/tcp, ...
```

`0.0.0.0:5672`는 "이 서버의 모든 네트워크 주소에서 5672를 받아 컨테이너로 넘긴다"는 뜻이다. ufw가 뭐라고 하든 Docker는 이 약속을 지킨다.

## 패킷이 실제로 지나가는 길

ufw와 Docker는 둘 다 리눅스의 iptables(정확히는 netfilter)를 쓴다. 문제는 둘이 규칙을 넣는 **위치**가 다르다는 것이다.

밖에서 이 서버의 5672로 패킷이 들어오면 이렇게 흘러간다.

```
들어온 패킷 (목적지: 서버:5672)
   │
   ▼
PREROUTING (nat)  ── Docker 규칙: 목적지를 컨테이너 172.17.0.x:5672로 바꿈(DNAT)
   │
   │  목적지가 "이 서버"가 아니라 "컨테이너"가 됐으므로
   ▼
FORWARD  ──▶ DOCKER-USER ──▶ DOCKER-FORWARD(허용) ──▶ 컨테이너
   
INPUT  ← ufw의 허용/거부 규칙(ufw-user-input)은 여기에 있다. 이 패킷은 여기를 지나지 않는다.
```

nat 테이블에서 실제 규칙을 꺼내 보면 첫 번째 단계가 그대로 보인다.

```
$ sudo iptables -t nat -S DOCKER | grep 5672
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 5672 -j DNAT --to-destination 172.17.0.x:5672
```

목적지 주소가 바뀌는 순간 커널은 이 패킷을 "서버가 받을 패킷(INPUT)"이 아니라 "다른 곳으로 넘길 패킷(FORWARD)"으로 분류한다. ufw의 `allow`/`deny` 규칙은 INPUT 쪽에 있으니 이 패킷을 볼 기회가 없다. FORWARD 쪽에도 ufw 체인이 있긴 하지만, 그보다 먼저 Docker의 `DOCKER-FORWARD`가 공개 포트를 허용해 버린다.

```
$ sudo iptables -S FORWARD | head -5
-P FORWARD DROP
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-FORWARD
-A FORWARD -j ufw-before-logging-forward
-A FORWARD -j ufw-before-forward
```

순서가 핵심이다. `DOCKER-USER`가 맨 앞, 그다음이 Docker 자신의 허용 규칙, ufw는 그 뒤다. 그래서 Docker가 허용한 포트를 막으려면 **ufw가 아니라 맨 앞의 `DOCKER-USER`에 규칙을 넣어야 한다.** [Docker 공식 문서](https://docs.docker.com/engine/network/packet-filtering-firewalls/)도 같은 방법을 안내한다.

## 반대로, ufw에서 허용한 Docker 포트는?

이걸 알고 나서 ufw 규칙 목록을 다시 보니 웃긴 줄이 몇 개 있었다. 예전에 컨테이너를 띄우면서 "포트를 열어야 하나 보다" 하고 넣어 둔 `ALLOW IN` 규칙들이다. Docker가 공개한 포트에 대한 ufw 허용 규칙은 **아무 일도 하지 않는다.** 지워도 접속은 그대로 되고, 남겨 둬도 막을 때 도움이 안 된다. 방화벽 목록이 실제 상태를 보여준다고 믿게 만든다는 점에서는 오히려 해롭다.

## 막는 방법 세 가지와 내가 고른 것

| 방법 | 어떻게 | 장점 | 단점 |
|---|---|---|---|
| 내부 주소에만 공개 | `-p 127.0.0.1:6379:6379` | 가장 확실. 방화벽과 무관하게 밖에서 안 보임 | 다른 기기나 다른 경로로 접속하는 앱이 있으면 끊김 |
| `DOCKER-USER` 규칙 | 출발지·포트 기준으로 차단 | 컨테이너를 다시 만들 필요 없음 | 규칙을 직접 관리해야 함 |
| Docker의 iptables 끄기 | `daemon.json`에 `"iptables": false` | ufw 하나로 관리 | 컨테이너 네트워크가 깨지기 쉬움. 권하지 않음 |

원칙대로라면 첫 번째가 맞다. 같은 서버 안에서만 쓰는 Redis를 `0.0.0.0`에 공개할 이유가 없다. 다만 그날은 그 Redis를 쓰던 앱이 공인 IP로 접속하도록 짜여 있었다. 공인 IP로 접속하면 공유기를 한 바퀴 돌아 들어오기 때문에, `127.0.0.1`에만 공개하면 그 앱이 바로 끊긴다. 앱을 다시 빌드하기 전까지 버틸 방법이 필요했고, 그래서 두 번째를 골랐다.

## 넣은 규칙, 한 줄씩

ufw를 쓰는 서버라면 `/etc/ufw/after.rules` 맨 끝에 넣는 게 편하다. ufw가 켜질 때마다 같이 적용되기 때문이다. 넣기 전에 원본은 백업해 뒀다.

```
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -s 10.0.0.0/8 -j RETURN
-A DOCKER-USER -s 172.16.0.0/12 -j RETURN
-A DOCKER-USER -s 192.168.0.0/16 -j RETURN
-A DOCKER-USER -p tcp -m conntrack --ctorigdstport 6379 --ctdir ORIGINAL -j DROP
-A DOCKER-USER -p tcp -m conntrack --ctorigdstport 5672 --ctdir ORIGINAL -j DROP
-A DOCKER-USER -p tcp -m conntrack --ctorigdstport 61613 --ctdir ORIGINAL -j DROP
-A DOCKER-USER -j RETURN
COMMIT
```

- `:DOCKER-USER - [0:0]` — 체인을 선언한다. ufw가 규칙을 다시 읽을 때 이 체인을 비우고 아래 규칙을 새로 채우므로, 여러 번 다시 읽어도 규칙이 중복으로 쌓이지 않는다.
- 사설망 세 줄(`RETURN`) — 출발지가 사설 주소면 이 체인을 그냥 통과시킨다. 컨테이너끼리의 통신(172.16.0.0/12 대역)과 공유기를 돌아 들어오는 내 앱의 연결이 여기에 해당한다. 차단 규칙보다 **먼저** 와야 한다.
- `DROP` 세 줄 — 밖에서 6379(Redis), 5672(RabbitMQ), 61613(RabbitMQ STOMP)으로 오는 연결을 버린다.
- 마지막 `RETURN` — 나머지는 Docker의 원래 규칙으로 넘긴다. 이 줄이 없어도 체인 끝에서 자동으로 돌아가지만, 의도를 분명히 하려고 적었다.

여기서 제일 헷갈렸던 게 `--dport` 대신 쓴 `--ctorigdstport`다. FORWARD에 도착한 패킷은 이미 DNAT로 목적지가 바뀐 상태다. 위 nat 규칙을 다시 보면, 이 서버에서 8083으로 들어온 요청은 컨테이너의 8080으로 바뀐다. 이때 `--dport 8083`으로 막으려 하면 패킷의 포트는 이미 8080이라 규칙에 걸리지 않는다. `conntrack --ctorigdstport`는 연결 추적 정보에 남아 있는 **바뀌기 전의 원래 목적지 포트**를 보기 때문에, 바깥에서 두드린 번호 그대로 막을 수 있다. 이번에 막은 세 포트는 바깥 번호와 안쪽 번호가 같아서 `--dport`로도 동작했겠지만, 습관을 이쪽으로 들이는 게 안전하다.

적용은 `sudo ufw reload` 한 번이면 된다.

## 재부팅해도 남는가

규칙을 넣고 하루 반쯤 지나 보안 업데이트 때문에 서버가 재부팅됐다(11월 21일 0시 31분). Docker가 켜지면서 규칙을 지우지 않을까 걱정했는데, 재부팅 후 체인은 그대로였다.

```
$ sudo iptables -L DOCKER-USER -v -n --line-numbers   # 일부 열 생략
num   pkts bytes target  prot  source           destination
1        0     0 RETURN  all   10.0.0.0/8       0.0.0.0/0
2     664K   51M RETURN  all   172.16.0.0/12    0.0.0.0/0
3        3   306 RETURN  all   192.168.0.0/16   0.0.0.0/0
4        0     0 DROP    tcp   0.0.0.0/0        0.0.0.0/0   ctorigdstport 6379 ctdir ORIGINAL
5        0     0 DROP    tcp   0.0.0.0/0        0.0.0.0/0   ctorigdstport 5672 ctdir ORIGINAL
6        0     0 DROP    tcp   0.0.0.0/0        0.0.0.0/0   ctorigdstport 61613 ctdir ORIGINAL
7     748K  511M RETURN  all   0.0.0.0/0        0.0.0.0/0
```

Docker는 시작할 때 `DOCKER-USER` 체인이 없으면 만들고, 이미 있으면 안의 규칙을 지우지 않는다. ufw는 부팅 때 `after.rules`를 다시 읽는다. 둘이 맞물려서 규칙이 유지된다.

`DROP` 줄의 숫자가 0인 데는 이유가 있다. 재부팅하면 카운터가 0부터 다시 시작하고, 그사이 공유기에서도 이 세 포트의 포워딩을 지웠다. 바깥 패킷이 아예 서버까지 오지 않으니 막을 일도 없는 것이다. 그래서 이 표로는 "몇 번 막았다"를 보여줄 수 없다. 방화벽 규칙과 공유기 포워딩 삭제, 두 겹이 다 걸려 있다는 것만 확인할 수 있다.

## 놓치기 쉬운 것

- **IPv6** — `docker ps`에 `[::]:5672`도 같이 나온다. IPv6 주소가 있는 서버라면 같은 규칙을 `/etc/ufw/after6.rules`에도 넣어야 한다. 내 서버는 `ip -6 addr show scope global` 결과가 비어 있어서 해당이 없었다. 공유기가 IPv6를 나눠주는 집이라면 꼭 확인해야 한다.
- **사설망은 다 통과** — 위 규칙은 같은 공유기 아래 기기라면 누구나 6379에 붙을 수 있게 둔다. 집에 모르는 기기가 붙을 일이 없어서 그렇게 했지만, 손님용 와이파이를 같은 망에 두고 있다면 범위를 더 좁혀야 한다.
- **근본 해결은 아니다** — 이 규칙은 앱을 고치기 전까지 버티는 장치였다. 결국 그 앱과 Redis는 내렸다. 같은 서버에서 도는 다른 프로젝트의 DB 컨테이너는 처음부터 `127.0.0.1:3307`에만 공개해 둔 덕에 이번 일과 무관했다. 내부용 컨테이너는 그렇게 띄우는 게 맞다.

Docker를 쓰는 집 서버라면 아래 두 줄의 결과를 나란히 놓고 보길 권한다. 첫 줄에 나온 포트는 두 번째 줄에 있든 없든 ufw와 상관없이 열려 있다. 두 번째 줄만 보고 안심하면 안 된다.

```bash
docker ps --format '{% raw %}{{.Ports}}{% endraw %}' | tr ',' '\n' | grep 0.0.0.0
sudo ufw status
```
