---
comments: true
layout: post
title: ufw deny를 넣었는데 안 막혔다, 규칙 순서와 이미 붙어 있는 연결을 실험으로 확인하기
date: 2026-12-05 09:00:00 +0900
category: security
---

홈서버 방화벽 규칙을 정리하다가 이상한 줄을 발견했다. 예전에 문제가 있던 외부 IP를 막으려고 넣어 둔 `DENY` 규칙이 목록 **맨 아래**에 있었다. 그 위로는 22, 80, 443 같은 `ALLOW` 규칙이 줄지어 있었다. 이 차단 규칙은 실제로 뭘 막고 있었을까.

결론은 "거의 아무것도 막지 못하고 있었다"이다. 이 글은 그걸 내 서버의 실제 규칙과 격리된 실험으로 확인한 기록이다. 실험 중에 "차단 규칙을 넣어도 이미 붙어 있는 연결은 끊기지 않는다"는 것도 같이 확인했다.

## 내 서버의 규칙 목록

`ufw status numbered` 결과에서 필요한 줄만 옮긴다. IP 끝자리는 가렸다.

```
[ 1] Anywhere                   DENY IN     117.144.80.x
[ 2] 22                         ALLOW IN    Anywhere
 ...
[ 5] 80                         ALLOW IN    Anywhere
[ 6] 443                        ALLOW IN    Anywhere
 ...
[ 9] 141.94.96.x 3333/tcp       DENY OUT    Anywhere                   (out)
[10] Anywhere                   DENY IN     141.94.96.x
```

1번은 지난달 Redis 사고 때 SSH 대입을 하던 IP를 막으면서 `ufw insert 1`로 맨 앞에 넣은 것이다. 9번과 10번은 그보다 훨씬 전에 넣은 규칙이다. `141.94.96.x`는 whois를 보면 프랑스 OVH의 서버 대역이다.

문제는 10번이다. ufw는 규칙을 **위에서부터 차례로 보다가 처음 맞는 규칙 하나만 적용**한다. `141.94.96.x`에서 22번 포트로 들어오는 연결은 10번까지 내려오기 전에 2번(`22 ALLOW IN Anywhere`)에 먼저 걸려 허용된다. 80, 443도 마찬가지다. 결국 10번이 막을 수 있는 건 위에서 허용하지 않은 포트뿐인데, 그런 포트는 기본 정책(들어오는 연결 거부)이 어차피 막는다. 있으나 마나 한 규칙이었다.

ufw가 실제로 만든 iptables 규칙을 보면 순서가 그대로 드러난다.

```
$ sudo iptables -S ufw-user-input
-A ufw-user-input -s 117.144.80.x/32 -j DROP
-A ufw-user-input -p tcp -m tcp --dport 22 -j ACCEPT
-A ufw-user-input -p udp -m udp --dport 22 -j ACCEPT
...
-A ufw-user-input -p tcp -m tcp --dport 443 -j ACCEPT
...
-A ufw-user-input -s 141.94.96.x/32 -j DROP
```

덤으로 하나 더 보인다. 22번이 tcp와 udp 두 줄로 들어가 있다. 예전에 `ufw allow 22`처럼 프로토콜을 빼고 넣어서다. SSH는 tcp만 쓰니 udp 22는 쓸데없이 열린 셈이다. 앞으로는 `ufw allow 22/tcp`처럼 프로토콜을 붙여서 넣어야겠다.

## 실험 1: 순서가 정말 결과를 바꾸나

머리로는 알겠는데, 눈으로 보고 싶었다. 운영 중인 서버 방화벽을 건드리지 않으려고 컨테이너 두 개로 작은 실험장을 만들었다. 컨테이너는 각자 자기 네트워크 공간을 가지므로, 안에서 ufw 규칙을 바꿔도 서버 방화벽에는 영향이 없다.

- `srv` (10.99.0.10) — ufw를 켜고, 8000번에서 간단한 웹 서버를 띄움
- `atk` (10.99.0.66) — "막고 싶은 IP" 역할, `srv:8000`에 접속을 시도함

먼저 서버와 같은 순서로 규칙을 넣었다. 허용이 위, 차단이 아래다.

```bash
ufw allow 8000/tcp
ufw deny from 10.99.0.66
```

```
[ 1] 8000/tcp                   ALLOW IN    Anywhere
[ 2] Anywhere                   DENY IN     10.99.0.66
접속 시도: 200 (curl exit 0)
```

막겠다고 넣은 IP에서 접속이 그대로 됐다. 이번엔 차단 규칙을 지우고 `insert 1`로 맨 앞에 다시 넣었다.

```bash
ufw delete deny from 10.99.0.66
ufw insert 1 deny from 10.99.0.66
```

```
[ 1] Anywhere                   DENY IN     10.99.0.66
[ 2] 8000/tcp                   ALLOW IN    Anywhere
접속 시도: 000 (curl exit 28)
```

