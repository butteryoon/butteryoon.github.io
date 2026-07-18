# Awesome Life

윈도우즈 및 리눅스 터미널 환경에서 할 수 있는 것들에 대한 기록이자 AI를 이용하여 블로그 관리하는 목적으로 사용한다. 

- 사이트: [https://butteryoon.github.io](https://butteryoon.github.io)
- 작성자: James Yoon ([@butteryoon](https://github.com/butteryoon))

> 이 블로그의 글 초안은 **Claude (Claude Code)** 로 작성됩니다. 또한 오픈소스 AI 플랫폼을 기업환경에 구성하기 위한 서베이 목적의 글들을 다룹니다.

## 기술 스택

- [Jekyll](https://jekyllrb.com/) + GitHub Pages (`master` 브랜치에서 자동 배포)
- 테마: [Flexible-Jekyll](https://github.com/artemsheludko/flexible-jekyll) (Artem Sheludko)
- 플러그인: jekyll-sitemap, jekyll-paginate, jekyll-gist, jemoji, jekyll-youtube
- 댓글: Disqus / 분석: Google Analytics, MS Clarity
- 글의 초안은 Orca 환경에서 **Claude Code**를 이용하여 작성하고, 사람이 검토·보완 후 발행

## 로컬 실행

```bash
bundle install
bundle exec jekyll serve            # http://localhost:4000
bundle exec jekyll serve --drafts   # _drafts 포함
```

## 글 작성

1. `_drafts/0000-00-00-templete.md` 템플릿을 복사해 `_posts/YYYY-MM-DD-제목.md`로 저장
2. front matter의 `title`, `description`, `img`, `date`(`+0900`), `tags`, `categories` 작성
3. 요약 구분자 `<!--more-->`를 본문 도입부 뒤에 삽입
4. 이미지는 `assets/img/`에 추가
5. `master`에 push하면 GitHub Pages가 자동 빌드

## License

[GNU General Public License v3.0](LICENSE) — 테마 원본 라이선스를 따릅니다.