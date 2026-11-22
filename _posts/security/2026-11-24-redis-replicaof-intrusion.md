---
comments: true
layout: post
title: 홈서버 Redis가 5개월 동안 남의 서버 복제본이었다, 발견부터 조치까지 10분 기록
date: 2026-11-24 09:00:00 +0900
category: security
---

11월 19일 오후, nginx 취약점 소식을 보고 우리 서버가 해당되는지 확인하던 중이었다. 취약점과는 상관없이 "그 김에 밖에서 열려 있는 포트나 보자"며 돌린 명령 하나에서 일이 커졌다. 집에서 돌리는 Redis가 **중국 IP의 Redis를 마스터로 모시는 복제본(replica)**이 되어 있었다. 흔적을 따라가 보니 처음 당한 건 6월 2일이었다. 다섯 달 넘게 몰랐던 셈이다.

이 글은 그날 실제로 친 명령과 출력, 그리고 그 출력을 어떻게 읽었는지를 시간 순서대로 적은 것이다. 공인 IP는 `<공인IP>`로, 공격자 IP는 끝자리를 가렸다.

## 17:20 — 밖에서 6379가 열려 있다

서버 안에서 공인 IP로 포트를 두드려 봤다. 공유기가 포트포워딩을 해주고 있다면 여기서 `open`이 나온다.

```bash
for p in 22 443 3306 6379 5672 15672 8081 3001; do
  timeout 3 bash -c "exec 3<>/dev/tcp/<공인IP>/$p" 2>/dev/null && echo "$p open" || echo "$p closed"
done
```

```
443 open
6379 open
5672 open
8081 open
3001 open
```

443은 nginx라 당연하다. 문제는 6379(Redis)와 5672(RabbitMQ)였다. 둘 다 외부에서 들어올 이유가 전혀 없는 포트다. 예전에 개인 프로젝트 앱 하나가 Redis 주소를 **공인 IP로 적어두는 바람에** 공유기에 6379 포워딩을 열어준 적이 있는데, 그걸 닫지 않고 잊어버린 것이다.

Redis 설정부터 봤다.

```bash
docker exec redis redis-cli CONFIG GET requirepass
docker exec redis redis-cli CONFIG GET protected-mode
```

```
requirepass
(빈 값)
protected-mode
no
```

비밀번호 없음, 보호 모드 꺼짐. 인터넷 누구나 `redis-cli -h <공인IP>`로 들어와서 아무 명령이나 칠 수 있는 상태였다.

## 17:21 — role:slave

가장 먼저 확인한 건 복제 상태였다. Redis를 노리는 공격은 대부분 `REPLICAOF`로 피해자 Redis를 자기 서버의 복제본으로 만드는 단계를 거치기 때문이다.

```bash
docker exec redis redis-cli INFO replication | head -3
```

```
# Replication
role:slave
master_host:175.24.232.x
```

나는 이 Redis를 혼자 쓰는 캐시로 띄웠다. 마스터가 있을 리가 없다. `175.24.232.x`는 내가 모르는 IP였고, whois로 조회해 보니 `TencentCloud`, 중국 텐센트 클라우드 대역이었다.

`/data` 폴더에는 처음 보는 파일이 하나 있었다.

```
-rw------- 1 redis redis   88 Nov 18 11:17 dump.rdb
-rw------- 1 redis redis  155 Jun  2 03:28 dump.rdb.broken
```

155바이트짜리 `dump.rdb.broken`을 `od -c`로 열어 봤다.

```
R E D I S 0 0 1 1 ... r e d i s - v e r 005 7 . 2 . 7 ...
l o a d e d _ f r o m _ m a s t e r \0 r e p l i c a t i o n
_ e x p l o i t _ s u c c e s s
```

두 가지가 보였다. 하나는 이 파일을 만든 Redis 버전이 **7.2.7**이라는 점이다. 우리 서버는 7.4.0이었으니 이건 공격자 마스터가 보낸 데이터다. 다른 하나는 안에 든 키와 값이 `loaded_from_master` = `replication_exploit_success`라는 점이다. 공격 도구가 "복제로 데이터 밀어넣기 성공"을 확인하려고 심는 표식으로 보인다. 파일 날짜가 6월 2일이니, 적어도 그때부터 이 Redis는 공격자 손에 있었다.

## 공격자가 실제로 뭘 했나 — 명령 통계로 읽기

