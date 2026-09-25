---
comments: true
layout: post
title: open-in-view 경고를 끄자 LazyInitializationException이 터진 이유
date: 2026-10-23 09:00:00 +0900
category: error
---

스프링 부트로 JPA 프로젝트를 띄우면 시작할 때마다 이 경고가 찍힌다.

```
WARN JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default.
Therefore, database queries may be performed during view rendering.
Explicitly configure spring.jpa.open-in-view to disable this warning
```

경고가 거슬려서 `spring.jpa.open-in-view=false`를 넣으면, 멀쩡하던 API가 갑자기 500을 뱉기 시작한다. 스프링 부트 4.1.1에서 재현해봤다.

### 1. 재현 코드

컨트롤러에서 엔티티를 조회하고 지연 로딩 컬렉션을 건드리는 코드다. 서비스 계층 없이 바로 쓰는 경우가 생각보다 많다.

```java
@RestController
class TeamController {
    private final TeamRepository teams;

    @GetMapping("/teams/{id}/member-count")
    int memberCount(@PathVariable Long id) {
        Team team = teams.findById(id).orElseThrow();
        return team.getMembers().size();   // 지연 로딩
    }
}
```

### 2. 결과

**open-in-view 기본값(true)**

```
select t1_0.id,t1_0.name from team t1_0 where t1_0.id=?
select m1_0.team_id,m1_0.id,m1_0.name from member m1_0 where m1_0.team_id=?
status=200 body=2
```

**open-in-view=false**

```
select t1_0.id,t1_0.name from team t1_0 where t1_0.id=?
org.hibernate.LazyInitializationException: Cannot lazily initialize collection of role
'demo.Team.members' with key '1' (no session)
status=500
```

설정 한 줄 차이로 같은 코드가 200과 500으로 갈렸다.

### 3. 원인: 영속성 컨텍스트가 언제 닫히나

**open-in-view(OSIV)가 켜져 있으면** 스프링이 요청이 들어올 때 영속성 컨텍스트(EntityManager)를 열고, 응답이 나갈 때까지 유지한다. 그래서 컨트롤러는 물론 JSON 직렬화 중에도 지연 로딩이 된다.

**꺼져 있으면** 영속성 컨텍스트는 트랜잭션 범위에서만 산다. `teams.findById()`는 리포지토리 메서드의 트랜잭션 안에서 실행되고, 끝나면 컨텍스트가 닫힌다. 그 뒤에 `getMembers().size()`로 지연 로딩을 시도하면 쿼리를 날릴 세션이 없다. 그래서 `(no session)`이 뜬다.

### 4. 그럼 켜두면 되나?

경고가 괜히 있는 게 아니다. OSIV가 켜져 있으면 **요청이 끝날 때까지 DB 커넥션을 쥐고 있는다.** 컨트롤러에서 외부 API를 부르느라 2초를 기다리면, 그동안 커넥션도 같이 묶인다. 트래픽이 몰리면 커넥션 풀이 금방 바닥난다.

그래서 보통은 이렇게 나눈다.

- **관리자 화면이나 트래픽이 적은 서비스**: 켜둬도 큰 문제 없다. 경고만 끄고 싶으면 `spring.jpa.open-in-view=true`를 명시하면 된다.
- **트래픽이 많은 API 서버**: 끄고, 필요한 데이터는 트랜잭션 안에서 다 불러온다.

### 5. 끄고 나서 고치는 방법

지연 로딩이 필요한 조회를 **서비스 계층의 트랜잭션 안으로** 옮기고, 엔티티 대신 DTO를 반환한다.

```java
@Service
class TeamQueryService {
    private final TeamRepository teams;

    @Transactional(readOnly = true)
    public int memberCount(Long id) {
        return teams.findById(id).orElseThrow().getMembers().size();
    }
}
```

화면에 컬렉션 내용이 필요하면 fetch join이나 `@EntityGraph`로 **한 번에 불러온 뒤** DTO로 바꿔서 반환한다. 이렇게 하면 트랜잭션이 끝난 뒤에 엔티티를 건드릴 일이 없다.

엔티티를 그대로 JSON으로 반환하는 코드가 많다면 OSIV를 끄는 순간 여기저기서 터진다. 끄기 전에 컨트롤러에서 엔티티를 반환하는 곳부터 찾아보자.

### 6. 정리

- OSIV 경고는 "요청 내내 DB 커넥션을 쥔다"는 뜻이다.
- 끄면 트랜잭션 밖에서의 지연 로딩이 `LazyInitializationException (no session)`으로 실패한다.
- 끌 거라면 지연 로딩을 서비스 트랜잭션 안으로 옮기고, 컨트롤러에는 DTO만 넘기자.

참고: [Spring Boot 문서 - Open EntityManager in View](https://docs.spring.io/spring-boot/reference/data/sql.html#data.sql.jpa-and-spring-data.open-entity-manager-in-view)

끝
