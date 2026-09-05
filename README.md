# KLOG

개발하며 만난 에러와 해결 기록을 남기는 개인 기술 블로그입니다.

- 사이트: https://nokkiwa.github.io
- 정적 사이트 생성기: [Jekyll](https://jekyllrb.com/)
- 호스팅: GitHub Pages

## 카테고리

| 디렉터리 | 내용 |
|---|---|
| `_posts/error` | 개발 중 만난 오류의 원인과 해결 과정 |
| `_posts/infra` | 서버 운영, 배포, 네트워크 설정 |
| `_posts/go` | Go 언어 학습 노트 |
| `_posts/programmers` | 알고리즘 문제 풀이와 접근 방식 |
| `_posts/project` | 토이 프로젝트 회고 |

## 로컬 실행

```bash
bundle install
bundle exec jekyll serve
```

기본 주소는 http://localhost:4000 입니다.

## 디렉터리 구조

```
_config.yml      사이트 설정
_includes/       head, sidebar, 목록·본문 템플릿 조각
_layouts/        default / page / post / category 레이아웃
_posts/          글 (카테고리별 디렉터리)
public/          CSS, 이미지 등 정적 파일
about.md         소개 페이지
privacy.md       개인정보처리방침
contact.md       연락처
```

## 라이선스

글의 저작권은 작성자에게 있습니다. 인용 시 출처를 남겨 주세요.