Redis는 명령마다 호출 횟수와 실패 횟수를 센다. `INFO commandstats`다. 단, 이 숫자는 Redis가 재시작되면 0부터 다시 세기 때문에 컨테이너가 떠 있던 최근 20일치만 담겨 있다.

```bash
docker exec redis redis-cli INFO commandstats
```

필요한 줄만 추리면 이렇다.

| 명령 | 호출 | 실패/거부 | 의미 |
|---|---|---|---|
| `config\|set` | 184 | 실패 175 | 설정을 바꾸려다 거의 다 실패 |
| `save` | 45 | 실패 1 | 디스크에 파일 쓰기 시도 |
| `module\|loadex` | 0 | 거부 3 | 악성 모듈 올리기 시도, 실행 전 차단 |
| `set` | 15 | 거부 46 | 복제본이 된 뒤 쓰기가 막힌 것으로 보임 |
| `auth` | 15 | 실패 15 | 비밀번호 대입(애초에 비밀번호가 없어서 전부 오류) |
| `dbsize` | 252 | 0 | 데이터가 있는지 반복 확인 |

이 표를 보고 나서야 숨을 좀 돌렸다. Redis 공격은 보통 두 갈래다.

첫째는 **파일 쓰기**다. `CONFIG SET dir /root/.ssh`처럼 저장 폴더를 바꾸고, `dbfilename`을 `authorized_keys`로 바꾼 뒤 `SAVE`를 치면, Redis가 공격자의 SSH 키가 섞인 파일을 서버에 써준다. crontab 폴더를 노리는 변형도 있다. `config|set` 184번에 `save` 45번은 이 시도로 보인다.

둘째는 **악성 모듈 올리기**다. 피해자를 자기 마스터의 복제본으로 만든 다음, 복제 데이터 대신 `.so` 파일을 내려보내고 `MODULE LOAD`로 실행시킨다. 그러면 Redis 프로세스 권한으로 아무 코드나 돌릴 수 있다. `role:slave`와 `replication_exploit_success`는 이 공격의 앞단계가 성공했다는 뜻이다.

그런데 두 갈래 모두 마지막 단계에서 막혀 있었다. 이유는 Redis 7.0부터 바뀐 기본값이다.

- `enable-protected-configs no` — `dir`, `dbfilename` 같은 위험한 설정은 실행 중에 `CONFIG SET`으로 바꿀 수 없다. 그래서 `config|set`이 175번 실패했다.
- `enable-module-command no` — `MODULE LOAD`, `MODULE LOADEX` 명령 자체가 꺼져 있다. 그래서 `module|loadex`는 실행도 안 되고 거부만 3번 찍혔다.

내가 잘해서 막은 게 아니다. 이미지 태그를 `latest`로 받아 둔 덕에 마침 7.x가 깔려 있었을 뿐이다. 이 Redis가 6.x였다면 결과는 달랐을 것이다.

## 17:25 — 복제부터 끊었다

```bash
docker exec redis redis-cli REPLICAOF NO ONE
docker exec redis redis-cli INFO replication | head -2
```

```
OK
# Replication
role:master
```

복제를 끊어도 포트가 열려 있는 한 1분 뒤에 다시 붙을 수 있다. 그래서 끊자마자 다음 확인으로 넘어갔다.

## 17:26 — 서버 본체까지 뚫렸나

Redis 안에서 끝났는지, 호스트까지 넘어왔는지가 제일 중요했다. 확인한 것들은 이렇다.

- **컨테이너 안 프로세스** — `docker top redis` 결과는 `redis-server *:6379` 하나뿐이었다. `/tmp`, `/var/tmp`, `/dev/shm`에도 새 파일이 없었다.
- **계정과 SSH 키** — `/etc/passwd`에서 로그인 가능한 계정은 원래 있던 것뿐이었고, `authorized_keys`는 아무 계정에도 없었다.
- **cron** — `/etc/cron.d`와 각 계정 crontab에 내가 넣지 않은 줄이 없었다.
- **밖으로 나가는 연결** — `ss -tnp`로 본 연결 중 모르는 곳은 SSH로 들어와 있던 `117.144.80.x` 하나였다. 상하이 쪽 통신사 대역이다. `grep -cE 'Failed password|Invalid user' /var/log/auth.log`를 쳐 보니 지금 남아 있는 로그에만 11만 6천 줄이 나왔다. 비밀번호 대입 시도이고, 성공한 로그인은 내 것뿐이었다.

