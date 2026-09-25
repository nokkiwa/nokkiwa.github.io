---
comments: true
layout: post
title: 에러 0건인데 9시간 동안 아무것도 못 하고 있던 감시 루프
date: 2026-10-05 09:00:00 +0900
category: project
---

개인적으로 돌리는 코인 자동매매 봇이 있다. 서버에서 상시로 돌고, 그중 하나가 **60초마다 보유 종목 가격을 받아 손절선 아래로 내려갔는지 확인하는 감시 루프**다.

어느 날 아침 로그를 보다가 이상한 걸 발견했다. 새벽 0시 55분부터 오전 9시 45분까지 약 9시간 동안 서버 네트워크가 끊겨 있었다. 그런데 감시 루프가 남긴 주기 통계는 멀쩡해 보였다.

```
cycle_start checked=27 errors=0
cycle_start checked=27 errors=0
cycle_start checked=27 errors=0
...
```

9시간 동안 가격을 한 번도 못 받았는데 `errors=0`이었다. 그 사이 가격이 급락했으면 손절이 전혀 안 걸렸을 것이다. 다행히 그날은 큰 움직임이 없어서 피해는 없었다.

### 1. 왜 에러가 0이었나

감시 루프 구조를 단순화하면 이렇다.

```python
# 단순화한 코드
def check_stops(positions):
    prices = fetch_prices(positions)   # 네트워크 실패 시 빈 dict
    checked = errors = 0
    for p in positions:
        checked += 1
        price = prices.get(p.market)
        if price is None:
            continue                   # 가격이 없으면 그냥 넘어감
        try:
            if price <= p.stop_price:
                sell(p)
        except Exception:
            errors += 1
    log.info(f"cycle_start checked={checked} errors={errors}")
```

문제는 세 가지가 겹친 데 있었다.

1. 가격 조회 함수가 네트워크 실패를 예외로 올리지 않고 **빈 결과**를 돌려줬다.
2. 가격이 없는 종목은 `continue`로 조용히 건너뛰었다.
3. `checked`는 **확인을 시도한 수**였지 **실제로 확인한 수**가 아니었다.

`errors`는 매도 주문 단계의 예외만 세고 있었다. 가격을 못 받은 건 에러로 치지 않았다. 그래서 통계만 보면 27개를 다 확인했고 에러도 없는 정상 상태처럼 보였다.

### 2. 수정

"건너뛴 것"을 따로 세도록 바꿨다.

```python
# 단순화한 코드
def check_stops(positions):
    prices = fetch_prices(positions)
    checked = errors = unpriced = 0
    for p in positions:
        price = prices.get(p.market)
        if price is None:
            unpriced += 1
            continue
        checked += 1
        ...
    level = logging.WARNING if unpriced else logging.INFO
    log.log(level, f"cycle_start checked={checked} unpriced={unpriced} errors={errors}")
```

- `unpriced`를 새로 두고, 가격을 못 받은 종목은 `checked`에 넣지 않았다.
- `unpriced`가 하나라도 있으면 로그 레벨을 `WARNING`으로 올렸다. INFO 사이에 묻히지 않고 경고로 눈에 띄게 하려는 것이다.
- 매일 보는 요약 리포트에도 `unpriced` 합계를 표시하게 했다. 로그를 한 줄씩 안 읽어도 "어젯밤 몇 번 못 봤는지"가 바로 보인다.

기존 테스트가 통과하는 걸 확인하고, 봇을 재시작해서 반영했다.

### 3. 배운 것

- **"에러 0"은 "정상"이 아니다.** 실패를 조용히 삼키는 경로가 있으면 에러 카운터는 아무것도 말해주지 않는다.
- 감시 로직이라면 **실제로 확인한 수**를 세야 한다. 시도한 수를 세면 안 된다. 기대값(보유 종목 27개)과 실제값이 다르면 그 자체가 신호다.
- `continue`로 건너뛰는 분기가 있으면 거기에도 카운터를 하나 달아두자. 제일 싸게 관측성을 얻는 방법이다.

이번에 고치면서 알게 된 건, 이 봇에는 프로세스가 죽거나 멈췄을 때 알려주는 장치가 따로 없다는 점이다. 그건 알고 있는 상태로 유지하기로 했다. 대신 적어도 "살아 있는데 눈을 감고 있는" 상태는 로그에서 바로 보이게 됐다.

끝