이번엔 3초 동안 응답이 없어 시간 초과(`exit 28`)로 끝났다. 규칙 두 줄의 내용은 똑같고 순서만 바뀌었다. 특정 IP를 막는 규칙은 반드시 `ufw insert 1`로 맨 앞에 넣어야 한다.

## 실험 2: 막았는데 이미 붙어 있는 연결은?

Redis 사고 날, SSH 대입을 하던 IP를 막고 나서 "지금 붙어 있는 연결도 끊겼나"를 따로 확인했었다. 그 IP는 몇 분 전까지 SSH로 붙어 있었는데, 규칙을 넣은 직후 봤을 때는 마침 연결이 없었다. 규칙 덕분인지 우연인지는 알 수 없었다. 이번 실험장에서 제대로 확인해 봤다.

`srv`에 9000번으로 들어온 줄 수를 세는 작은 서버를 띄우고, `atk`에서 연결 하나를 연 채로 1초에 한 줄씩 보내게 했다. 그리고 도중에 차단 규칙을 맨 앞에 넣었다.

```
5초 뒤 받은 줄: 6
-> deny from 공격자 1번에 insert
다시 5초 뒤 받은 줄: 12
새 연결 시도(8000): 000 (exit 28)
```

차단 규칙을 넣은 뒤에도 **이미 열려 있던 연결로는 줄이 계속 들어왔다.** 새 연결만 막혔다. 이유는 ufw가 만드는 다른 체인에 있다.

```
$ sudo iptables -S ufw-before-input | head -3
-N ufw-before-input
-A ufw-before-input -i lo -j ACCEPT
-A ufw-before-input -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

ufw는 내가 넣은 규칙(`ufw-user-input`)보다 앞에서 "이미 성립된 연결(ESTABLISHED)의 패킷은 통과"를 먼저 처리한다. 매 패킷마다 규칙 전체를 다시 따지지 않으려는 정상적인 설계다. 그래서 새로 넣은 차단은 새로 맺는 연결에만 적용된다.

이미 붙어 있는 연결을 끊으려면 커널의 연결 추적 기록을 지우면 된다. `conntrack` 도구를 쓴다.

```
4초 뒤 받은 줄: 5
-> deny insert
4초 뒤 받은 줄: 10
-> conntrack -D -s 10.99.0.66
그 뒤 6초·10초에 받은 줄: 10 · 10
```

연결 기록을 지우자 그 연결의 다음 패킷은 "이미 성립된 연결"이 아니게 되고, 맨 앞의 차단 규칙에 걸려 버려졌다. 줄 수가 10에서 멈췄다. SSH처럼 사람이 붙어 있는 세션이라면 해당 `sshd` 프로세스를 직접 끝내는 방법도 있다.

## 실제 서버에서 1번 규칙은 일을 하고 있다

맨 앞에 넣은 1번 규칙은 제대로 동작하고 있는지 iptables 카운터로 봤다.

```
$ sudo iptables -L ufw-user-input -v -n | grep DROP
  490 29400 DROP  all  --  *  *  117.144.80.x   0.0.0.0/0
    0     0 DROP  all  --  *  *  141.94.96.x    0.0.0.0/0
```

11월 21일 재부팅 이후에만 `117.144.80.x`에서 온 패킷 490개를 버렸다. 그 IP는 차단된 뒤에도 계속 두드리고 있다는 뜻이다. 반면 맨 아래 규칙은 0이다. 그 IP가 안 왔을 수도 있고, 왔더라도 위의 허용 규칙에 먼저 걸려 이 줄까지 내려오지 않았을 수도 있다. 카운터만으로는 둘을 구분할 수 없다는 점까지가 이 규칙의 한계다.

참고로 9번 `DENY OUT`은 문제가 없다. 나가는 연결 규칙은 들어오는 규칙과 다른 체인(`ufw-user-output`)에 있고, 거기에는 이 한 줄뿐이라 앞에서 가로챌 규칙이 없다.

## 고치는 법과 습관

맨 아래 차단 규칙을 맨 앞으로 옮기면 된다.

```bash
sudo ufw delete deny from 141.94.96.x
sudo ufw insert 1 deny from 141.94.96.x
```

이번 일로 정한 습관이다.

- **IP 차단은 항상 `ufw insert 1`.** 그냥 `ufw deny`는 목록 맨 뒤에 붙는다.
- **넣고 나면 `ufw status numbered`로 순서를 눈으로 확인.**
- **포트 허용엔 프로토콜을 붙인다.** `ufw allow 22/tcp`.
- **급하게 막아야 하면 연결 기록도 지운다.** `sudo conntrack -D -s <IP>`. 이미 붙은 연결은 차단 규칙만으로는 안 끊긴다.

방화벽 규칙 목록은 "뭘 막고 있는지"를 보여주는 것 같지만, 실제로는 **순서까지 읽어야** 뭘 막는지 알 수 있다.
