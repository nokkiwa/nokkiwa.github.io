---
comments: true
layout: post
title: docker stop이 10초씩 걸리는 이유, CMD 형태 5가지로 직접 비교해보기
date: 2026-09-29 09:00:00 +0900
category: infra
---

`docker stop`을 치면 어떤 컨테이너는 바로 꺼지고 어떤 컨테이너는 정확히 10초를 기다린 뒤에 꺼진다. 10초짜리는 앱이 종료 신호(SIGTERM)를 못 받아서 결국 강제 종료(SIGKILL)된 것이다. 이러면 앱의 종료 처리가 돌지 않는다. 처리 중이던 요청, 커넥션 정리, 버퍼 플러시가 전부 날아간다.

"shell form으로 쓰면 신호가 안 간다"는 설명이 많은데, 직접 해보니 항상 그렇지는 않았다. CMD를 5가지로 바꿔가며 비교해봤다.

### 1. 테스트 앱

SIGTERM을 받으면 메시지를 찍고 정상 종료하는 파이썬 앱이다.

```python
import signal, sys, time, os

def bye(signum, frame):
    print(f"[app pid={os.getpid()}] SIGTERM 받음 -> 정리 후 종료", flush=True)
    sys.exit(0)

signal.signal(signal.SIGTERM, bye)
print(f"[app pid={os.getpid()}] 시작", flush=True)
while True:
    time.sleep(1)
```

베이스 이미지는 `python:3.12-alpine`이고, 테스트 환경은 Docker 28.3이다.

### 2. 비교한 CMD

```dockerfile
# exec     : CMD ["python", "-u", "app.py"]
# shell1   : CMD python -u app.py
# shell2   : CMD python -u app.py && echo done
# script   : CMD ["./start.sh"]        # 안에서 python -u app.py
# scriptexec: CMD ["./start_exec.sh"]  # 안에서 exec python -u app.py
```

`start.sh`는 실무에서 흔히 보는 "준비 작업 후 앱 실행" 스크립트다.

```sh
#!/bin/sh
echo "[start.sh] 준비 작업"
python -u app.py        # start_exec.sh는 여기만 exec python -u app.py
```

### 3. 결과

| CMD | PID 1 | docker stop 시간 | 종료 코드 | 앱이 SIGTERM 받음 |
|---|---|---|---|---|
| exec | `python app.py` | 0.4초 | 0 | O |
| shell1 | `python app.py` | 0.4초 | 0 | O |
| shell2 | `/bin/sh -c ...` | **10.4초** | **137** | X |
| script | `/bin/sh ./start.sh` | **10.4초** | **137** | X |
| scriptexec | `python app.py` | 0.3초 | 0 | O |

종료 코드 137은 128 + 9(SIGKILL), 즉 강제 종료됐다는 뜻이다.

### 4. 핵심은 "PID 1이 누구냐"

`docker stop`은 컨테이너의 **PID 1에게만** SIGTERM을 보낸다. PID 1이 셸이면 셸이 신호를 받는다. 셸은 자식 프로세스인 앱에 신호를 넘겨주지 않는다. 게다가 리눅스는 PID 1에게 기본 신호 동작을 적용하지 않아서, 셸은 SIGTERM을 받고도 죽지 않고 버틴다. 그렇게 10초가 지나면 SIGKILL이 날아온다.

**shell1이 멀쩡했던 이유**가 재미있다. `CMD python -u app.py`도 원래는 `/bin/sh -c "python -u app.py"`로 실행된다. 그런데 이 이미지의 `/bin/sh`인 busybox는 `-c`로 받은 명령이 **단순 명령 하나뿐이면 셸을 띄우지 않고 그 명령으로 바로 교체(exec)**한다. 그래서 PID 1이 python이 됐다. `&& echo done`처럼 명령이 둘 이상이 되는 순간(shell2) 이 최적화가 꺼지고 셸이 PID 1에 남는다.

이 동작은 셸마다 다르다. 베이스 이미지를 바꾸거나 CMD에 뭔가 한 줄 덧붙이는 순간 조용히 깨질 수 있다. 결국 **shell form에 기대지 말고 exec form을 쓰는 게 맞다.**

### 5. --init(tini)만으로는 부족했다

신호 문제의 해결책으로 `docker run --init`(tini)이 자주 나온다. 스크립트 케이스에 붙여봤다.

```
script + --init                          : 0.4초, 종료 코드 143, 앱 로그에 "SIGTERM 받음" 없음
script + --init + TINI_KILL_PROCESS_GROUP=1 : 0.3초, 종료 코드 143, 앱 로그에 "SIGTERM 받음" 있음
```

`--init`만 붙이면 빨리 꺼지긴 한다. 하지만 **앱은 여전히 SIGTERM을 못 받았다.** tini는 자기 직계 자식인 `sh`에게만 신호를 넘기고, `sh`가 죽으면서 컨테이너가 내려갈 때 앱은 그냥 같이 정리됐다. 10초 대기만 사라졌을 뿐 종료 처리는 여전히 안 돈다. 오히려 "빨리 꺼지니 괜찮다"고 착각하기 쉬워서 더 위험하다.

tini가 프로세스 그룹 전체에 신호를 보내게(`TINI_KILL_PROCESS_GROUP=1`, 또는 `tini -g`) 해야 앱까지 신호가 닿았다.

### 6. 정리

- CMD와 ENTRYPOINT는 exec form(`["python", "app.py"]`)으로 쓴다.
- 시작 스크립트가 필요하면 마지막 줄을 `exec 앱`으로 끝낸다. 가장 확실한 방법이다(scriptexec).
- `docker stop`이 10초 걸리거나 종료 코드가 137이면 신호 전달 문제를 의심하자. `docker exec <컨테이너> cat /proc/1/cmdline`으로 PID 1이 누군지 보면 바로 보인다.
- `--init`은 좀비 프로세스 정리에는 좋지만 래퍼 스크립트의 신호 문제까지 자동으로 풀어주지는 않는다.

참고: [Why Your Dockerized Application Isn't Receiving Signals](https://hynek.me/articles/docker-signals/)

끝
