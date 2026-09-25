---
comments: true
layout: post
title: HikariPool Connection is not available, REQUIRES_NEW가 커넥션 풀을 막는 경우
date: 2026-10-07 09:00:00 +0900
category: error
---

트래픽이 몰리는 순간 이런 에러가 쏟아지는 경우가 있다.

```
HikariPool-1 - Connection is not available, request timed out after 30000ms
(total=10, active=10, idle=0, waiting=...)
```

DB는 한가한데 커넥션 풀만 꽉 찼다. 원인은 여러 가지지만, 그중 코드만 보고는 잘 안 보이는 게 **`REQUIRES_NEW`로 인한 풀 교착**이다. 스프링 부트 4.1.1(HikariCP 7.0.2) 환경에서 재현해봤다.

### 1. 재현 코드

주문을 저장하면서 감사 로그는 별도 트랜잭션으로 남기는 흔한 구조다. 주문이 롤백돼도 로그는 남기려고 `REQUIRES_NEW`를 썼다.

```java
@Service
class OrderService {
    @Transactional
    public void order(String name) {
        repo.save(new Member(name, null));
        audit.write(name);                 // REQUIRES_NEW
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void write(String name) {
        repo.save(new Member("audit-" + name, null));
    }
}
```

빨리 확인하려고 풀 크기와 대기 시간을 줄였다.

```properties
spring.datasource.hikari.maximum-pool-size=2
spring.datasource.hikari.connection-timeout=3000
```

### 2. 결과

동시 요청 수를 바꿔가며 호출했다.

```
pool=2, 동시요청 1개 -> [OK] (136ms)
pool=2, 동시요청 2개 -> [FAIL, FAIL] (3037ms)
pool=2, 동시요청 4개 -> [FAIL, FAIL, FAIL, FAIL] (3034ms)
```

실패한 요청의 원인은 전부 이것이었다.

```
java.sql.SQLTransientConnectionException: HikariPool-5 - Connection is not available,
request timed out after 3001ms (total=2, active=2, idle=0, waiting=1)
```

요청 1개일 때는 멀쩡했다. 그런데 **동시 요청 수가 풀 크기와 같아지는 순간 전부 실패했다.** 하나도 성공하지 못했다.

### 3. 원인: 커넥션을 쥔 채로 커넥션을 기다린다

`REQUIRES_NEW`는 바깥 트랜잭션을 잠시 멈추고 **새 커넥션으로** 새 트랜잭션을 연다. 바깥 트랜잭션의 커넥션은 반납되지 않고 그대로 쥐고 있다. 그러니 요청 하나가 커넥션을 **2개** 써야 끝난다.

풀 크기가 2이고 요청이 2개 동시에 들어오면 이렇게 된다.

```
요청 A: 커넥션 1 획득 → order() 진행 → write()에서 커넥션 하나 더 요청 → 대기
요청 B: 커넥션 2 획득 → order() 진행 → write()에서 커넥션 하나 더 요청 → 대기
풀: 남은 커넥션 0. 둘 다 상대가 반납하기를 기다림 → 타임아웃까지 교착
```

두 요청 모두 자기 커넥션을 쥔 채 서로를 기다린다. `connection-timeout`이 지나서야 둘 다 실패한다. 기본값 30초 동안 서버가 멈춘 것처럼 보인다.

운영에서는 풀이 10이면 동시 요청 10개가 딱 겹칠 때 터진다. 평소엔 멀쩡하다가 트래픽이 몰리는 순간에만 재현되니 원인 찾기가 어렵다.

### 4. 해결

1. **REQUIRES_NEW를 꼭 써야 하는지 다시 본다.** 감사 로그처럼 "실패해도 남아야 하는" 작업이면, 바깥 트랜잭션이 끝난 뒤에 처리하는 게 낫다. `@TransactionalEventListener(phase = AFTER_COMPLETION)`나 비동기 큐가 대안이다.
2. **풀 크기를 늘린다.** 동시 스레드 수 × (요청 하나가 동시에 쥐는 커넥션 수)보다 풀이 커야 교착이 안 생긴다. 톰캣 스레드가 200인데 풀이 10이면 계산이 안 맞는다. HikariCP 위키에 교착을 피하는 최소 크기 공식이 나온다.
   ```
   pool size = Tn × (Cm - 1) + 1
   Tn: 최대 스레드 수, Cm: 스레드 하나가 동시에 쥐는 최대 커넥션 수
   ```
3. **REQUIRES_NEW 작업은 바깥 트랜잭션 전에 끝낸다.** 바깥 트랜잭션이 커넥션을 쥐기 전에 호출하면 커넥션을 동시에 2개 쥘 일이 없다.

### 5. 정리

- `REQUIRES_NEW`는 커넥션을 하나 더 쓴다. 바깥 커넥션은 그동안 반납되지 않는다.
- 동시 요청 수가 풀 크기에 닿으면 **전원이 교착**에 빠지고, `connection-timeout` 뒤에 한꺼번에 실패한다.
- 에러 메시지의 `active=풀 크기, idle=0`인데 DB 쪽은 한가하다면 이 패턴을 의심해보자.

참고: [HikariCP Wiki - About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

끝
