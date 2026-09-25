---
comments: true
layout: post
title: 파이썬 모듈을 import만 했는데 운영 로그 파일이 회전된 문제
date: 2026-10-19 09:00:00 +0900
category: error
---

매일 크론으로 도는 파이썬 스크립트가 있다. 이 스크립트가 만든 결과물을 점검하는 별도 스크립트를 새로 만들고 있었다. 결과물을 파싱하는 함수가 이미 본 스크립트 안에 있어서, 그걸 가져다 쓰려고 이렇게 시작했다.

```python
# check.py
from main_script import parse_output
```

점검 스크립트를 몇 번 돌려보고 나서 본 스크립트의 로그를 확인하러 갔다. 그런데 로그 파일이 방금 비워진 새 파일로 바뀌어 있었다. 그 전 내용은 날짜가 붙은 파일로 옮겨져 있었다.

### 1. 원인: 모듈 최상단의 코드

본 스크립트의 맨 위는 대략 이런 구조였다.

```python
# main_script.py (단순화)
import logging, os, shutil, datetime

LOG = "blog.txt"
if os.path.exists(LOG) and os.path.getsize(LOG) > 0:
    stamp = datetime.date.today().isoformat()
    shutil.move(LOG, f"logs/{stamp}.txt")        # 로그 회전
logging.basicConfig(filename=LOG, level=logging.INFO)

def parse_output(path):
    ...

def main():
    ...

if __name__ == "__main__":
    main()
```

`main()` 호출은 `if __name__ == "__main__":` 안에 잘 들어가 있었다. 그런데 **로그 회전 코드는 모듈 최상단**에 있었다.

파이썬에서 `import`는 그 파일을 위에서부터 끝까지 실행하는 것이다. 함수 정의(`def`)는 함수 객체를 만들기만 하지만, 최상단의 일반 코드는 import하는 순간 그대로 실행된다. 그래서 `from main_script import parse_output` 한 줄 때문에 로그 회전이 돌고 `basicConfig`까지 실행됐다.

단독 실행할 때는 "시작할 때마다 로그를 새로 연다"는 의도대로 동작하니까 한 번도 문제가 된 적이 없었다.

### 2. 피해 범위

- 실행 중인 로그가 중간에 잘려서, 그날 로그가 날짜 파일 두 개로 나뉘었다.
- 로그 파일 소유자가 root였는데 점검 스크립트는 일반 사용자로 돌려서, 환경에 따라서는 권한 에러로 import 자체가 실패할 수도 있었다.
- `basicConfig`가 점검 스크립트의 로깅 설정까지 가져가서, 점검 스크립트의 로그가 본 스크립트 로그 파일에 섞일 수 있었다.

### 3. 해결

선택지는 두 가지였다.

**① 본 스크립트를 고친다.** 부수효과가 있는 코드를 함수로 옮기고 `main()`에서 부른다.

```python
def setup_logging():
    if os.path.exists(LOG) and os.path.getsize(LOG) > 0:
        ...
    logging.basicConfig(filename=LOG, level=logging.INFO)

if __name__ == "__main__":
    setup_logging()
    main()
```

**② 점검 스크립트가 import하지 않는다.** 필요한 파싱 로직만 점검 스크립트 안에 따로 구현한다.

원칙적으로는 ①이 맞다. 하지만 본 스크립트는 매일 운영 중이고 다른 작업도 같이 손대고 있는 파일이었다. 점검 도구 하나 때문에 운영 스크립트의 시작 순서를 바꾸는 건 부담이 컸다. 그래서 이번에는 ②를 골랐다. 점검 스크립트 맨 위에 "본 스크립트를 import하면 로그가 회전되므로 독립적으로 파싱한다"는 주석을 남겼다.

### 4. 정리

- `import`는 "함수만 가져오기"가 아니다. 모듈 최상단 코드가 전부 실행된다.
- 파일 이동, 로그 설정, DB 연결, 네트워크 호출처럼 **부수효과가 있는 코드는 최상단에 두지 말자.** 함수로 감싸고 `if __name__ == "__main__":` 아래에서 부르자.
- 이미 운영 중인 스크립트라서 못 고친다면, 적어도 그 이유를 가져다 쓰는 쪽에 남겨두자.

직접 확인해보고 싶다면 이렇게 하면 된다.

```bash
python -c "import main_script"   # 이 한 줄로 무슨 일이 일어나는지 본다
```

아무 일도 안 일어나야 정상이다.

끝
