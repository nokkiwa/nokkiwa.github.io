---
comments: true
layout: post
title: JPA N+1과 fetch join 페이징, 하이버네이트 7.4에서 달라진 점
date: 2026-10-17 09:00:00 +0900
category: error
---

JPA를 쓰면 N+1 문제는 꼭 한 번 만난다. 해결책으로 fetch join을 쓰고, fetch join에 페이징을 붙이면 경고가 뜨고, 그래서 `default_batch_fetch_size`를 쓴다는 흐름이 잘 알려져 있다.

그런데 하이버네이트 7.4부터 이 흐름의 가운데가 바뀌었다. 스프링 부트 4.1.1(하이버네이트 7.4.5)로 직접 쿼리 수를 세어봤다. 팀 5개에 멤버를 3명씩 넣고, 하이버네이트 통계(`generate_statistics`)로 실행된 쿼리 수를 셌다.

### 1. N+1 재현

```java
@Entity
public class Team {
    @Id @GeneratedValue Long id;
    String name;
    @OneToMany(mappedBy = "team") List<Member> members = new ArrayList<>();
}
```

```java
teams.findAll().forEach(t -> t.getMembers().size());
```

```sql
select t1_0.id,t1_0.name from team t1_0
select ... from member m1_0 where m1_0.team_id=?   -- 5번 반복
```

```
실행된 쿼리 수 = 6
```

팀 목록 1번에 팀마다 멤버 조회 1번씩, 합쳐서 1 + 5 = 6번이다. 팀이 1,000개면 1,001번이 된다.

### 2. fetch join

```java
@Query("select t from Team t join fetch t.members")
List<Team> findAllWithMembers();
```

```sql
select t1_0.id,m1_0.team_id,m1_0.id,m1_0.name,t1_0.name
from team t1_0 join member m1_0 on t1_0.id=m1_0.team_id
```

```
팀 수 = 5
실행된 쿼리 수 = 1
```

쿼리가 1번으로 줄었다. 조인 결과는 15행이지만 팀은 5개로 나왔다. 하이버네이트 6부터는 fetch join 결과의 중복 엔티티를 알아서 합쳐준다. 예전처럼 `distinct`를 붙이지 않아도 된다.

### 3. fetch join + 페이징: 7.4에서 바뀐 것

예전 하이버네이트는 컬렉션 fetch join에 페이징을 붙이면 이런 경고를 냈다.

```
HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

조인 결과는 "팀 × 멤버" 행이라, SQL에 `limit 2`를 걸면 팀 2개가 아니라 **행 2개**만 가져오게 된다. 그래서 하이버네이트는 전체를 다 읽어온 다음 메모리에서 잘랐다. 데이터가 많으면 메모리가 터지는 유명한 함정이었다.

7.4.5에서 같은 쿼리에 페이징을 걸어봤다.

```java
@Query(value = "select t from Team t join fetch t.members",
       countQuery = "select count(t) from Team t")
Page<Team> findPageWithMembers(Pageable pageable);

teams.findPageWithMembers(PageRequest.of(0, 2));
```

```sql
select t1_0.id,m1_0.team_id,m1_0.id,m1_0.name,t1_0.name
from (select t1_0.id,t1_0.name from team t1_0
      where exists(select 1 from member m1_0 where t1_0.id=m1_0.team_id)
      offset ? rows fetch first ? rows only) t1_0(id,name)
