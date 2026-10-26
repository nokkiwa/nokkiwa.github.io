---
comments: true
layout: post
title: 서치콘솔 '문자열의 이스케이프 시퀀스가 잘못됨', Jekyll JSON-LD에 escape를 쓰면 생기는 일
date: 2026-10-28 09:00:00 +0900
category: error
---

구글 서치콘솔에서 "문자열의 이스케이프 시퀀스가 잘못됨"이라는 구조화 데이터 오류 메일이 왔다. 원인은 Jekyll 템플릿의 JSON-LD에 HTML용 `escape` 필터를 쓴 것이었다. 받은 메일은 이랬다.

```
https://nokkiwa.github.io/에서 파싱할 수 없는 구조화된 데이터 문제가 감지됨
심각한 문제 중 주요 문제: 문자열의 이스케이프 시퀀스가 잘못됨
```

영향을 받은 페이지는 딱 하나, 이 블로그의 MySQL 이모지 저장 에러 글이었다. 템플릿은 모든 글이 똑같이 쓰는데 왜 이 글만 깨졌는지부터 봤다.

### 1. 실제로 나간 JSON

라이브 페이지의 `<script type="application/ld+json">` 블록을 그대로 꺼내 파이썬으로 파싱해봤다.

```python
json.loads(block)
# json.decoder.JSONDecodeError: Invalid \escape: line 10 column 104
```

문제의 위치는 `description` 값이었다.

```json
"description": "게시글 제목에 이모지가 들어가자 저장이 실패했다. ERROR 1366 (HY000): Incorrect string value: &amp;#39;\xF0\x9F\x9A\x80&amp;#39; for column ..."
```

글 본문 첫 부분에 MySQL 에러 메시지를 그대로 넣었는데, 거기 들어 있던 `\xF0\x9F\x9A\x80`이 설명문으로 그대로 들어갔다. JSON 문자열에서 역슬래시 다음에 올 수 있는 건 `"` `\` `/` `b` `f` `n` `r` `t` `u` 뿐이다. `\x`는 JSON에 없는 이스케이프라 파서가 거기서 멈춘다. 다른 글에는 본문 앞부분에 역슬래시가 없어서 운 좋게 안 터졌던 것이다.

### 2. 원인은 템플릿의 escape 필터

`_includes/head.html`의 JSON-LD는 이렇게 되어 있었다.

{% raw %}
```liquid
"headline": "{{ page.title | escape }}",
"description": "{{ meta_desc | default: site.description | strip_newlines | escape | truncate: 200 }}",
```
{% endraw %}

`escape`는 **HTML용** 필터다. `<` `>` `&` `"` `'`를 HTML 엔티티로 바꿀 뿐 역슬래시는 건드리지 않는다. JSON 안에 값을 넣을 때 필요한 건 JSON 규칙대로 이스케이프하는 것인데, 엉뚱한 필터를 쓰고 있었다.

같이 보다가 하나 더 찾았다. 위 JSON에 `&amp;#39;`가 보인다. 설명문(`meta_desc`)을 만들 때 이미 `escape`를 한 번 했는데 출력할 때 또 `escape`를 해서 `'` → `&#39;` → `&amp;#39;`가 된 것이다. JSON-LD뿐 아니라 `<meta name="description">`에도 똑같이 들어가 있었고, 이러면 검색 결과 설명문에 `&#39;`가 글자 그대로 보일 수 있다.

### 3. 수정

원칙은 하나다. **값은 평문으로 들고 있다가, 출력하는 곳의 문법에 맞게 딱 한 번만 이스케이프한다.**

{% raw %}
```liquid
{%- assign meta_desc = meta_body | strip_html | strip_newlines | truncatewords: 28 -%}
{%- assign meta_desc = meta_desc | default: site.description | strip_newlines
    | replace: '&lt;', '<' | replace: '&gt;', '>' | replace: '&quot;', '"'
    | replace: '&#39;', "'" | replace: '&amp;', '&' -%}

<meta name="description" content="{{ meta_desc | escape }}" />

"headline": {{ page.title | jsonify | replace: '</', '<\/' }},
"description": {{ meta_desc | truncate: 200 | jsonify | replace: '</', '<\/' }},
```
{% endraw %}

- `strip_html`은 태그만 지우고 kramdown이 만든 엔티티(`&lt;` 같은)는 남긴다. 그래서 평문으로 되돌리는 `replace`를 한 번 거친다. `&amp;`는 반드시 마지막에 되돌려야 `&amp;lt;` 같은 게 두 번 풀리지 않는다.
- HTML 속성에는 `escape`, JSON에는 `jsonify`를 쓴다. `jsonify`는 따옴표까지 붙여서 내보내므로 바깥의 `"`를 지워야 한다.
- `jsonify`는 `</`를 건드리지 않는다. 설명문에 `</script>`가 들어가면 스크립트 블록이 거기서 끝나버리니 `<\/`로 바꿔둔다. JSON에서 `\/`는 `/`와 같은 뜻이라 값은 그대로다.

### 4. 확인

한 페이지만 보고 끝내면 다음 글에서 또 터진다. 예약해둔 글까지 포함해서 프로덕션 빌드를 하고 모든 페이지의 JSON-LD를 파싱해봤다.

```bash
JEKYLL_ENV=production bundle exec jekyll build --future -d /tmp/site
```

```python
for f in glob.glob("/tmp/site/**/*.html", recursive=True):
    for block in re.findall(r'<script type="application/ld\+json">(.*?)</script>', open(f).read(), re.S):
        json.loads(block)   # 실패하면 예외
```

| | 수정 전 | 수정 후 |
|---|---|---|
| JSON-LD 파싱 성공 | 61 / 62 | 62 / 62 |
| `&amp;#39;` 같은 이중 이스케이프 페이지 | 5 | 0 |

배포한 뒤에는 서치콘솔의 해당 항목에서 '수정 결과 확인'을 눌러 다시 검사를 요청하면 된다. 참고로 이 오류가 있어도 페이지 자체는 색인된다. 구조화 데이터만 무시될 뿐이다.

### 정리

- Liquid의 `escape`는 HTML용이다. JSON-LD 안에서는 `jsonify`를 쓴다.
- 이스케이프는 출력 직전에 한 번만. 중간 변수에 미리 해두면 다른 곳에서 한 번 더 하게 된다.
- 에러 메시지·경로·정규식처럼 역슬래시가 들어가는 글을 쓰는 개발 블로그라면 언젠가는 걸린다. 빌드 결과를 `json.loads`로 한 번 돌려보는 게 제일 싸다.

끝
