---
comments: true
layout: post
title: Jekyll 빌드 한 번에 git 변경 파일 1,790개가 생기는 문제 (_site 추적)
date: 2026-09-25 09:00:00 +0900
category: error
---

블로그 홈 화면을 고치고 로컬에서 `bundle exec jekyll build`로 확인해봤다. 빌드는 문제없이 끝났는데 `git status`를 치니 변경된 파일이 1,790개 가까이 떴다. 바꾼 파일은 3개뿐이었다.

아래는 원인을 찾은 과정이다.

### 1. 무엇이 바뀌었나 확인

```bash
git status --short | awk '{print $2}' | cut -d/ -f1 | sort | uniq -c | sort -rn
```

변경 파일 대부분이 `_site/` 아래였고, 나머지는 `.jekyll-cache/`였다. 둘 다 Jekyll이 빌드할 때 만들어내는 결과물이다.

### 2. 빌드 결과물이 왜 git에 있나

```bash
git ls-files _site | wc -l          # 1825
git ls-files .jekyll-cache | wc -l  # 44
ls -a | grep gitignore              # 없음
```

원인은 단순했다. 저장소에 `.gitignore`가 아예 없었다. 그래서 언젠가 `git add .`를 했을 때 `_site` 폴더가 통째로 커밋됐고, 그 뒤로 빌드할 때마다 파일이 새로 쓰이니 전부 변경으로 잡힌 것이다.

`_site`를 마지막으로 건드린 커밋을 찾아봤다.

```bash
git log --format='%ci' -1 -- _site
# 2026-01-18 19:37:53 +0900
```

1월 이후로는 한 번도 갱신되지 않았다. 안을 열어보니 `_site/2025-9/`, `_site/2025-10/` 같은 폴더에 이 블로그와 상관없는 예전 글 HTML이 수백 개 남아 있었다. 폴더 용량만 107MB였다.

### 3. 라이브 사이트에는 영향이 있었나

처음엔 저 옛 HTML이 사이트에 그대로 노출되고 있는 줄 알고 놀랐다. 그런데 배포 방식을 보니 괜찮았다. 이 블로그는 GitHub Actions로 배포하는데, 워크플로가 매번 소스를 새로 빌드한다.

{% raw %}
```yaml
- name: Build with Jekyll
  run: bundle exec jekyll build --baseurl "${{ steps.pages.outputs.base_path }}"
- name: Upload artifact
  uses: actions/upload-pages-artifact@v3
```
{% endraw %}

러너에서 새로 만든 `_site`를 올리기 때문에, 저장소에 커밋된 `_site`는 배포에 쓰이지 않는다. 실제로 남아 있던 옛 글 주소로 접속해보니 404가 떴다.

정리하면 **사이트에는 문제가 없고, 저장소만 지저분해지는 문제**였다. 대신 불편한 점이 있었다.

- 빌드 한 번에 수천 개 파일이 바뀌어서 진짜로 고친 파일이 `git status`에 묻힌다.
- 실수로 `git add .`를 하면 의미 없는 대형 커밋이 또 생긴다.
- 클론할 때마다 쓰지도 않는 100MB를 같이 받는다.

### 4. 임시로 피해간 방법

바로 정리하기 전까지는 빌드 결과를 저장소 밖으로 빼서 확인했다.

```bash
bundle exec jekyll build -d /tmp/klog-build
```

`-d`(destination) 옵션을 주면 `_site` 대신 지정한 경로에 빌드된다. 저장소는 건드리지 않으니 `git status`가 깨끗하게 유지된다.

### 5. 제대로 정리하기

근본적인 해결은 결과물을 추적 대상에서 빼는 것이다.

```bash
cat > .gitignore <<'EOF'
_site/
.jekyll-cache/
.sass-cache/
.jekyll-metadata
EOF

git rm -r --cached _site .jekyll-cache
git add .gitignore
git commit -m "빌드 결과물 추적 해제"
```

`--cached`를 꼭 붙여야 한다. 이 옵션이 있어야 git 추적만 끊기고 로컬 파일은 그대로 남는다.

이렇게 해도 예전 커밋 안에는 파일이 그대로 남아 있어서 `.git` 폴더 크기는 줄지 않는다. 히스토리까지 지우려면 `git filter-repo`로 과거 커밋을 다시 써야 한다. 하지만 그러면 강제 푸시가 필요하고, 개인 블로그에서 그만큼 수고할 가치는 없다고 봤다. 앞으로 안 쌓이게 막는 것으로 충분하다.

### 6. 덤: CRLF 파일

비슷한 시기에 같이 발견한 문제가 하나 더 있다. `public/css/style.css`가 CRLF와 LF 줄바꿈이 섞인 파일이었다.

```bash
file public/css/style.css
# Unicode text, UTF-8 text, with CRLF, LF line terminators
```

이런 파일을 LF로 저장하는 에디터나 스크립트로 한 줄만 고쳐도 diff에는 파일 전체가 바뀐 것으로 나온다. 내가 뭘 바꿨는지 리뷰하기 어려워진다. 줄바꿈을 한쪽으로 맞추려면 `.gitattributes`에 규칙을 두는 게 가장 깔끔하다.

```
*.css text eol=lf
```

### 정리

- Jekyll 저장소에는 처음부터 `.gitignore`에 `_site/`와 `.jekyll-cache/`를 넣어두자.
- GitHub Actions로 배포하면 커밋된 `_site`는 배포에 쓰이지 않는다. 그래서 라이브에는 티가 안 나고 발견이 늦어진다.
- 급할 땐 `jekyll build -d <다른 경로>`로 피해가고, 정리할 땐 `git rm -r --cached`를 쓰면 된다.

끝
