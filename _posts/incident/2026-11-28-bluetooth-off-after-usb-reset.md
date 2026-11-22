---
comments: true
layout: post
title: 켜 둔 블루투스가 하루 만에 혼자 꺼졌다, 범인은 내부 USB 재연결과 설정 한 줄
date: 2026-11-28 09:00:00 +0900
category: incident
---

홈서버 앞에 앉아 작업할 때는 블루투스 마우스와 키보드를 붙여 쓴다. 11월 18일 저녁에 블루투스를 켜 두었는데, 다음 날 저녁 마우스가 안 움직였다. 확인해 보니 블루투스가 꺼져 있었다. 나는 끈 적이 없다.

이 글은 "혼자 꺼졌다"는 증상에서 원인까지 가설을 하나씩 지워 간 과정이다. 결론은 짧다. **블루투스 장치가 새벽에 한 번 끊겼다 다시 붙었고, 다시 붙은 장치를 켜지 말라는 설정이 들어 있었다.** 다만 거기까지 가는 길에 확인할 게 몇 개 있었다.

## 가설 1: 블루투스 서비스가 죽었다

가장 먼저 의심한 건 `bluetoothd` 서비스였다.

```
$ systemctl is-active bluetooth
active
$ systemctl is-enabled bluetooth
enabled
```

살아 있고, 부팅 때 켜지도록 되어 있다. 서비스 로그에도 재시작 흔적이 없었다. 프로세스 번호(`bluetoothd[705]`)가 서버가 부팅된 10월 29일부터 그대로였다. 서비스는 범인이 아니다.

## 가설 2: 어딘가에서 차단(rfkill)을 걸었다

노트북의 비행기 모드처럼, 리눅스는 무선 장치를 소프트웨어나 하드웨어 스위치로 차단할 수 있다. `rfkill`로 확인한다.

```
$ rfkill list bluetooth
2: hci0: Bluetooth
	Soft blocked: no
	Hard blocked: no
```

차단도 없다. 그런데 장치 번호가 눈에 걸렸다. 이 서버의 무선 장치는 와이파이와 블루투스 두 개뿐이고 와이파이가 1번을 쓰는데, 블루투스는 0번이 아니라 2번이었다. 부팅 때 0번을 받았던 블루투스가 사라지고 새로 등록된 흔적일 수 있다. 이 시점에는 짐작이었다.

## 가설 3: 절전이나 화면 잠금 때 꺼졌다

이 서버는 24시간 켜 두는 기계지만 혹시 몰라 그 시간대 로그에서 `suspend`, `sleep`, 세션 종료를 찾았다. 아무것도 없었다. 대신 커널 로그에서 이런 줄이 나왔다.

```
11월 18 23:00:14 kernel: usb 1-1: USB disconnect, device number 2
11월 18 23:00:15 kernel: usb 1-1: new full-speed USB device number 12 using xhci_hcd
11월 18 23:00:15 kernel: usb 1-1: New USB device found, idVendor=0bda, idProduct=c821
11월 18 23:00:15 kernel: usb 1-1: Product: Bluetooth Radio
11월 18 23:00:15 kernel: usb 1-1: Manufacturer: Realtek
11월 18 23:00:15 kernel: Bluetooth: hci0: RTL: loading rtl_bt/rtl8821c_fw.bin
11월 18 23:00:15 kernel: Bluetooth: hci0: RTL: fw version 0x75b8f098
```

밤 11시 0분 14초에 USB 장치 하나가 끊겼고, 1초 뒤 같은 장치가 새 번호로 다시 잡혔다. 펌웨어도 다시 올라갔다. 블루투스가 꺼진 시점은 여기라고 봐도 된다.

## 잠깐, 나는 USB 동글을 안 쓰는데

처음엔 "블루투스 동글이 헐거워서 빠졌다 붙었나" 싶었다. 그런데 이 서버에는 블루투스 동글이 꽂혀 있지 않다. 그럼 USB로 잡히는 블루투스는 뭘까.

```
$ lspci | grep -i wireless
02:00.0 Network controller: Realtek ... RTL8821CE 802.11ac PCIe Wireless Network Adapter

$ lsusb | grep 0bda
Bus 001 Device 012: ID 0bda:c821 Realtek Semiconductor Corp. Bluetooth Radio
```

메인보드에 꽂힌 무선랜 카드 RTL8821CE가 답이었다. 이런 카드는 와이파이와 블루투스가 한 칩에 들어 있는데, **와이파이는 PCIe로, 블루투스는 카드 안쪽의 USB 선으로** 메인보드에 연결된다. 그래서 몸체는 메인보드 안에 있어도 리눅스에게는 USB 장치로 보인다. 손댈 일이 없는 부품이니 물리적으로 빠졌을 가능성은 없고, 장치나 드라이버 쪽에서 스스로 재연결한 것이다.

