---
comments: true
layout: post
title: MySQL 이모지 저장 에러 Incorrect string value, 테이블만 바꾸면 안 되는 이유
date: 2026-10-03 09:00:00 +0900
category: error
---

게시글 제목에 이모지가 들어가자 저장이 실패했다.

```
ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x9A\x80' for column 'title' at row 1
```

원인은 유명하다. MySQL의 `utf8`은 진짜 UTF-8이 아니다. 그런데 "테이블을 utf8mb4로 바꾸면 끝"이라는 설명만 보고 고쳤다가 같은 에러를 또 볼 수 있다. MySQL 8.4 도커 컨테이너로 단계별로 재현해봤다.

### 1. 재현

```sql
CREATE TABLE post_utf8 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(50)
) CHARACTER SET utf8;
```

```
Query OK, 0 rows affected, 1 warning
Warning 3719: 'utf8' is currently an alias for the character set UTF8MB3,
but will be an alias for UTF8MB4 in a future release.
```

테이블을 만들 때부터 경고가 나온다. `SHOW CREATE TABLE`로 보면 실제로는 `utf8mb3`로 만들어져 있다.

```sql
INSERT INTO post_utf8 (title) VALUES ('배포 완료 🚀');
-- ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x9A\x80' for column 'title'
```

### 2. 왜 이모지만 안 되나

```sql
SELECT CHAR_LENGTH('🚀'), LENGTH('🚀'), HEX('🚀'), LENGTH('가');
-- 1 | 4 | F09F9A80 | 3
```

한글 '가'는 3바이트, 🚀는 **4바이트**다. MySQL의 `utf8mb3`는 이름 그대로 글자당 최대 3바이트까지만 저장한다. 한글·한자·대부분의 기호는 3바이트 안에 들어가니 평소엔 문제가 없다. 그러다 이모지처럼 4바이트 글자가 처음 들어오는 날 터진다. 에러 메시지의 `\xF0\x9F\x9A\x80`이 바로 그 4바이트다.

### 3. 테이블을 utf8mb4로 바꾸기

```sql
ALTER TABLE post_utf8 CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
INSERT INTO post_utf8 (title) VALUES ('배포 완료 🚀');   -- 성공
```

바꾸기 전후 컬럼 정보를 보면 차이가 보인다.

```sql
SELECT CHARACTER_MAXIMUM_LENGTH, CHARACTER_OCTET_LENGTH
FROM information_schema.COLUMNS WHERE TABLE_NAME='post_utf8' AND COLUMN_NAME='title';
-- 변경 전: 50 | 150
-- 변경 후: 50 | 200
```

`VARCHAR(50)`은 글자 수 기준이라 50글자는 그대로다. 대신 최대 바이트가 150에서 200으로 늘었다. 긴 `VARCHAR`에 인덱스가 걸려 있으면 인덱스 길이 제한에 걸릴 수 있으니 변환 전에 확인하자.

`CONVERT TO`는 테이블 전체를 다시 쓴다. 큰 테이블이면 운영 중에 바로 돌리지 말고 시간대를 잡자.

### 4. 테이블을 바꿨는데 또 같은 에러

테이블은 이제 utf8mb4다. 이번엔 클라이언트 연결 문자셋을 `utf8mb3`로 두고 넣어봤다.

```sql
SHOW VARIABLES LIKE 'character_set_c%';
-- character_set_client     | utf8mb3
-- character_set_connection | utf8mb3

INSERT INTO post_utf8 (title) VALUES ('연결 문자셋 🚀');
-- ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x9A\x80' for column 'title' at row 1
```

**컬럼이 utf8mb4여도 연결이 utf8mb3면 똑같은 에러가 난다.** 에러 메시지가 컬럼 이름을 가리키니 테이블 문제라고 착각하기 쉽다. 결국 세 군데를 다 맞춰야 한다.

1. 컬럼·테이블·DB 문자셋: `utf8mb4`
2. 서버 기본값: `character-set-server=utf8mb4` (MySQL 8.0부터 기본값이 이미 utf8mb4)
3. **연결 문자셋**: 애플리케이션 드라이버 설정

스프링부트 + MySQL Connector/J라면 JDBC URL에 넣는다.

```
jdbc:mysql://localhost:3306/db?characterEncoding=UTF-8
```

[Connector/J 문서](https://dev.mysql.com/doc/connector-j/en/connector-j-reference-charsets.html)에 따르면 `characterEncoding=UTF-8`은 MySQL의 `utf8mb4`로 매핑된다. 다만 `connectionCollation=utf8_general_ci`처럼 **utf8mb3 콜레이션을 같이 적어두면 그쪽이 이긴다.** 예전 설정을 복사해온 프로젝트라면 URL에 이런 옵션이 남아 있지 않은지 확인하자.

### 5. 더 위험한 경우: 에러가 안 나는 것

`sql_mode`에서 strict 모드를 끄고 넣어봤다.

```sql
SET sql_mode='';
CREATE TABLE post_loose (title VARCHAR(50)) CHARACTER SET utf8mb3;
INSERT INTO post_loose VALUES ('배포 완료 🚀');
-- Query OK, 1 warning
SELECT title FROM post_loose;
-- 배포 완료 ?
```

에러 없이 **이모지가 `?`로 바뀌어 저장됐다.** 경고 한 줄만 남기고 데이터가 조용히 깨진다. MySQL 8의 기본 `sql_mode`에는 `STRICT_TRANS_TABLES`가 들어 있어서 에러가 난다. 하지만 오래된 서버나 설정 파일에서 `sql_mode`를 비워둔 곳이라면 이 상태일 수 있다. 에러가 나는 쪽이 차라리 낫다.

### 6. 정리

- MySQL의 `utf8`은 `utf8mb3`다. 새로 만드는 건 무조건 `utf8mb4`로 쓰자. 공식 문서도 `utf8mb3`를 deprecated로 두고 있다([MySQL 문서](https://dev.mysql.com/doc/refman/8.4/en/charset-unicode-utf8mb3.html)).
- 테이블만 바꾸고 끝내지 말고 **연결 문자셋**까지 확인하자.
- strict 모드가 꺼져 있으면 에러 대신 `?`로 저장된다. `SELECT @@sql_mode;`를 한 번 확인해두자.

끝
