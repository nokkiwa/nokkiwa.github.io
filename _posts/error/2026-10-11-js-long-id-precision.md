---
comments: true
layout: post
title: 백엔드 Long ID가 프론트에서 끝자리가 바뀌는 문제 (JSON 숫자 정밀도)
date: 2026-10-11 09:00:00 +0900
category: error
---

스프링 백엔드에서 `Long` 타입 ID를 JSON으로 내려주면 프론트에서 끝자리가 바뀌는 경우가 있다. 더 곤란한 건 그 바뀐 ID로 다시 요청을 보내면 "존재하지 않는 데이터"가 된다는 점이다. 에러도 안 나고 값만 조용히 달라져서 원인을 찾기 어렵다.

Node로 직접 재현해봤다.

### 1. 재현

```js
const body = '{"id":1834567890123456789,"orderNo":9007199254740993,"name":"주문"}';
const obj = JSON.parse(body);

console.log(obj.id);       // 1834567890123456800
console.log(obj.orderNo);  // 9007199254740992
console.log(JSON.stringify({ id: obj.id }));  // {"id":1834567890123456800}
```

- `1834567890123456789`가 `...6800`이 됐다.
- `9007199254740993`은 `...992`가 됐다. 1 차이라서 눈으로도 잘 안 보인다.
- 에러는 하나도 없다. 그리고 다시 서버로 보내면 **바뀐 값이 그대로 전송된다.**

### 2. 원인: 자바스크립트 숫자는 전부 double

자바의 `long`은 64비트 정수라 약 922경까지 정확하게 담는다. 자바스크립트의 `Number`는 64비트 **부동소수점**이다. 정수를 정확히 표현할 수 있는 건 가수부 53비트까지다.

```js
Number.MAX_SAFE_INTEGER            // 9007199254740991 (2^53 - 1)
9007199254740992 === 9007199254740993  // true
```

2^53을 넘어가면 인접한 정수끼리 구분하지 못한다. `JSON.parse`는 숫자를 무조건 `Number`로 바꾸기 때문에, 파싱하는 순간 정밀도가 사라진다.

### 3. 언제 이 문제를 만나나

`AUTO_INCREMENT` PK만 쓰면 평생 안 만날 가능성이 크다. 2^53은 약 9천조라 1씩 증가하는 ID가 거기까지 갈 일은 거의 없다.

문제는 **큰 값에서 시작하는 ID**다.

- 스노우플레이크 ID처럼 시간값을 상위 비트에 넣는 분산 ID. 이런 ID는 처음부터 18~19자리다.
- 외부 시스템(결제사, 메신저 API 등)이 주는 64비트 ID
- 해시 값을 `long`으로 잘라 쓴 키

DB나 ID 생성 방식을 바꾸는 순간 갑자기 터진다. 테스트 데이터는 작은 값이라 통과하고 운영에서만 깨지는 것도 흔하다.

### 4. 해결 1: 서버에서 문자열로 내려주기 (가장 확실)

프론트가 어떤 환경이든 안전하려면 ID를 JSON 문자열로 내려주는 게 제일 확실하다.

필드 단위로 적용할 때:

```java
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;

public class OrderResponse {
    @JsonSerialize(using = ToStringSerializer.class)
    private Long id;
    private String name;
}
```

```json
{"id":"1834567890123456789","name":"주문"}
```

전체 `Long`에 일괄 적용할 수도 있지만 그러면 `count`, `price`처럼 숫자로 써야 하는 필드까지 문자열이 된다. **식별자 필드에만** 붙이는 편이 낫다.

받는 쪽도 신경 써야 한다. 프론트가 `"id":"123"`을 다시 보내면 Jackson은 문자열 `"123"`을 `Long`으로 문제없이 역직렬화해준다. 요청 DTO는 그대로 둬도 된다.

### 5. 해결 2: 프론트에서 원문 그대로 읽기

서버를 못 바꾸는 상황이면 프론트에서 처리해야 한다. 최근에는 `JSON.parse`의 reviver 함수가 **원본 텍스트**를 받을 수 있게 됐다(TC39 "JSON.parse source text access").

```js
const safe = JSON.parse(body, (key, value, ctx) =>
  typeof value === 'number' && !Number.isSafeInteger(value) && ctx?.source
    ? BigInt(ctx.source)
    : value
);
console.log(safe.id, typeof safe.id);  // 1834567890123456789n 'bigint'
```

직접 돌려보니 버전에 따라 결과가 갈렸다.

| 환경 | 결과 |
|---|---|
| Node 20.19 | `ctx` 없음 → 여전히 `1834567890123456800` |
| Node 22.23 | `1834567890123456789n` (정확) |

브라우저는 [Can I use](https://caniuse.com/mdn-javascript_builtins_json_parse_reviver_parameter_context_argument) 기준으로 Chrome·Edge 114, Firefox 135, Safari 18.4부터 지원한다. 구형 브라우저를 신경 써야 하면 `json-bigint` 같은 라이브러리를 쓰거나 서버에서 문자열로 주는 방법(4번)으로 가야 한다.

`BigInt`는 `JSON.stringify`가 기본으로 처리하지 못해서(`TypeError`), 다시 보낼 때 문자열로 바꾸는 처리도 따로 필요하다. 이 점까지 생각하면 결국 4번이 가장 단순하다.

### 6. 정리

- 자바스크립트는 2^53(약 9천조)을 넘는 정수를 정확히 못 다룬다. `JSON.parse`는 에러 없이 값을 반올림한다.
- 스노우플레이크 같은 큰 ID를 쓰면 반드시 만난다.
- 식별자는 서버에서 문자열로 내려주자(`@JsonSerialize(using = ToStringSerializer.class)`).
- 의심되면 `Number.isSafeInteger(id)`로 바로 확인할 수 있다.

참고: [MDN - Number.MAX_SAFE_INTEGER](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER), [MDN - JSON.parse()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse)

끝