join member m1_0 on t1_0.id=m1_0.team_id
```

```
size=2 content=2 total=5 names=[team1, team2]
```

경고가 없다. **팀을 먼저 서브쿼리로 2개 자르고, 그 팀들의 멤버만 조인하는 SQL**로 바뀌었다. 페이징이 DB에서 제대로 걸린다. [하이버네이트 7.4 변경사항 문서](https://docs.hibernate.org/orm/7.4/whats-new/)에도 "limit/offset을 서브쿼리 안에서 지원하는 DB라면 이제 컬렉션 fetch join과 페이징을 안전하게 같이 써도 된다"고 나와 있다.

### 4. 그런데 같은 쿼리를 먼저 쓰면 페이징이 무시됐다

여기서 이상한 걸 하나 발견했다. 같은 JPQL 문자열을 **페이징 없이 먼저 한 번 실행하고**, 그다음에 페이징을 걸면 결과가 달라졌다.

```java
String q = "select t from Team t join fetch t.members";
em.createQuery(q, Team.class).getResultList();                       // 1)
em.createQuery(q, Team.class).setMaxResults(2).getResultList();      // 2)
em.createQuery(q + " ", Team.class).setMaxResults(2).getResultList(); // 3) 공백 하나 추가
```

```
1) 페이징 없이 = 5
2) setMaxResults(2) = 5     ← 2개를 요청했는데 5개
3) 다른 문자열(공백 추가) setMaxResults(2) = 2
```

2)에서 실행된 SQL을 보면 서브쿼리도 `limit`도 없는, 1)과 똑같은 SQL이었다. 경고도 예외도 없이 **요청한 것보다 많은 결과가 돌아왔다.** 쿼리 문자열에 공백 하나만 달라도(3) 정상이었다.

몇 가지를 더 확인했다.

| 조건 | 2)의 결과 |
|---|---|
| 하이버네이트 7.4.5 + H2 | 5 (문제) |
| 하이버네이트 7.4.10 + H2 | 5 (문제) |
| 하이버네이트 7.4.10 + MySQL 8.4 | 5 (문제) |
| 7.4.10 + `hibernate.query.plan_cache_enabled=false` | 2 (정상) |

쿼리 플랜 캐시를 끄면 정상이다. 그래서 **페이징 없이 만들어진 실행 계획이 캐시에 남아 있다가, 같은 문자열의 페이징 쿼리에 재사용되는 것**으로 보인다. 확인한 시점(2026년 9월)의 7.4 최신 패치에서도 그대로였다.

실무에서는 이런 식으로 터질 수 있다. 같은 `@Query` 문자열을 목록 전체 조회용 메서드와 페이징용 메서드에 복사해서 쓰는 경우다. 전체 조회가 먼저 한 번 불리면, 그 뒤의 페이징 API가 전체를 반환한다. 서버 재시작 직후 어느 쪽이 먼저 불리느냐에 따라 동작이 달라지니 재현도 어렵다.

당장의 대응은 이렇다.

- 페이징용과 비페이징용 쿼리 **문자열이 겹치지 않게** 한다.
- 또는 아래 5번의 `default_batch_fetch_size` 방식을 쓴다. 컬렉션 fetch join에 페이징을 거는 조합 자체를 피하는 것이다.

### 5. default_batch_fetch_size

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=100
```

fetch join 없이 일반 페이징을 하고, 컬렉션은 지연 로딩에 맡긴다.

```java
teams.findAll(PageRequest.of(0, 2)).forEach(t -> t.getMembers().size());
```

```sql
select t1_0.id,t1_0.name from team t1_0 offset ? rows fetch first ? rows only
select count(t1_0.id) from team t1_0
select ... from member m1_0 where m1_0.team_id in (?,?,?, ...)
```

```
실행된 쿼리 수 = 3
```

팀 페이지 1번, 카운트 1번, 멤버는 `IN` 절로 한 번에 가져왔다. 팀 수가 늘어도 쿼리 수는 거의 늘지 않는다(100개 단위로 1번씩). 페이징도 DB에서 걸리고 캐시 문제와도 상관없다. 여러 컬렉션을 한꺼번에 가져와야 할 때도 이 방식이 편하다. fetch join은 컬렉션 두 개를 동시에 걸 수 없기 때문이다.

### 6. 정리

- N+1은 지연 로딩 컬렉션을 루프에서 건드릴 때 생긴다. 통계로 쿼리 수를 세보면 바로 보인다.
- 하이버네이트 7.4부터는 컬렉션 fetch join + 페이징이 서브쿼리로 DB에서 처리된다. 메모리 페이징 경고는 옛날 이야기다.
- 단, 같은 쿼리 문자열을 페이징 없이 먼저 쓰면 페이징이 무시되는 동작을 확인했다(7.4.5, 7.4.10).
- 목록 + 컬렉션 조회에는 여전히 `default_batch_fetch_size`가 가장 무난하다.

끝
