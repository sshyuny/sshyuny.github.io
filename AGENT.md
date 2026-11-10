# sshyuny.github.io

Jekyll + Chirpy 테마(`jekyll-theme-chirpy` gem, `chirpy-starter` 템플릿 기반) 개인 블로그. GitHub Pages로 배포.

## 배포

- `main`에 push하면 `.github/workflows/pages-deploy.yml`이 빌드·배포한다.
- 레포 Settings → Pages의 Source는 `GitHub Actions`로 설정돼 있다(레포 파일이 아닌 GitHub 설정).
- 확인: `gh run list`로 build/deploy 성공 확인 후 https://sshyuny.github.io/ 응답 확인.

## 로컬 빌드

- 시스템 Ruby는 2.6이라 Chirpy 7 빌드 불가. 로컬 미리보기가 필요하면 Ruby 3.x를 별도 설치해야 한다.
- 아직 설치하지 않았으므로 지금은 Actions 빌드로 검증한다.

## 커스텀 방식

- 레이아웃·include·CSS는 레포에 없고 gem 안에 있다.
- 바꾸려면 gem(https://github.com/cotes2020/jekyll-theme-chirpy)의 해당 파일을 같은 경로(`_layouts/`, `_includes/`, `_sass/` 등)로 복사해 덮어쓴다.
- 사이트에 노출되면 안 되는 루트 파일은 `_config.yml`의 `exclude`에 추가한다(이 파일 포함).

## 공개 정보

- 이메일은 공개하지 않는다(`social.email` 비움, 사이드바 email 아이콘 제거).
