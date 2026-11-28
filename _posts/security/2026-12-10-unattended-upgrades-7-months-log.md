---
comments: true
layout: post
title: 우분투 자동 보안 업데이트 7개월치 로그를 열어 보니, 패키지 986개를 깔았지만 다 적용된 건 아니었다
date: 2026-12-10 09:00:00 +0900
category: security
---

지난번 nginx 취약점 글을 쓰면서, 내가 손대지 않았는데 새벽 6시에 nginx 보안 패치가 깔려 있던 걸 확인했다. 우분투의 자동 보안 업데이트(unattended-upgrades)가 한 일이었다. 고마운 일이지만, 그럼 이게 그동안 정확히 뭘 깔았고 깐 것이 정말 다 적용됐는지가 궁금해졌다.

남아 있는 로그를 처음부터 끝까지 세어 봤다. 결론부터 말하면 "깔린 것"과 "지금 돌고 있는 것"은 세 군데에서 달랐다.

## 설정은 우분투 기본값 그대로

```
$ cat /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";

$ systemctl cat apt-daily-upgrade.timer | grep -E 'OnCalendar|Randomized'
OnCalendar=*-*-* 6:00
RandomizedDelaySec=60m
```

매일 새벽 6시부터 7시 사이 아무 때나 한 번 돈다. 어떤 저장소에서 받을지는 `/etc/apt/apt.conf.d/50unattended-upgrades`의 `Allowed-Origins`가 정하는데, 기본값인 `jammy`, `jammy-security`, ESM 보안 저장소 네 줄이 그대로였다. 자동 재부팅(`Automatic-Reboot`)은 설정에 없으니 기본값인 꺼짐이다. 이게 뒤에서 중요해진다.

## 7개월 동안 한 일

로그는 `/var/log/unattended-upgrades/`에 한 달 단위로 돌려 가며 6개까지 남는다. 압축된 옛 로그까지 이어 붙여 셌다.

```bash
cd /var/log/unattended-upgrades
L() { for f in $(ls -r unattended-upgrades.log.*.gz); do zcat "$f"; done; cat unattended-upgrades.log; }
L | grep -c 'Packages that will be upgraded:'                                  # 89
L | grep -c 'No packages found'                                                # 123
L | grep 'Packages that will be upgraded:' | sed 's/.*upgraded: //' | wc -w     # 986
```

5월 1일부터 11월 28일까지, 뭔가를 깐 날이 89일, 깔 게 없던 실행이 123번이었다. 바꾼 패키지는 모두 986개다. 2~3일에 한 번꼴로 뭔가가 바뀌고 있었던 셈이다. 자주 나온 이름을 정확한 패키지 이름으로 셌다.

| 패키지 | 횟수 |
|---|---|
| `nginx-core` | 12 |
| `linux-generic` (5.15 커널 계열) | 12 |
| `vim-common` | 8 |
| `curl`, `libcurl4` | 각 6 |
| `linux-generic-hwe-22.04` (6.8 커널 계열) | 5 |
| `openssl`, `mysql-server-core-8.0` | 각 4 |

커널 줄이 두 개인 게 이상했다. 그 얘기는 바로 아래에서 한다.

## 다른 점 1: 커널은 깔리기만 하고 쓰이지 않는다

`dpkg.log`는 1년 넘게 남아 있어서 커널 설치 횟수를 셀 수 있었다.

```
$ uname -r
6.8.0-149-generic

$ (zcat /var/log/dpkg.log.*.gz; cat /var/log/dpkg.log.1 /var/log/dpkg.log) \
    | grep -oE ' install linux-image-(5\.15|6\.8)' | sort | uniq -c
     22  install linux-image-5.15
     14  install linux-image-6.8
```

지금 쓰는 건 6.8 커널인데, 13개월 동안 한 번도 부팅하지 않은 5.15 커널이 22번이나 설치됐다. 이유는 메타 패키지에 있었다.

```
$ dpkg -l 'linux-generic*' 'linux-image-generic*' | awk '/^ii/{print $2, $3}'
linux-generic 5.15.0.204.176
linux-generic-hwe-20.04 5.15.0.204.176
linux-generic-hwe-22.04 6.8.0-149.149~22.04.1
linux-image-generic 5.15.0.204.176
linux-image-generic-hwe-22.04 6.8.0-149.149~22.04.1
```

22.04 기본 커널(5.15)을 따라가는 `linux-generic`과 새 커널(6.8)을 따라가는 `linux-generic-hwe-22.04`가 둘 다 깔려 있다. 20.04 시절 이름(`hwe-20.04`)이 남은 걸 보면 판올림하면서 둘 다 남은 것 같다. 둘 다 있으면 양쪽 커널이 나올 때마다 모두 받는다. 5.15 커널 하나가 모듈만 457MB다. 오래된 커널은 자동으로 지워져서 디스크가 계속 쌓이지는 않지만, 매번 받고 설치하는 시간은 든다. 6.8이 부팅에 실패할 때 쓸 비상용이라 생각하고 일단 그대로 뒀다.

더 중요한 건 따로 있었다. **새 커널은 재부팅해야 적용되는데, 자동 재부팅은 꺼져 있다.** 지금 커널 6.8.0-149는 11월 13일 새벽에 설치됐고, 실제로 이 커널로 부팅한 건 11월 21일이다. 8일 동안은 패치 전 커널이 돌고 있었다. 지금도 재부팅이 필요하다는 표시가 하나 떠 있다.

