---
comments: true
layout: post
title: Go 1.22 루프 변수 변경, 컴파일러 버전이 아니라 go.mod가 정한다
date: 2026-10-01 09:00:00 +0900
category: go
---

Go 1.22부터 `for` 루프 변수가 반복마다 새로 만들어진다는 건 많이 알려진 이야기다. 그런데 "Go 1.22로 올렸으니 이제 괜찮겠지" 하고 넘어가면 틀릴 수 있다. 새 동작을 켜는 건 **설치된 Go 버전이 아니라 `go.mod`의 `go` 줄**이기 때문이다.

같은 Go 1.22.12로 `go.mod`만 바꿔가며 직접 돌려봤다.

### 1. 테스트 코드

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	ids := []int{1, 2, 3}
	for _, id := range ids {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println("id:", id)
		}()
	}
	wg.Wait()

	var funcs []func()
	for i := 0; i < 3; i++ {
		funcs = append(funcs, func() { fmt.Println("i:", i) })
	}
	for _, f := range funcs {
		f()
	}
}
```

고루틴에서 루프 변수를 쓰는 경우와, 클로저를 슬라이스에 모아뒀다가 나중에 호출하는 경우 두 가지다.

### 2. go.mod가 `go 1.21`일 때

```
$ go version
go version go1.22.12 linux/amd64
$ go run .
id: 3
id: 3
id: 3
i: 3
i: 3
i: 3
```

컴파일러는 1.22인데 결과는 예전 동작 그대로다. 모든 고루틴과 클로저가 **같은 변수 하나**를 공유해서 마지막 값인 3만 찍힌다.

### 3. go.mod가 `go 1.22`일 때

```
$ go run .
id: 3
id: 1
id: 2
i: 0
i: 1
i: 2
```

같은 바이너리로 `go.mod`의 한 줄만 바꿨는데 결과가 달라졌다. 이제 반복마다 변수가 새로 생겨서 각자 자기 값을 들고 간다. 고루틴 출력 순서가 섞인 건 스케줄링 때문이라 정상이다.

### 4. 왜 go.mod 기준인가

공식 블로그 글([Fixing For Loops in Go 1.22](https://go.dev/blog/loopvar-preview))에 이유가 나온다. 이 변경은 언어 의미 자체를 바꾸는 것이라, 예전 동작에 기대던 코드가 조용히 달라질 수 있다. 그래서 **`go 1.22` 이상을 선언한 모듈 안의 패키지에만** 새 동작을 적용한다. 모듈 단위로 천천히 옮겨가라는 뜻이다.

실무에서는 이런 상황이 생긴다.

- 로컬과 CI의 Go는 최신인데 `go.mod`는 몇 년 전 `go 1.19`로 남아 있다. 이러면 여전히 예전 동작이다.
- 반대로 `go.mod`만 `go 1.22`로 올렸다. 그러면 루프 변수의 주소(`&v`)를 모아서 "같은 변수"라고 가정하던 코드의 동작이 바뀐다.

### 5. go vet도 절반만 잡아준다

`go 1.21` 상태에서 `go vet`을 돌려봤다.

```
$ go vet .
./main.go:15:23: loop variable id captured by func literal
```

고루틴(`go func(){...}()`) 경우는 잡았다. 그런데 **클로저를 슬라이스에 모아두는 두 번째 경우는 아무 경고도 없었다.** vet의 `loopclosure` 검사는 루프 본문 마지막의 `go`, `defer` 문처럼 확실한 패턴만 본다. 오탐을 줄이려고 범위를 좁혀둔 것이다. "vet 통과했으니 안전하다"고 믿으면 안 된다.

### 6. 정리

- 새 루프 동작은 `go.mod`의 `go` 버전이 1.22 이상이어야 켜진다. 컴파일러만 올려선 안 바뀐다.
- `go.mod`를 올릴 때는 루프 변수의 주소나 클로저에 기대던 코드를 한 번 훑어보자. 공식 위키([LoopvarExperiment](https://go.dev/wiki/LoopvarExperiment))에는 버그가 어디서 생기는지 찾는 `bisect` 도구 사용법도 나와 있다.
- 예전 버전을 유지해야 하는 모듈이면 예전 관용구 `id := id`를 계속 쓰면 된다.

끝