## 그런데 왜 다시 켜지지 않았나

장치가 다시 붙었으면 블루투스도 다시 켜져야 정상이다. 여기서 BlueZ(리눅스 블루투스 서비스) 설정 파일을 열었다.

```
$ grep -n AutoEnable /etc/bluetooth/main.conf
86:# AutoEnable defines option to enable all controllers when they are found.
89:AutoEnable=false
```

주석 그대로다. `AutoEnable`은 "장치가 발견되면 켤지"를 정하는 값이다. 부팅할 때 있던 장치뿐 아니라 **나중에 새로 붙은 장치에도 적용된다.** 이 값이 `false`였으니, 밤 11시에 새로 붙은 블루투스는 꺼진 상태로 등록되고 그대로 있었던 것이다.

궁금해서 우분투 패키지에 원래 들어 있는 설정과 비교해 봤다. 같은 버전(bluez 5.64-0ubuntu1.6)의 패키지를 내려받아 풀어 보니 기본값은 `AutoEnable=true`였다. 내 서버 파일은 패키지 원본과 딱 이 한 줄만 달랐다.

```
$ diff 패키지원본/main.conf 내서버/main.conf
89c89
< AutoEnable=true
---
> AutoEnable=false
```

언제, 왜 `false`로 바뀌었는지는 기록이 없어 모른다. 예전에 블루투스를 안 쓸 때 꺼 두려고 내가 바꿨을 가능성이 제일 높다. 설정을 바꾸면 이유를 같은 줄에 주석으로 남겨야 한다는 걸 이번에 다시 배웠다.

## 고친 방법

설정 한 줄을 원래 값으로 돌리고 서비스를 다시 띄웠다.

```bash
sudo cp /etc/bluetooth/main.conf /etc/bluetooth/main.conf.bak
sudo sed -i 's/^AutoEnable=false/AutoEnable=true/' /etc/bluetooth/main.conf
sudo systemctl restart bluetooth
bluetoothctl show | grep Powered
```

```
	Powered: yes
```

블루투스 서비스를 재시작하면 연결돼 있던 기기는 끊기니, 마우스나 키보드를 다시 연결해야 할 수 있다.

## 사흘 뒤, 진짜 시험

고쳤다고 끝이 아니다. 원인인 USB 재연결은 그대로라서 언젠가 또 일어날 텐데, 그때 블루투스가 살아 돌아오는지가 진짜 확인이다. 기회는 생각보다 빨리 왔다.

```
$ journalctl _TRANSPORT=kernel | grep 'usb 1-1: USB disconnect'
2026-11-18T23:00:14 ...
2026-11-22T07:45:58 ...
```

11월 22일 아침 7시 45분 58초에 똑같이 끊겼다가 다시 붙었다. 이번에는 확인해 보니 블루투스가 `Powered: yes`로 켜져 있었다. 증상은 사라졌다.

로그가 남아 있는 약 한 달(10월 23일~) 동안 이 재연결은 딱 두 번이었다. 자주 일어나는 일은 아니지만, 한 번 일어나면 다음 날까지 모르고 지나가기 쉬운 종류의 장애다.

## 아직 모르는 것

**USB가 왜 끊기는지는 아직 모른다.** 끊기기 직전 로그에는 블루투스와 관련된 줄이 하나도 없다. 의심 가는 건 USB 자동 절전이다.

```
$ cat /sys/bus/usb/devices/1-1/power/control
auto
$ cat /sys/module/btusb/parameters/enable_autosuspend
Y
```

블루투스 장치에 USB 자동 절전이 켜져 있다. 이 계열 Realtek 블루투스에서 절전이 재연결을 부른다는 이야기가 있긴 하지만, 내 서버에서 그게 원인이라는 증거는 아직 없다. 지금은 재연결이 일어나도 블루투스가 다시 켜지니 급하지 않다. 앞으로 재연결이 잦아지면 `btusb.enable_autosuspend=0`으로 절전을 끄고, 그 뒤 횟수가 줄어드는지 보는 순서로 확인할 생각이다.

## 비슷한 증상이라면 이 순서로

1. `systemctl is-active bluetooth` — 서비스가 살아 있는지
2. `rfkill list bluetooth` — 차단이 걸렸는지, 장치 번호가 늘었는지
3. `journalctl -k | grep -i 'usb.*disconnect'` — 장치가 끊겼다 붙었는지
4. `grep AutoEnable /etc/bluetooth/main.conf` — 다시 붙은 장치를 켜게 되어 있는지

3번에서 끊김이 보이고 4번이 `false`라면, 내가 겪은 것과 같은 경우일 가능성이 높다.