```
$ cat /var/run/reboot-required.pkgs
network-manager
```

이 파일은 알림일 뿐이고 아무것도 하지 않는다. 로그인할 때 "System restart required" 문구로 보이는 게 전부라, 데스크톱으로만 쓰면 볼 일이 없다.

## 다른 점 2: 서비스 재시작은 자동이지만 짧게 끊긴다

패키지 자체가 서비스인 경우는 업그레이드할 때 알아서 재시작한다. nginx 글에서 본 것처럼 nginx는 설치 5초 뒤 다시 떴다. MySQL도 같았다.

```
$ journalctl -u mysql --since 2026-11-13 -o short | grep -E 'Stopp|Started' | head -3
11월 13 07:02:41 ... Stopping MySQL Community Server...
11월 13 07:02:46 ... Stopped MySQL Community Server.
11월 13 07:03:02 ... Started MySQL Community Server.
```

새벽 7시 2분부터 21초 동안 DB가 내려가 있었다. 13개월 동안 이런 MySQL 자동 업그레이드가 6번 있었다. 새벽에 도는 배치 작업이 DB에 붙는다면 이 시간대는 피하는 게 맞다.

## 다른 점 3: 라이브러리를 바꿔도 쓰던 프로그램은 옛 것을 쓴다

이게 제일 놓치기 쉽다. 11월 27일 새벽, XML 파서 라이브러리 `libexpat1`에 보안 수정(CVE 여덟 개)이 들어왔다. 라이브러리 파일은 바뀌었지만, 이미 실행 중인 프로그램은 메모리에 올려 둔 옛 파일을 그대로 쓴다. 디스크에서 지워진 옛 파일은 `/proc/<PID>/maps`에 `(deleted)`로 표시된다.

```bash
for p in $(pgrep -x nginx); do
  echo "$p $(grep -c 'libexpat.so.1.8.7 (deleted)' /proc/$p/maps) $(cut -d: -f3 /proc/$p/cgroup)"
done
```

```
(PID) 6 /system.slice/nginx.service          ← 호스트 nginx 5개 프로세스 모두
(PID) 0 /system.slice/docker-<컨테이너>.scope ← 컨테이너 안 nginx
```

호스트 nginx는 지워진 옛 expat을 쓰고 있었다. 컨테이너 안 nginx는 자기 이미지에 든 라이브러리를 쓰니 상관이 없다. 반대로 컨테이너는 호스트 업데이트로 절대 고쳐지지 않는다는 뜻이기도 하다.

nginx가 XML 파서를 왜 들고 있나 싶어 찾아봤다.

```
$ for m in /usr/lib/nginx/modules/*.so; do ldd "$m" | grep -q libexpat && echo "$m"; done
/usr/lib/nginx/modules/ngx_http_image_filter_module.so

$ nginx -T 2>/dev/null | sed 's/#.*//' | grep -c image_filter
0
```

이미지 크기 변환 모듈이 그림 라이브러리와 글꼴 라이브러리를 거쳐 expat을 끌어오고 있었다. 우분투 nginx는 깔린 모듈을 `modules-enabled`로 전부 불러오기 때문에, 쓰지 않는 모듈의 라이브러리까지 메모리에 올라온다. 설정에 `image_filter`가 한 줄도 없으니 요청으로 이 코드에 닿을 길은 없다. 그래서 급하지 않다고 봤다. 그래도 옛 코드가 돌고 있는 건 사실이니, 트래픽이 적은 시간에 `systemctl restart nginx`로 새로 띄우면 된다. `reload`는 실행 중인 프로세스를 그대로 두므로 이 경우엔 소용이 없다.

같은 방법으로 전체를 훑으면 데스크톱 프로그램(브라우저, dbus, 음향 서버 등)도 여럿 옛 라이브러리를 쓰고 있었다. `needrestart` 패키지를 깔면 이 확인을 업그레이드 때마다 해 주는데, 이 서버에는 없었다.

## 덤: 10월에 일주일 동안 뜬 경고

10월 16일부터 23일까지 로그에 이런 경고가 하루 두세 번씩 찍혔다.

```
WARNING Could not figure out development release: Distribution data outdated.
```

10월 우분투 새 버전이 나올 무렵, 버전 목록을 담은 `distro-info-data`가 그걸 따라가지 못해 생긴 경고로 보인다. 업데이트를 막지는 않았다. 경고가 뜨던 10월 17일에도 curl 보안 업데이트는 정상으로 깔렸다. 10월 23일 새벽에 `distro-info-data`가 자동으로 업데이트되면서 경고도 사라졌다.

## 자동 업데이트를 켜 뒀다면 가끔 확인할 것

1. **무엇을 깔았나**: `unattended-upgrades.log`에서 `Packages that will be upgraded` 줄
2. **재부팅이 밀려 있나**: `/var/run/reboot-required.pkgs`. 커널은 재부팅 전까지 옛것이다
3. **옛 라이브러리를 쓰는 프로세스가 있나**: `/proc/*/maps`의 `(deleted)`, 또는 `needrestart`
4. **컨테이너는 따로**: 호스트 업데이트로 고쳐지지 않는다. 이미지를 새로 받아야 한다
5. **서비스가 끊기는 시간**: DB처럼 재시작되는 패키지는 새벽 6~7시에 수십 초 내려갈 수 있다

자동 보안 업데이트는 "깔아 주는" 데까지가 일이다. 적용은 재부팅이나 재시작이 있어야 끝난다.
