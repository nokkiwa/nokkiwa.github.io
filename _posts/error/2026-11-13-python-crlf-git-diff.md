---
comments: true
layout: post
title: 파이썬으로 CSS 37줄 추가했는데 git diff가 2,000줄, CRLF 파일과 newline 옵션
date: 2026-11-13 09:00:00 +0900
category: error
---

블로그 CSS 파일에 스타일 몇 개를 추가하는 작업이었다. 앵커가 되는 블록을 찾아 그 뒤에 새 규칙을 붙이는 간단한 파이썬 스크립트로 처리했다. 실제로 추가한 건 37줄인데 `git diff`를 보니 이랬다.

```
$ git diff --numstat
1114    1077    public/css/style.css
```

1,114줄 추가, 1,077줄 삭제. 파일 전체가 바뀐 것처럼 나온다.

### 1. 원인: 원래 파일이 CRLF였다

```bash
grep -c $'\r$' public/css/style.css     # 수정 전 원본
# 1077
```

이 파일은 1,167줄 중 1,077줄의 끝이 `\r\n`(CRLF)이었다. 그리고 파이썬은 텍스트 모드로 파일을 열면 이걸 조용히 바꾼다.

- **읽을 때**: `\r\n`을 `\n`으로 바꿔서 준다 (universal newlines).
- **쓸 때**: `\n`을 운영체제 기본 줄바꿈으로 쓴다. 리눅스에서는 `\n` 그대로다.

결국 읽고 쓰기만 해도 모든 줄 끝의 `\r`이 사라진다. git 입장에선 모든 줄이 바뀐 것이다.

### 2. 작게 재현

CRLF로 된 CSS 규칙 100개짜리 파일을 만들고 규칙 하나(3줄)를 추가해봤다. 파이썬 3.10이다.

```python
s = open("style.css", encoding="utf-8").read()
print(s.count("\r"))     # 0  ← 원래 300개 있던 \r이 읽는 순간 사라짐
s = s.replace(".a5 {\n  color: red;\n}", ".a5 {\n  color: red;\n}\n.new {\n  color: blue;\n}")
open("style.css", "w", encoding="utf-8").write(s)
```

```
$ git diff --numstat
303     300     style.css
```

3줄 추가가 303줄 추가, 300줄 삭제로 나온다. 원래 파일이 300줄이니 전부 갈아엎은 셈이다.

### 3. 해결: newline=''

`open()`에 `newline=''`을 주면 읽을 때도 쓸 때도 줄바꿈을 건드리지 않는다.

```python
s = open("style.css", encoding="utf-8", newline="").read()
print(s.count("\r"))     # 300  ← 그대로 남아 있음
s = s.replace(".a5 {\r\n  color: red;\r\n}", ".a5 {\r\n  color: red;\r\n}\r\n.new {\r\n  color: blue;\r\n}")
open("style.css", "w", encoding="utf-8", newline="").write(s)
```

```
$ git diff --numstat
3       0       style.css
```

이제 진짜 바뀐 3줄만 나온다. 대신 찾고 바꾸는 문자열에도 `\r\n`을 직접 넣어야 한다. 안 그러면 `replace`가 아무것도 못 찾는다.

실제 블로그 CSS도 바이트 단위로 읽고 `\r\n`으로 이어 붙이는 방식으로 다시 적용했고, diff는 `37 0`이 됐다. 바이너리 모드(`"rb"`/`"wb"`)로 다뤄도 결과는 같다.

### 4. 저질렀는지 확인하는 법

내가 바꾼 게 진짜 내용인지 줄바꿈인지 헷갈리면 줄 끝의 CR을 무시하고 diff를 보면 된다.

```bash
git diff --ignore-cr-at-eol --numstat
# 3   0   style.css   ← 줄바꿈 말고 실제로 바뀐 건 이것뿐
```

원래대로 되돌리려면 `git checkout -- 파일`로 되돌린 뒤 `newline=''`로 다시 작업하면 된다.

### 5. 근본적으로는

한 저장소 안에 CRLF와 LF 파일이 섞여 있는 게 문제의 출발점이다. `git ls-files --eol`로 저장소 전체를 확인할 수 있다.

```bash
git ls-files --eol | grep -E 'w/(crlf|mixed)'
```

이 블로그는 레이아웃·include·CSS·JS와 `Gemfile` 등 40개 넘는 파일이 CRLF였고, 이번에 고친 `style.css`는 아예 한 파일 안에 CRLF 줄과 LF 줄이 섞여 있었다(`w/mixed`). 예전에 누가 어떤 도구로 고쳤는지에 따라 줄마다 달라진 것이다.

정리하고 싶다면 `.gitattributes`에 규칙을 두고 한 번에 맞추는 방법이 있다.

```
* text=auto eol=lf
```

다만 이걸 넣고 정규화하면 그 커밋에서 CRLF 파일 전체가 한 번 바뀐다. 남이 같이 쓰는 저장소라면 미리 알리고 하는 게 좋다. 나는 지금은 손대지 않고, 이 파일들을 고칠 때만 줄바꿈을 지키는 쪽을 택했다.

### 정리

- 파이썬 텍스트 모드는 읽을 때 `\r\n`을 `\n`으로 바꾼다. 수정한 파일을 그대로 다시 쓰면 CRLF가 전부 LF가 된다.
- 기존 파일의 일부만 고치는 스크립트라면 `newline=''`(또는 바이너리 모드)로 열자.
- diff가 터무니없이 크면 `git diff --ignore-cr-at-eol`로 줄바꿈 문제인지부터 확인한다.

끝
