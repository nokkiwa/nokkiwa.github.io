---
comments: true
layout: post
title: 스프링 @Transactional, checked 예외는 롤백되지 않는다
date: 2026-10-13 09:00:00 +0900
category: error
---

외부 API 호출이 실패해서 예외가 났는데 DB에는 데이터가 저장돼 있는 경우가 있다. `@Transactional`도 붙어 있고 내부 호출 문제도 아니다. 이럴 때는 **던진 예외의 종류**를 봐야 한다.

스프링 부트 4.1.1 환경에서 재현해봤다.

### 1. 재현

```java
@Service
class CheckedService {
    @Transactional
    public void saveThenChecked(String name) throws Exception {
        repo.save(new Member(name, null));
        throw new Exception("외부 API 실패(checked)");
    }
}
```

```
예외: 외부 API 실패(checked)
checked 예외 후 member 수 = 1
```

예외가 호출한 쪽까지 올라왔는데도 **데이터가 커밋됐다.**

### 2. 원인: 스프링의 기본 롤백 규칙

스프링의 기본 규칙은 이렇다.

| 예외 종류 | 기본 동작 |
|---|---|
| `RuntimeException`과 그 하위(unchecked) | 롤백 |
| `Error` | 롤백 |
| `Exception`과 그 하위 중 RuntimeException이 아닌 것(checked) | **커밋** |

EJB 시절부터 이어진 관례다. checked 예외는 "비즈니스적으로 예상된 결과"라서 호출한 쪽이 처리할 수 있다고 보는 것이다. 그런데 실무에서는 `IOException`, 직접 만든 `extends Exception` 예외처럼 **실패를 뜻하는 checked 예외**가 많아서 자주 문제가 된다.

### 3. 해결 1: rollbackFor 지정

```java
@Transactional(rollbackFor = Exception.class)
public void saveThenCheckedFixed(String name) throws Exception {
    repo.save(new Member(name, null));
    throw new Exception("외부 API 실패(checked)");
}
```

```
예외: 외부 API 실패(checked)
rollbackFor 적용 후 member 수 = 0
```

롤백됐다.

### 4. 해결 2: unchecked 예외로 감싸기

예외를 직접 정의한다면 처음부터 `RuntimeException`을 상속하면 된다.

```java
public class ExternalApiException extends RuntimeException {
    public ExternalApiException(String message, Throwable cause) { super(message, cause); }
}

try {
    client.call();
} catch (IOException e) {
    throw new ExternalApiException("외부 API 실패", e);
}
```

스프링 진영에서는 이 방식을 더 많이 쓴다. `rollbackFor`는 메서드마다 붙여야 해서 하나만 빠뜨려도 같은 문제가 생긴다.

### 5. 주의: 예외를 잡아버리면 롤백 안 된다

비슷해 보이지만 다른 경우가 하나 더 있다.

```java
@Transactional
public void save(String name) {
    repo.save(new Member(name, null));
    try {
        client.call();
    } catch (Exception e) {
        log.error("실패", e);   // 잡고 끝
    }
}
```

예외가 `@Transactional` 메서드 밖으로 안 나가면, 스프링 입장에선 정상 종료다. 그러면 커밋된다. 롤백이 필요하면 다시 던지거나 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`를 호출해야 한다.

### 6. 정리

- `@Transactional`의 기본 롤백 대상은 **unchecked 예외와 Error뿐**이다.
- checked 예외로 롤백하려면 `rollbackFor`를 쓰거나 unchecked 예외로 감싸자.
- 예외를 메서드 안에서 잡아버리면 예외 종류와 상관없이 커밋된다.

참고: [Spring Framework 문서 - Rolling Back a Declarative Transaction](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)

끝
