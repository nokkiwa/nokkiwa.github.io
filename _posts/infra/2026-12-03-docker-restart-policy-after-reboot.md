---
comments: true
layout: post
title: docker stop으로 멈춘 컨테이너가 재부팅하면 살아난다, 재시작 정책 4가지를 데몬 재시작으로 직접 비교
date: 2026-12-03 09:00:00 +0900
category: infra
---

Redis 침해 사고를 정리하던 날, 당분간 쓰지 않을 컨테이너 몇 개를 `docker stop`으로 멈췄다. 멈추고 나서 찜찜한 생각이 들었다. "이거 서버 재부팅하면 다시 켜지는 거 아닌가?"

확인해 보니 실제로 그럴 뻔했다. 멈춘 컨테이너 중 두 개가 재시작 정책이 `always`였다. 이 글은 그날 서버의 실제 상태, 이틀 뒤 실제 재부팅에서 일어난 일, 그리고 정책 네 가지를 격리된 환경에서 직접 시험해 본 결과다. 시험 중에 문서만 봐서는 몰랐던 동작도 하나 나왔다.

## 그날 서버 상태

멈춘 직후 모든 컨테이너의 정책과 상태를 뽑았다. 프로젝트 이름은 가렸다.

```bash
docker ps -a --format '{% raw %}{{.Names}}{% endraw %}' | while read c; do
  echo "$c $(docker inspect $c --format '{% raw %}{{.HostConfig.RestartPolicy.Name}} {{.State.Status}}{% endraw %}')"
done
```

```
앱A-서버       always          exited    ← 방금 멈춤
앱A-웹         always          exited    ← 방금 멈춤
앱B-서버       always          running
캐시(redis)    unless-stopped  running
메시지큐       unless-stopped  running
옛날-실험용    unless-stopped  exited    ← 2025년 4월에 멈춘 뒤 그대로
빌드서버       no              exited
```

`always`인데 `exited`인 줄이 문제다. Docker 문서는 `always`를 이렇게 설명한다. 컨테이너가 멈추면 항상 다시 시작하되, **사람이 직접 멈춘 경우에는 Docker 데몬이 다시 시작될 때나 사람이 직접 다시 켤 때만 시작한다.** 서버를 재부팅하면 Docker 데몬도 다시 시작되니, 앱A는 재부팅과 함께 살아난다.

그래서 멈춘 컨테이너의 정책을 `no`로 바꿨다. 컨테이너를 지우지 않고 정책만 바꿀 수 있다.

```bash
docker update --restart=no 앱A-서버 앱A-웹
```

## 이틀 뒤, 실제 재부팅

11월 21일 0시 30분에 보안 업데이트로 서버가 재부팅됐다. 재부팅 뒤 컨테이너들의 마지막 시작 시각을 보면 누가 다시 켜졌는지 알 수 있다.

| 컨테이너 | 정책 | 재부팅 전 | 재부팅 후 |
|---|---|---|---|
| 앱B-서버 | always | 실행 중 | 0시 32분에 다시 시작 |
| 메시지큐 등 | unless-stopped | 실행 중 | 0시 32분에 다시 시작 |
| 앱A-서버·웹, 캐시(redis) 등 | no (전날 멈추며 바꿈) | 멈춤 | 멈춘 채 그대로 |
| 옛날-실험용 | unless-stopped | 멈춤 | 멈춘 채 그대로 |

정책을 바꿔 둔 덕에 앱A는 살아나지 않았다. 눈여겨볼 건 `옛날-실험용`이다. `unless-stopped`인데 2025년 4월에 멈춘 뒤로, 로그가 남아 있는 것만 세 번의 부팅을 거치면서도 한 번도 다시 켜지지 않았다. `unless-stopped`는 이름 그대로 "사람이 멈추지 않았다면" 다시 켠다.

다만 이 재부팅으로는 "`always`인 채로 멈춰 두면 정말 살아나는가"를 확인할 수 없었다. 그 전에 정책을 바꿔 버렸기 때문이다. 운영 중인 Docker를 일부러 재시작해 볼 수는 없어서, 따로 시험해 보기로 했다.

## 시험: 격리된 Docker 안에서 데몬 재시작

Docker 안에서 Docker를 돌리는 공식 이미지(`docker:dind`)를 띄우면, 운영 중인 컨테이너를 건드리지 않고 그 안의 Docker 데몬만 마음대로 재시작할 수 있다. 서버와 같은 버전(29.10.0)으로 맞췄다.

```bash
docker run -d --privileged --name rp-lab docker:29.10.0-dind
X="docker exec rp-lab docker"

for p in no on-failure always unless-stopped; do
  $X run -d --name "k-$p" --restart $p busybox:1.37.0 sleep 1d
  $X run -d --name "c-$p" --restart $p busybox:1.37.0 \
    sh -c 'trap "exit 0" TERM; while :; do sleep 1; done'
done

$X stop -t 2 k-no k-on-failure k-always k-unless-stopped \
             c-no c-on-failure c-always c-unless-stopped
docker restart rp-lab      # 안쪽 Docker 데몬 재시작 = 재부팅 흉내
$X ps -a --format '{% raw %}{{.Names}} {{.Status}}{% endraw %}'
```

