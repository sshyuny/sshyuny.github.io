# sshyuny의 개발 블로그

개발하며 배우고 경험한 내용을 기록하는 개인 블로그([sshyuny.github.io](https://sshyuny.github.io/))입니다.

Jekyll 기반의 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 테마와
[Chirpy Starter](https://github.com/cotes2020/chirpy-starter)를 사용합니다.

## 주요 파일

- `_posts/`: 발행할 글
- `_drafts/`: 작성 중인 초안
- `_tabs/`: 소개·태그·카테고리·아카이브 페이지
- `_config.yml`: 블로그 설정

## 배포

`main`에 PR을 머지하거나 직접 push하면 GitHub Actions가 빌드·검증 후 GitHub Pages에 배포합니다.
설정은 [pages-deploy.yml](.github/workflows/pages-deploy.yml)에 있습니다.

`README.md`, `LICENSE`, `.gitignore`만 변경한 경우에는 배포하지 않습니다.
