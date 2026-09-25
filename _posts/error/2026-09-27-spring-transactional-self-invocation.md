---
comments: true
layout: post
title: 스프링 @Transactional이 같은 클래스 안에서 호출하면 안 먹는 이유
date: 2026-09-27 09:00:00 +0900
category: error
---

`@Transactional`을 붙였는데 예외가 나도 데이터가 롤백되지 않는 경우가 있다. 가장 흔한 원인은 **같은 클래스 안에서 메서드를 호출한 것**이다. 스프링 부트 4.1.1 + 하이버네이트 7.4 환경에서 직접 재현해봤다.

### 1. 재현 코드

```java
@Service
class SignupService {
    private final MemberRepository repo;

    public void signup(String name) {   // 트랜잭션 없음
        saveMember(name);               // 같은 클래스 내부 호출
    }

    @Transactional
    public void saveMember(String name) {
        System.out.println("트랜잭션 활성? " +
            TransactionSynchronizationManager.isActualTransactionActive());
        repo.save(new Member(name, null));
        throw new IllegalStateException("저장 후 실패");
    }
}
```

`saveMember`는 저장한 뒤 예외를 던진다. `@Transactional`이 제대로 동작하면 저장이 롤백돼서 남는 데이터가 없어야 한다.

### 2. 결과

```
트랜잭션 활성? false
예외: 저장 후 실패
내부 호출 후 member 수 = 1
```

`@Transactional`을 붙였는데 **트랜잭션이 아예 시작되지 않았다.** 그래서 롤백할 트랜잭션도 없었고, `repo.save()`가 자기 트랜잭션으로 커밋해버려서 데이터가 1건 남았다.

### 3. 원인: 프록시를 거치지 않았다

스프링은 `@Transactional`이 붙은 빈을 **프록시 객체로 감싸서** 등록한다. 다른 빈이 `signupService.saveMember()`를 부르면 실제로는 프록시가 먼저 호출된다. 프록시가 트랜잭션을 열고, 진짜 객체의 메서드를 부르고, 끝나면 커밋하거나 롤백한다.

```
컨트롤러 → [프록시: 트랜잭션 시작] → SignupService.signup() → this.saveMember()
                                                              ↑ 프록시를 안 거침
```

그런데 `signup()` 안에서 `saveMember()`를 부르는 건 `this.saveMember()`다. 이미 진짜 객체 안에 들어와 있으니 프록시를 거치지 않는다. 그래서 `@Transactional`이 무시된다. `private` 메서드에 붙인 `@Transactional`이 안 먹는 것도 같은 이유다.

### 4. 해결: 다른 빈으로 분리

트랜잭션이 필요한 메서드를 별도 빈으로 옮기고 주입받아 호출했다.

```java
@Service
class SignupWorker {
    @Transactional
    public void saveMember(String name) {
        repo.save(new Member(name, null));
        throw new IllegalStateException("저장 후 실패");
    }
}

@Service
class SignupService {
    private final SignupWorker worker;
    public void signupFixed(String name) {
        worker.saveMember(name);   // 프록시를 거친다
    }
}
```

```
트랜잭션 활성? true
예외: 저장 후 실패
다른 빈 호출 후 member 수 = 0
```

이번엔 트랜잭션이 열렸고 예외와 함께 롤백됐다.

### 5. 다른 방법들

- **바깥 메서드(`signup`)에 `@Transactional`을 붙인다.** 내부 호출된 메서드도 바깥 트랜잭션 안에서 돈다. 다만 안쪽 메서드에 `REQUIRES_NEW` 같은 전파 옵션을 붙여놨다면 여전히 무시된다.
- **자기 자신을 주입받는 방법(self-injection)**도 있다. 동작은 하지만 순환 참조처럼 보이고 읽기 어려워서 추천하지 않는다.

### 6. 정리

- `@Transactional`은 **외부에서 프록시를 통해 호출될 때만** 동작한다.
- 같은 클래스 안의 호출, `private` 메서드에는 적용되지 않는다. 에러도 경고도 없다.
- 의심되면 메서드 안에서 `TransactionSynchronizationManager.isActualTransactionActive()`를 찍어보자. 바로 확인된다.

참고: [Spring Framework 문서 - Using @Transactional](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

끝
