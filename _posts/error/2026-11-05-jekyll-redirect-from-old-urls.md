---
comments: true
layout: post
title: 글 날짜를 고쳤더니 옛 주소가 전부 404, GitHub Pages에서 jekyll-redirect-from으로 옮기기
date: 2026-11-05 09:00:00 +0900
category: error
---

이 블로그는 몇 번 손을 보면서 글 날짜를 여러 번 바꿨다. 옮겨 오면서 날짜가 2025년으로 찍힌 적도 있고, 2026년 봄 날짜로 바뀐 적도 있고, 마지막에는 실제로 쓴 날짜(2021년 등)로 되돌렸다.

문제는 Jekyll의 기본 주소 규칙이 날짜를 포함한다는 점이다.

```
/:categories/:year/:month/:day/:title.html
```

날짜를 바꾸면 주소가 바뀐다. 사이트 안의 링크는 빌드 때 다시 만들어지니 괜찮지만, **바깥에서 옛 주소로 들어오는 사람은 404를 본다.** 구글도 그중 하나다.

### 1. 옛 주소가 아직 검색에 나오고 있었다

서치콘솔 검색 실적을 페이지별로 보다가 이상한 주소를 발견했다.

```
/programmers/2026/05/10/프로그래머스_H-Index.html
```

지금 이 글의 주소는 `/programmers/2021/02/16/...`이다. 2026/05/10은 중간에 잠깐 썼던 날짜다. 그런데 이 404 주소가 한 달 동안 검색 결과에 **12번 노출되고 1번 클릭**됐다. 클릭한 사람은 404를 봤다는 뜻이다.

### 2. 옛 주소 목록을 git 기록에서 만들기

어떤 주소들이 있었는지 기억으로는 알 수 없다. 대신 git 기록에는 커밋마다 글 파일의 날짜·카테고리·파일명이 남아 있다. 모든 커밋을 돌면서 그 시점의 주소를 계산하면 된다.

```python
import subprocess, re

def git(*args):
    return subprocess.run(["git", *args], capture_output=True, text=True).stdout

urls = {}
for commit in git("rev-list", "HEAD", "--", "_posts").split():
    for path in git("ls-tree", "-r", "--name-only", commit, "_posts").splitlines():
        if not path.endswith(".md"):
            continue
        src = git("show", f"{commit}:{path}")
        date = re.search(r"^date:\s*(\d{4})-(\d{2})-(\d{2})", src, re.M)
        cat = re.search(r"^categor(?:y|ies):\s*(\S+)", src, re.M)
        slug = re.sub(r"^\d{4}-\d{2}-\d{2}-", "", path.rsplit("/", 1)[1])[:-3]
        if date and cat:
            y, m, d = date.groups()
            urls.setdefault(slug, set()).add(
                f"/{cat.group(1)}/{y}/{m}/{d}/{slug.replace(' ', '-')}.html")
```

결과는 이랬다.

- git 기록에 등장한 옛 주소: **70개**
- 그중 지금 공개된 글로 이어지는 것: 49개
- 실제로 연결한 것: **46개**

빠진 3개는 슬러그가 같은 비공개 글이 따로 있어서, 어느 글의 옛 주소인지 확정할 수 없었던 경우다. 엉뚱한 글로 보내는 것보다 404가 낫다고 보고 뺐다. 비공개로 돌린 글의 옛 주소도 연결하지 않았다.

### 3. GitHub Pages에서는 301을 못 쓴다

보통이라면 서버에서 301 리다이렉트를 걸면 끝이다. 그런데 GitHub Pages는 정적 파일만 내려주는 곳이라 응답 코드나 서버 설정을 건드릴 수 없다.

대신 쓸 수 있는 게 `jekyll-redirect-from` 플러그인이다. 옛 주소 자리에 "바로 새 주소로 이동하라"는 작은 HTML 파일을 만들어준다.

**Gemfile**

```ruby
group :jekyll_plugins do
  gem "jekyll-redirect-from", "~> 0.16"
end
```

**_config.yml**

```yaml
plugins:
  - jekyll-redirect-from
```

**글 front matter**

```yaml
---
title: 프로그래머스 - H-Index - C++
date: 2021-02-16 20:22:00 +0900
category: programmers
redirect_from:
  - "/programmers/2025/02/16/프로그래머스_H-Index.html"
  - "/programmers/2026/05/10/프로그래머스_H-Index.html"
---
```

GitHub Pages의 기본 빌드는 허용된 플러그인만 쓴다. 이 블로그는 Actions로 직접 빌드하고 있어서 Gemfile에 넣기만 하면 됐다.

### 4. 만들어지는 페이지

옛 주소로 들어가면 이런 HTML이 내려온다.

```html
<link rel="canonical" href="https://nokkiwa.github.io/programmers/2021/02/16/...">
<script>location="https://nokkiwa.github.io/programmers/2021/02/16/..."</script>
<meta http-equiv="refresh" content="0; url=https://nokkiwa.github.io/programmers/2021/02/16/...">
<meta name="robots" content="noindex">
```

- 브라우저는 자바스크립트나 `meta refresh`로 바로 새 주소로 넘어간다.
- 검색엔진에는 `canonical`로 진짜 주소를 알려주고, 이 리다이렉트 페이지 자체는 `noindex`로 색인하지 말라고 한다.
- 구글 문서에 따르면 0초 `meta refresh`는 영구 리다이렉트로 해석한다. 301과 완전히 같지는 않아도 이 상황에서 할 수 있는 최선이다.

응답 코드는 200이라는 점은 알아두자. `curl -I`로 보면 리다이렉트가 아니라 정상 페이지로 보인다.

### 5. 확인

빌드 후 46개 옛 주소를 하나씩 열어서 `canonical`이 올바른 글을 가리키는지 확인했다. 사이트맵에는 리다이렉트 페이지가 들어가면 안 된다. 이 블로그의 `sitemap.xml`은 `site.posts`만 도는 직접 만든 템플릿이라 리다이렉트 페이지가 끼어들 틈이 없었고, 사이트맵 주소 개수도 추가 전후로 같았다. 사이트맵을 다른 방식으로 만든다면 이 부분은 한 번 확인해보는 게 좋다.

서치콘솔에서 옛 주소 노출이 완전히 사라지는 데는 몇 주가 걸린다. 그 사이에 들어오는 사람은 이제 404 대신 원래 글을 본다.

### 정리

- Jekyll 주소에 날짜가 들어가면 날짜 수정 = 주소 변경이다. 공개한 뒤에는 날짜를 함부로 고치지 말자.
- 이미 바꿨다면 git 기록에서 옛 주소를 계산해낼 수 있다.
- GitHub Pages에선 301 대신 `jekyll-redirect-from`. `canonical` + `meta refresh` + `noindex` 조합이 만들어진다.

끝
