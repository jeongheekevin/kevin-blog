---
title: "GitHub Pages에 Astro 블로그 배포하기: /blog 링크가 404가 된 이유"
description: "Astro 블로그를 GitHub Pages project site에 배포하면서 겪은 base path 문제와 해결 과정을 정리했습니다."
pubDate: "Jun 03 2026"
#heroImage: "../../assets/blog-placeholder-1.jpg" TODO fix
---

## 시작하며

기술 블로그와 포트폴리오를 운영하기 위해 Astro 기반 블로그를 만들고 GitHub Pages에 배포했습니다.

처음에는 Astro 블로그 템플릿을 생성하고 GitHub Pages에 올리면 끝날 것이라고 생각했습니다. 하지만 GitHub Pages의 project site 구조 때문에 `/blog`, `/about`, 개별 글 링크에서 404가 발생했습니다.

이 글에서는 Astro 블로그를 GitHub Pages에 배포하는 초기 세팅 과정과, 특히 `base path` 문제를 어떻게 해결했는지 정리했습니다.

## 목표

이번 세팅의 목표는 다음과 같았습니다.

- Markdown / MDX 기반으로 기술 블로그를 작성했습니다.
- GitHub Pages를 통해 외부에 공개했습니다.
- GitHub Actions로 빌드와 배포를 자동화했습니다.
- 포트폴리오용 기술 블로그로 사용할 수 있게 기본 구조를 잡았습니다.

최종 배포 주소는 다음 형태였습니다.

```text
https://jeongheekevin.github.io/kevin-blog/
```

## 프로젝트 생성

Astro 프로젝트는 blog template을 사용했습니다.

```bash
npm create astro@latest kevin-blog
```

선택지는 다음과 같이 설정했습니다.

```text
How would you like to start your new project?
→ Use blog template

Install dependencies?
→ Yes

Initialize a new git repository?
→ No
```

이미 GitHub repository를 clone한 폴더에서 작업할 예정이었기 때문에 Astro가 새 Git repository를 만들도록 하지 않았습니다.

이후 GitHub clone 폴더에서 Astro 프로젝트를 생성했습니다.

```bash
cd ~/IdeaProjects/kevin-blog
npm create astro@latest .
```

## 로컬 실행

개발 서버는 다음 명령어로 실행했습니다.

```bash
npm run dev
```

기본 주소는 다음과 같았습니다.

```text
http://localhost:4321/
```

Astro blog template은 기본적으로 다음 페이지들을 제공했습니다.

```text
/
/blog/
/about/
/blog/{slug}/
```

로컬에서는 `/blog`, `/about` 경로가 정상적으로 동작했습니다.

## GitHub Pages 배포 방식

Astro는 정적 사이트 생성기입니다. 즉, Markdown과 Astro 컴포넌트를 바로 배포하는 것이 아니라 빌드 결과물인 `dist` 디렉터리를 배포해야 했습니다.

빌드 명령어는 다음과 같습니다.

```bash
npm run build
```

이 명령을 실행하면 다음과 같은 정적 파일이 생성되었습니다.

```text
dist/
├── index.html
├── blog/
│   └── index.html
├── about/
│   └── index.html
└── _astro/
```

GitHub Pages에는 이 `dist` 결과물이 배포되었습니다.

이를 자동화하기 위해 GitHub Actions workflow를 추가했습니다.

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    needs: build
    runs-on: ubuntu-latest
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

GitHub repository 설정에서는 다음 항목도 변경해야 했습니다.

```text
Settings
→ Pages
→ Build and deployment
→ Source: GitHub Actions
```

이 설정을 하지 않으면 `actions/deploy-pages` 단계에서 다음과 같은 에러가 발생했습니다.

```text
Error: Creating Pages deployment failed
Error: HttpError: Not Found
Ensure GitHub Pages has been enabled
```

이 에러는 빌드 실패가 아니라 GitHub Pages가 repository에서 활성화되지 않았기 때문에 발생했습니다.

## Project site와 User site의 차이

GitHub Pages는 크게 두 가지 방식이 있습니다.

첫 번째는 user site입니다.

```text
https://{username}.github.io/
```

이 경우 repository 이름은 다음과 같아야 합니다.

```text
{username}.github.io
```

두 번째는 project site입니다.

```text
https://{username}.github.io/{repository-name}/
```

이번 블로그 repository 이름은 `kevin-blog`였기 때문에 최종 주소는 다음과 같았습니다.

```text
https://jeongheekevin.github.io/kevin-blog/
```

즉, 사이트가 루트(`/`)가 아니라 `/kevin-blog/` 하위에 배포되는 구조였습니다.

이 차이를 무시하면 링크가 깨집니다.

## base path 설정

Astro에서 GitHub Pages project site에 배포하려면 `astro.config.mjs`에 `site`와 `base`를 설정해야 했습니다.

```js
export default defineConfig({
	site: 'https://jeongheekevin.github.io',
	base: '/kevin-blog/',
	integrations: [mdx(), sitemap()],
});
```

여기서 중요한 점은 `site`와 `base`의 역할이 다르다는 점이었습니다.

```text
site = 사이트의 도메인
base = repository 하위 경로
```

처음에는 다음처럼 설정하면 되는 줄 알았습니다.

```js
site: 'https://jeongheekevin.github.io/kevin-blog/'
```