컨테이너를 두 종류로 만든 이유가 있다. `k-`는 `sleep`만 돌리는데, 컨테이너의 첫 프로세스(PID 1)인 `sleep`은 종료 신호(SIGTERM)를 무시한다. 그래서 `docker stop`이 2초 기다리다 강제 종료(SIGKILL)하고, 종료 코드는 137이 된다. `c-`는 종료 신호를 받으면 `exit 0`으로 깔끔하게 끝나게 했다. 둘 다 사람이 `docker stop`으로 멈춘 건 똑같다.

결과는 이랬다.

```
c-always          Up 4 seconds
c-no              Exited (0) 8 seconds ago
c-on-failure      Exited (0) 8 seconds ago
c-unless-stopped  Exited (0) 8 seconds ago
k-always          Up 4 seconds
k-no              Exited (137) 7 seconds ago
k-on-failure      Up 4 seconds
k-unless-stopped  Exited (137) 7 seconds ago
```

표로 정리하면 이렇다. 모두 "사람이 `docker stop`으로 멈춘 뒤 데몬이 재시작된" 경우다.

| 정책 | 멈출 때 종료 코드 0 | 멈출 때 종료 코드 137 |
|---|---|---|
| `no` | 안 켜짐 | 안 켜짐 |
| `on-failure` | 안 켜짐 | **켜짐** |
| `always` | **켜짐** | **켜짐** |
| `unless-stopped` | 안 켜짐 | 안 켜짐 |

`always`는 문서대로였다. 놀란 건 `on-failure`다. 사람이 직접 멈췄는데도, **멈출 때 종료 코드가 0이 아니면 데몬 재시작 때 다시 켜졌다.** `on-failure`는 "실패하면 다시 시작"이니, 멈춰 두면 당연히 안 켜질 줄 알았다. Docker 문서에서 이 경우를 설명한 부분은 찾지 못했다. 같은 시험을 27.5.1 버전에서도 해 봤는데 결과가 똑같았다.

## 이게 왜 중요하냐면: 자바 앱은 143으로 끝난다

"멈출 때 종료 코드가 0이 아닌 경우"는 생각보다 흔하다. 이 서버에서 그날 멈춘 컨테이너들의 종료 코드를 보면 이렇다.

```
앱A-서버   Exited (143)
앱A-웹     Exited (0)
```

143은 128 + 15, 즉 종료 신호(SIGTERM)를 받고 끝났다는 뜻이다. 스프링 부트 같은 자바 앱은 `docker stop`을 받으면 정상적으로 정리하고 내려가지만, 종료 코드는 0이 아니라 143을 남긴다. 그러니 자바 앱을 "평소엔 알아서 살아나고, 멈춰 두면 안 켜지게" 하려고 `on-failure`를 고르면, 멈춰 둔 앱이 재부팅 때 살아난다. 시험 결과대로라면 그렇다.

PID 1이 신호를 어떻게 처리하느냐에 따라 종료 코드가 달라지는 이야기는 [docker stop이 10초씩 걸리는 이유](/infra/2026/09/29/docker-sigterm-cmd-forms.html)에 따로 정리해 두었다.

## 그래서 어떻게 쓰나

이번 일을 겪고 정한 규칙이다.

- **계속 돌아야 하는 서비스는 `unless-stopped`.** 재부팅하면 살아나고, 내가 멈춰 두면 멈춘 채로 남는다. 의도와 동작이 가장 잘 맞는다.
- **`always`는 멈춰 둘 일이 없는 것만.** 멈춰 둔 걸 잊고 재부팅하면 살아난다.
- **`on-failure`는 "멈춰 두기"용으로 믿지 않는다.** 종료 코드가 0이 아니면 재부팅 때 살아날 수 있다.
- **당분간 안 쓸 컨테이너는 멈추면서 정책도 `no`로.** `docker stop` 뒤에 `docker update --restart=no`를 습관처럼 붙인다.

마지막으로, 지금 내 서버에 "멈춰 있지만 재부팅하면 살아날" 컨테이너가 있는지는 이 한 줄로 찾을 수 있다.

```bash
docker ps -a --filter status=exited --format '{% raw %}{{.Names}}{% endraw %}' | while read c; do
  docker inspect "$c" --format '{% raw %}{{.Name}} {{.HostConfig.RestartPolicy.Name}} {{.State.ExitCode}}{% endraw %}'
done | awk '$2=="always" || ($2=="on-failure" && $3!=0)'
```

아무것도 안 나오면 안심해도 된다. 뭔가 나온다면, 그 컨테이너는 다음 재부팅 때 다시 켜진다.
