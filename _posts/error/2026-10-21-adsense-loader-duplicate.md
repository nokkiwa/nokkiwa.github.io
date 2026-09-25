---
comments: true
layout: post
title: 애드센스 adsbygoogle.js가 페이지에 두 번 로드되던 문제
date: 2026-10-21 09:00:00 +0900
category: error
---

블로그 광고 설정을 정리하다가 페이지 소스에 `adsbygoogle.js` 스크립트가 두 번 들어가 있는 걸 발견했다.

### 1. 확인

템플릿 폴더에서 검색해봤다.

```bash
grep -rn "adsbygoogle.js" _includes _layouts
# _includes/head.html:5:     src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-..."
# _includes/default.html:21: src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-..."
```

`head.html`은 모든 페이지의 `<head>`에 들어가고, `default.html`은 그 `head.html`을 include한 다음 본문 아래 광고 칸을 그린다. 결과적으로 모든 페이지에서 같은 스크립트가 `<head>`에 한 번, 광고 칸 바로 앞에 한 번 들어가고 있었다.

```html
<!-- head.html -->
<script async
  src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-..."
  crossorigin="anonymous"></script>

<!-- default.html, 광고 칸 -->
<div class="post-ad">
  <script async
    src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-..."
    crossorigin="anonymous"></script>
  <ins class="adsbygoogle" data-ad-slot="..." ...></ins>
  <script>(adsbygoogle = window.adsbygoogle || []).push({});</script>
</div>
```

### 2. 왜 이렇게 됐나

애드센스에서 코드를 받는 경로가 두 개라서 생긴 일이다.

- **사이트 인증/자동 광고용 코드**: 애드센스 가입할 때 "`<head>`에 붙여넣으세요"라고 주는 한 줄짜리 로더다.
- **광고 단위 코드**: 광고 단위를 만들면 로더 `<script>`, `<ins>`, `push()`를 한 묶음으로 준다.

두 번째 묶음을 통째로 복사해서 넣으니 로더가 겹쳤다. 광고 단위 코드 안의 로더는 "혹시 head에 로더가 없을 때"를 위한 것이라, head에 이미 있으면 필요 없다.

### 3. 수정

광고 칸 쪽의 로더만 지우고 `<ins>`와 `push()`는 남긴다.

```html
<div class="post-ad">
  <ins class="adsbygoogle" data-ad-slot="..." ...></ins>
  <script>(adsbygoogle = window.adsbygoogle || []).push({});</script>
</div>
```

`push()`는 `window.adsbygoogle` 배열이 없으면 새로 만들어서 쌓아둔다. 그래서 로더가 `async`로 늦게 도착해도 도착한 뒤에 쌓인 요청을 처리한다. 로더가 head에 한 번만 있으면 충분하다.

### 4. 확인

```bash
grep -rn "adsbygoogle.js" _includes _layouts
# _includes/head.html:5: ...   <- 한 줄만 남아야 한다
```

배포 후에는 글 페이지에서 개발자 도구를 열고 `document.querySelectorAll('script[src*="adsbygoogle.js"]').length`를 쳐서 1이 나오는지 보면 된다.

### 정리

- 애드센스 로더는 페이지당 한 번만, `<head>`에 둔다.
- 광고 단위 코드를 붙일 때는 로더 `<script>`를 빼고 `<ins>`와 `push()`만 넣는다.
- Jekyll처럼 템플릿을 include로 쪼개 쓰면 어디에 뭐가 들어갔는지 한눈에 안 보인다. `grep`으로 한 번씩 확인하자.

끝