Redis 컨테이너는 root가 아닌 `redis` 사용자로 돌고, 특권 모드(`Privileged`)도 꺼져 있었다. 공격이 성공했더라도 바로 호스트 root가 되는 구조는 아니었다. 다만 그건 "덜 위험했다"는 뜻이지 "괜찮았다"는 뜻은 아니다.

## 17:29 — 다시 만들고, 문을 닫았다

감염된 컨테이너를 그대로 쓰고 싶지 않았다. 데이터 볼륨만 살리고 최신 패치 버전으로 새로 띄웠다.

```bash
docker pull redis:7.4
docker stop redis && docker rename redis redis-old-compromised
docker run -d --name redis --restart unless-stopped -p 6379:6379 \
  -v <기존볼륨>:/data redis:7.4 redis-server --protected-mode no
```

새 컨테이너는 `redis_version:7.4.12`, `role:master`로 떴다. 보면 알겠지만 비밀번호도 없고 보호 모드도 그대로 껐다. 그 앱이 비밀번호 없이 붙도록 짜여 있었기 때문이다. 보호 모드는 비밀번호가 없으면 같은 기계 안(127.0.0.1)에서 온 연결만 받는데, 그 앱은 공인 IP로 접속하느라 공유기를 한 바퀴 돌아 들어와서 출발지가 공유기 주소로 찍혔다. 보호 모드를 켜면 앱이 바로 끊긴다. 앱을 고치기 전까지 버티려고 문은 방화벽으로 닫기로 했다. 지금 다시 한다면 비밀번호부터 걸 것이다.

그다음이 진짜 함정이었다. 서버에는 ufw 방화벽이 켜져 있었다. 그런데 **Docker가 `-p`로 연 포트는 ufw 규칙을 거치지 않는다.** Docker가 iptables에 자기 규칙을 먼저 넣기 때문이다. ufw만 믿고 있었으면 6379는 계속 열려 있었을 것이다. 그래서 Docker가 제공하는 `DOCKER-USER` 체인에 직접 규칙을 넣었다. 사설망에서 오는 연결은 통과시키고, 밖에서 6379·5672·61613으로 오는 연결은 버리는 내용이다. 이 부분은 할 얘기가 많아서 다음 글에 따로 정리한다.

마지막으로 SSH로 붙어 있던 IP는 ufw 맨 앞에 차단 규칙으로 넣었다.

```bash
ufw insert 1 deny from 117.144.80.x
```

같은 날 이 Redis를 쓰던 앱을 내리면서 Redis 컨테이너도 아예 정지시켰고, 이후 공유기에서도 6379 포워딩을 지웠다. 쓰지 않는 서비스가 제일 안전하다.

## 돌아보니

- **포트포워딩은 연 사람이 닫지 않으면 아무도 안 닫는다.** 앱 설정에 공인 IP를 적는 바람에 연 포트였다. 같은 서버 안의 앱이면 처음부터 내부 주소로 붙었어야 했다.
- **"방화벽 켜져 있음"은 Docker 앞에서는 반쪽짜리다.** `ufw status`가 깨끗해도 `docker ps`의 `0.0.0.0:포트`는 따로 확인해야 한다.
- **`INFO commandstats`는 사후 분석 도구로 쓸 만하다.** 로그를 따로 켜두지 않았는데도 공격자가 무엇을 시도했고 어디서 막혔는지 숫자로 남아 있었다. 다만 재시작하면 사라지니, 이상하다 싶으면 재시작 전에 먼저 떠 둬야 한다.
- **모르는 것도 남았다.** 6월부터 11월 사이에 몇 번이나 들어왔는지, 20일보다 앞선 기간에 무엇을 시도했는지는 알 수 없다. `dump.rdb.broken`이라는 이름을 누가 붙였는지도 확실하지 않다. 확인할 수 있는 건 위의 숫자까지다.

지금 집 서버에서 Docker를 쓰고 있다면 {% raw %}`docker ps --format '{{.Names}} {{.Ports}}' | grep 0.0.0.0`{% endraw %} 한 줄만 쳐 보길 권한다. 거기 나온 포트가 공유기에서도 열려 있다면, 나처럼 누군가 이미 들어와 있을 수 있다.