하지만 이것만으로는 내부 링크 문제가 해결되지 않았습니다. `site`는 sitemap, RSS 같은 절대 URL 생성에 주로 사용되고, 실제 하위 경로 배포에는 `base` 설정이 필요했습니다.

정리하면 project site에서는 다음처럼 분리하는 편이 안전했습니다.

```js
site: 'https://jeongheekevin.github.io',
base: '/kevin-blog/',
```

## 발생한 문제: 개별 글 링크 404

메인 페이지, 블로그 목록, About 페이지는 정상적으로 열렸습니다.

```text
정상
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

하지만 블로그 목록에서 개별 글을 클릭하면 다음 주소로 이동했습니다.

```text
비정상
https://jeongheekevin.github.io/blog/markdown-style-guide/
```

정상 주소는 다음이어야 했습니다.

```text
정상
https://jeongheekevin.github.io/kevin-blog/blog/markdown-style-guide/
```

즉, `/kevin-blog/` prefix가 빠진 상태였습니다.

## 원인

문제는 `src/pages/blog/index.astro` 파일의 링크 생성 코드였습니다.

기존 코드는 다음과 같았습니다.

```astro
<a href={`/blog/${post.id}/`}>
```

이 코드는 루트 경로(`/`)를 기준으로 링크를 만듭니다. 로컬 개발 환경에서는 문제가 없어 보였습니다.

```text
http://localhost:4321/blog/markdown-style-guide/
```

하지만 GitHub Pages project site에서는 사이트가 `/kevin-blog/` 아래에 배포되므로 이 링크는 운영 환경에서 깨졌습니다.

```text
https://jeongheekevin.github.io/blog/markdown-style-guide/
```

## 해결

Astro는 `import.meta.env.BASE_URL` 값을 제공합니다. 이 값은 `astro.config.mjs`의 `base` 설정을 반영합니다.

따라서 `src/pages/blog/index.astro`에 다음 변수를 추가했습니다.

```astro
const BASE = import.meta.env.BASE_URL;
```

그리고 링크를 다음처럼 수정했습니다.

```astro
<a href={`${BASE}blog/${post.id}/`}>
```

수정 후 구조는 다음과 같았습니다.

```astro
const posts = (await getCollection('blog')).sort(
	(a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf(),
);

const BASE = import.meta.env.BASE_URL;
```

```astro
<a href={`${BASE}blog/${post.id}/`}>
```

이제 운영 환경에서는 다음처럼 정상 링크가 생성되었습니다.

```text
https://jeongheekevin.github.io/kevin-blog/blog/markdown-style-guide/
```

## 로컬 테스트에서 헷갈린 점

로컬에서는 `npm run dev`나 `npm run preview`를 실행하면 보통 다음 경로로 접근합니다.

```text
http://localhost:4321/blog/
http://localhost:4321/about/
```

반면 운영 환경은 다음과 같습니다.

```text
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

이 차이 때문에 로컬 테스트와 운영 테스트 결과가 다르게 보일 수 있었습니다.

특히 `npx serve dist`로 빌드 결과물을 확인할 때도 주의해야 했습니다.

```bash
npm run build
npx serve dist
```

이 방식은 `dist` 디렉터리를 로컬 서버의 루트(`/`)에 올려서 보여줍니다. 따라서 GitHub Pages의 `/kevin-blog/` mount 구조를 완전히 재현하지는 않습니다.

결국 확인 기준은 다음처럼 나누는 것이 좋았습니다.

```text
로컬 개발 확인
→ http://localhost:4321/blog/

운영 배포 확인
→ https://jeongheekevin.github.io/kevin-blog/blog/
```

## 최종 확인

수정 후 다음 명령어로 빌드가 통과하는지 확인했습니다.

```bash
npm run build
```

그리고 변경 사항을 커밋하고 push했습니다.

```bash
git add .
git commit -m "Fix blog post links for GitHub Pages"
git push
```

GitHub Actions가 자동으로 빌드와 배포를 수행했습니다.

배포 후 다음 경로들이 정상적으로 동작하는지 확인했습니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
https://jeongheekevin.github.io/kevin-blog/blog/markdown-style-guide/
```

## 앞으로의 규칙

이번 문제를 다시 겪지 않기 위해 다음 기준을 정했습니다.

- 내부 링크는 `/blog/...`처럼 루트 기준으로 직접 작성하지 않습니다.
- GitHub Pages project site 배포에서는 반드시 `base`를 설정합니다.
- 운영 URL 기준으로 최종 링크를 검증합니다.
- 로컬 preview 결과만 보고 배포 경로가 정상이라고 판단하지 않습니다.
- 내부 링크를 만들 때는 `import.meta.env.BASE_URL`을 고려합니다.

## 정리

이번 세팅에서 얻은 핵심은 다음과 같습니다.

- Astro는 빌드 결과물인 `dist`를 배포합니다.
- GitHub Pages project site는 `/repository-name/` 하위 경로에 배포됩니다.
- Astro에서는 project site 배포 시 `base` 설정이 필요합니다.
- 내부 링크를 직접 `/blog/...`처럼 만들면 운영 환경에서 404가 날 수 있습니다.
- 내부 링크 생성 시 `import.meta.env.BASE_URL`을 고려해야 합니다.

이번 문제는 단순히 “GitHub Pages 배포 실패”가 아니라, 로컬 환경과 운영 환경의 base path 차이를 제대로 이해하지 못해서 발생한 문제였습니다.