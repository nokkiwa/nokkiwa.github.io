---
comments: true
layout: post
title: 모니터 케이블 하나 때문에 리눅스 머신이 50분 멈췄다, HDMI 연결 신호와 OOM
date: 2026-11-09 09:00:00 +0900
category: infra
---

집에서 24시간 켜두는 우분투 머신이 있다. 개인 프로젝트 몇 개가 상주로 돌고, 그중 하나는 60초마다 로그를 남긴다. 어느 날 아침 그 로그에 **새벽 4시 2분부터 4시 51분까지 50분 동안 아무 기록이 없었다.** 프로세스가 죽은 것도 아니고, 4시 51분부터는 아무 일 없었다는 듯 다시 찍혀 있었다.

### 1. 무슨 일이 있었나

`journalctl`로 그 시간대를 보니 답이 있었다.

```bash
journalctl --since "04:00" --until "05:00" | grep -iE "oom|watchdog"
```

```
Out of memory: Killed process ... (gnome-shell) ... anon-rss:5.6GB
```

메모리가 바닥나서 머신 전체가 50분 동안 사실상 멈춰 있었고, 4시 51분에 커널의 OOM killer가 **gnome-shell(5.6GB)**을 죽이면서 풀린 것이다. 서버 프로그램이 아니라 데스크톱 화면 프로세스가 범인이었다.

스왑도 2GB 중 95%가 차 있었다. 그리고 colord(색상 프로필 관리 데몬)가 **3.2GB**를 쓰고 있었다. 색상 관리 프로그램이 3GB를 쓸 일은 없다.

### 2. 누가 계속 메모리를 늘렸나

화면 쪽 프로그램들이 왜 커지는지 로그를 더 봤다. 같은 줄이 끝없이 반복되고 있었다.

```
gnome-shell: this.primaryMonitor is null      (10분에 277번)
Xorg: modeset(0): EDID vendor "RTK" ...        (시간당 약 2,500번)
```

모니터 정보(EDID)를 계속 다시 읽고, 그때마다 gnome-shell이 화면 배치를 다시 짜다가 "주 모니터가 없다"는 에러를 낸다. 모니터가 붙었다 떨어졌다를 반복하고 있다는 뜻이다. 커널 이벤트로 확인했다.

```bash
timeout 20 udevadm monitor --kernel --subsystem-match=drm | grep -c change
# 32
```

20초에 32번, 초당 1.6번씩 **HDMI 연결 신호(HPD, Hot Plug Detect)**가 깜빡이고 있었다. HDMI-3에 꽂아둔 세로형 휴대용 보조 모니터였다. 신호가 깜빡일 때마다 다음 일이 줄줄이 일어났다.

1. Xorg가 모니터를 다시 감지 (CPU 38%)
2. gnome-shell이 화면 배치를 다시 계산 (CPU 52%, 시간당 약 200MB씩 누수)
3. gsd-color·colord가 색상 프로필을 다시 만듦 (colord 3.2GB까지 증가)

이게 며칠 동안 쌓이다가 새벽에 메모리가 바닥난 것이다. 덤으로 로그가 폭주해서 journal 용량 상한(4GB)이 차버렸고, 그 이전 며칠치 로그는 이미 지워져 있었다.

### 3. 멈추게 한 것

케이블을 손보기 전에 커널이 먼저 손을 들었다.

```
HPD interrupt storm detected on HDMI-A-3, switching to polling
```

신호가 너무 자주 들어오면 커널이 인터럽트 대신 주기적으로 상태를 확인하는 polling 방식으로 바꾼다. 이후 깜빡임 이벤트가 0이 됐다. 재부팅할 때마다 부팅 직후 1~2분 정도 깜빡이다가 이 줄이 찍히고 조용해지는 패턴이 반복됐다.

### 4. 이미 새어 나간 메모리 회수

원인이 멈춰도 이미 커진 프로세스는 메모리를 돌려주지 않는다. 재부팅하지 않고 회수했다.

```bash
sudo systemctl restart colord      # 3.2GB → 15MB

export XDG_RUNTIME_DIR=/run/user/1000
systemctl --user kill --kill-who=main --signal=SIGKILL org.gnome.SettingsDaemon.Color.service
systemctl --user kill --kill-who=main --signal=SIGKILL org.gnome.Shell@x11.service
```

| 프로세스 | 회수 전 | 회수 후 |
|---|---|---|
| colord | 3.2GB | 15MB |
| gsd-color | 818MB | 26MB |
| gnome-shell | 2.1GB | 245MB |

gnome-shell은 systemd가 자동으로 다시 띄워서 5초쯤 뒤 돌아왔다. X11 세션이라 열린 창도 그대로였다.

**주의할 점이 하나 있다.** 처음엔 gnome-shell에 SIGTERM을 보냈는데, GNOME 42에서는 정상 종료 처리에 들어가면서 CPU 100%로 1분 넘게 화면이 멈췄다. 이미 상태가 꼬인 프로세스라 정리하는 데 시간이 오래 걸린 것으로 보인다. `systemctl --user restart`는 이 서비스가 수동 정지를 거부하도록 되어 있어 안 먹힌다. 그래서 SIGKILL로 바로 죽이고 systemd가 다시 띄우게 했다.

### 5. 재발 확인용 명령

```bash
# 0이면 정상. 숫자가 나오면 연결 신호가 다시 깜빡이는 중
timeout 20 udevadm monitor --kernel --subsystem-match=drm | grep -c change

# 최근 10분 EDID 재탐지 횟수
journalctl --since "-10 min" | grep -c 'EDID'

# 화면 프로세스가 다시 커지는지
ps -o rss=,comm= -C gnome-shell,colord,gsd-color
```

### 정리

- 서버처럼 쓰는 머신에 모니터를 꽂아뒀다면 **화면 쪽도 장애 원인이 될 수 있다.** 이번엔 케이블·모니터의 연결 신호 문제가 메모리 누수로 이어졌다.
- 프로세스가 아니라 머신 전체가 멈추는 증상이면 `journalctl`에서 `oom`부터 찾자. 누가 죽었는지가 범인 후보다.
- gnome-shell을 원격으로 재시작할 때는 SIGTERM보다 SIGKILL이 오히려 빨랐다.
- 상주 프로그램 쪽에 남긴 "주기마다 한 줄" 로그 덕분에 50분 공백을 바로 알아챘다. 이런 심장박동 로그는 싸고 쓸모가 많다.

끝
