---
title: "Astro 블로그 GitHub Pages 배포: build artifact와 base path 문제"
description: "Astro 블로그를 GitHub Pages project site에 배포하면서 source, artifact, hosting 책임을 분리하고, 운영 환경에서 발생한 base path 링크 문제를 해결한 과정을 정리했습니다."
pubDate: "Jun 03 2026"
---

## 문제 상황

Astro 기반 기술 블로그를 GitHub Pages에 배포했습니다.

빌드와 배포 자체는 성공했습니다. GitHub Actions도 정상적으로 완료되었고, 다음 경로도 정상적으로 접근되었습니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

하지만 실제 화면에서 메뉴나 블로그 목록의 개별 글 링크를 클릭하면 문제가 발생했습니다.

예를 들어 블로그 목록에서 글을 클릭했을 때 기대한 주소는 다음과 같았습니다.

```text
https://jeongheekevin.github.io/kevin-blog/blog/markdown-style-guide/
```

하지만 실제로는 다음 주소로 이동했습니다.

```text
https://jeongheekevin.github.io/blog/markdown-style-guide/
```

결과적으로 404가 발생했습니다.

처음에는 GitHub Pages 설정, Astro의 `site` 설정, GitHub Actions 배포 문제를 의심했습니다. 하지만 빌드 산출물과 실제 링크 생성 방식을 확인해보니 문제는 배포 자체가 아니라 **내부 링크 생성 방식**에 있었습니다.

## 왜 순수 HTML, CSS, JavaScript로 만들지 않았는가

기술 블로그는 순수 HTML, CSS, JavaScript만으로도 만들 수 있습니다. 실제로 GitHub Pages는 정적 파일을 그대로 배포할 수 있기 때문에, 단순한 페이지 몇 개라면 별도의 빌드 도구 없이도 충분합니다.

예를 들어 다음과 같은 구조도 가능합니다.

```text
index.html
about.html
blog/
├── first-post.html
└── second-post.html
style.css
```

이 방식의 장점은 명확합니다.

- 빌드 과정이 없습니다.
- GitHub Pages의 `Deploy from a branch` 방식으로 바로 배포할 수 있습니다.
- 설정할 것이 적습니다.
- base path 문제도 상대적으로 단순합니다.

하지만 기술 블로그를 장기적으로 운영하기에는 한계가 있다고 판단했습니다.

가장 큰 이유는 글 작성과 유지보수 비용이었습니다. 순수 HTML로 글을 작성하면 매번 HTML 구조를 직접 작성해야 합니다.

```html
<article>
	<h1>제목</h1>
	<p>본문</p>
	<pre>
		<code>...</code>
	</pre>
</article>
```

처음에는 문제가 없어 보이지만 글이 늘어나면 반복 작업이 많아집니다. 목록 페이지, 날짜 표시, 태그, RSS, sitemap, 코드 하이라이팅, 공통 레이아웃 같은 기능도 직접 관리해야 합니다.

기술 블로그에서는 특히 다음 요소들이 중요했습니다.

```text
Markdown 기반 글 작성
코드 블록
글 목록 자동 생성
공통 레이아웃
RSS
sitemap
정적 빌드
향후 MDX 확장 가능성
```

순수 HTML 방식은 단순하지만, 글이 늘어날수록 콘텐츠 관리 비용이 커집니다. 반면 Astro는 Markdown / MDX 기반으로 글을 작성하고, 빌드 시 정적 HTML로 변환할 수 있습니다.

즉, 작성자는 Markdown으로 글을 작성하고:

```md
# 제목

본문입니다.

```java
public class Example {
}
```
```

Astro는 이를 정적 HTML로 빌드합니다.

```text
Markdown / MDX
→ Astro build
→ HTML
→ GitHub Pages
```

따라서 최종 결과물은 여전히 정적 사이트지만, 작성과 관리 방식은 훨씬 편해집니다.

이번 블로그의 목적은 단순히 페이지 하나를 올리는 것이 아니라, 장기적으로 기술 글을 누적하는 것입니다. 그래서 순수 HTML 방식보다 Astro를 선택했습니다.

정리하면 다음과 같습니다.

```text
순수 HTML/CSS/JS
→ 단순 배포에는 유리
→ 글이 늘어나면 관리 비용 증가

Astro
→ 초기 build 설정 필요
→ Markdown/MDX 기반 글 작성 가능
→ 글 목록, RSS, sitemap, 레이아웃 관리에 유리
```

이 선택 때문에 `npm run build` 단계가 필요해졌고, 자연스럽게 GitHub Actions를 통한 build artifact 배포 구조를 선택하게 되었습니다.



## 배포 구조 선택: GitHub Pages + GitHub Actions

이번 블로그는 다음 구조로 배포했습니다.

```text
GitHub Repository
→ GitHub Actions
→ npm run build
→ dist
→ GitHub Pages
```

이 구조에서 GitHub Actions는 웹 요청을 처리하는 서버가 아닙니다. 빌드와 배포를 수행하는 CI runner입니다. 실제 정적 파일을 제공하는 것은 GitHub Pages입니다.

즉, 역할은 다음처럼 나뉩니다.

```text
GitHub Repository = source code 저장소
GitHub Actions    = build/deploy runner
dist              = build artifact
GitHub Pages      = static hosting
```

이 구조를 선택한 이유는 source code와 build artifact를 분리하기 위해서였습니다.

Astro 프로젝트는 Markdown, Astro 컴포넌트, 설정 파일을 그대로 배포하는 것이 아니라 `npm run build`를 통해 정적 HTML을 생성한 뒤 배포합니다.

따라서 repository에는 source code만 유지하고, 배포 산출물은 GitHub Actions에서 생성해 GitHub Pages로 넘기는 구조가 적합했습니다.

## 왜 GitHub Actions를 사용했는가

GitHub Pages 설정에는 `Deploy from a branch` 방식도 있습니다.

단순 HTML, CSS, JavaScript만 있는 정적 사이트라면 특정 branch의 `/root` 또는 `/docs` 디렉터리를 그대로 배포해도 충분합니다.

하지만 Astro는 build step이 필요한 정적 사이트 생성기입니다.

```text
Astro source
→ npm run build
→ dist
→ GitHub Pages
```

즉, 실제로 배포해야 하는 것은 source code가 아니라 `dist` 디렉터리입니다.

물론 `dist` 결과물을 별도 branch에 올리는 방식도 가능합니다.

```text
main branch
→ Astro source

gh-pages branch
→ dist output
```

하지만 이 방식은 관리 포인트가 늘어납니다.

- source branch와 deploy branch를 따로 관리해야 합니다.
- `dist` 결과물을 직접 commit하거나 별도 배포 도구를 써야 합니다.
- 빌드 결과물이 source history에 섞일 가능성이 있습니다.
- 배포 실패 시 어느 단계에서 실패했는지 추적이 불편합니다.

그래서 GitHub Actions를 빌드 서버처럼 사용했습니다.

여기서 GitHub Actions의 역할은 다음과 같습니다.

```text
1. main branch push 감지
2. GitHub runner에서 dependency 설치
3. npm run build 실행
4. dist 디렉터리 생성
5. GitHub Pages artifact 업로드
6. GitHub Pages에 배포
```

이 구조의 장점은 다음과 같습니다.

- repository에는 source code만 남습니다.
- 배포 결과물인 `dist`를 직접 commit하지 않아도 됩니다.
- push 이후 build와 deploy가 자동화됩니다.
- build 실패와 deploy 실패를 Actions 로그에서 분리해서 확인할 수 있습니다.
- 나중에 테스트, lint, 링크 검증 같은 단계를 추가하기 쉽습니다.

이번 결정의 핵심은 “GitHub Actions를 서버처럼 쓴다”가 아니었습니다.

Astro는 build step이 필요한 정적 사이트이므로, source code와 build output을 분리해야 했습니다. GitHub Actions는 이 build step을 재현 가능한 CI 환경에서 수행하게 해주고, GitHub Pages는 생성된 정적 파일을 외부에 제공하는 역할을 맡았습니다.

## GitHub Pages project site 구조

이번 블로그는 GitHub Pages의 project site 형태로 배포했습니다.

GitHub Pages에는 크게 두 가지 방식이 있습니다.

```text
User site:
https://{username}.github.io/

Project site:
https://{username}.github.io/{repository-name}/
```

현재 repository 이름은 `kevin-blog`입니다. 따라서 사이트는 루트(`/`)가 아니라 `/kevin-blog/` 하위에 배포됩니다.

```text
https://jeongheekevin.github.io/kevin-blog/
```

이 구조에서는 `/blog/`와 `/kevin-blog/blog/`가 다릅니다.

```text
/blog/
→ https://jeongheekevin.github.io/blog/

/kevin-blog/blog/
→ https://jeongheekevin.github.io/kevin-blog/blog/
```

즉, project site에서는 내부 링크를 루트 기준으로 작성하면 운영 환경에서 깨질 수 있습니다.

## GitHub Actions 배포 설정

GitHub Actions workflow는 다음처럼 구성했습니다.

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

여기서 한 번 배포 오류도 발생했습니다.

```text
Error: Creating Pages deployment failed
Error: HttpError: Not Found
Ensure GitHub Pages has been enabled
```

이 에러는 빌드 실패가 아니었습니다. repository의 GitHub Pages 설정에서 배포 source가 GitHub Actions로 활성화되지 않아서 발생한 문제였습니다.

GitHub repository 설정에서 다음 항목을 변경해 해결했습니다.

```text
Settings
→ Pages
→ Build and deployment
→ Source: GitHub Actions
```

이후 GitHub Actions의 build와 deploy 단계는 정상적으로 완료되었습니다.

## base path 설정

GitHub Pages project site에 배포하려면 Astro의 `base` 설정이 필요했습니다.

처음에는 다음처럼 고정값으로 설정했습니다.

```js
export default defineConfig({
	site: 'https://jeongheekevin.github.io',
	base: '/kevin-blog/',
	integrations: [mdx(), sitemap()],
});
```

이 설정은 운영 환경에는 맞습니다. 실제 배포 주소가 다음과 같기 때문입니다.

```text
https://jeongheekevin.github.io/kevin-blog/
```

하지만 로컬 테스트에서는 혼란이 생겼습니다. 로컬 개발 서버에서는 보통 다음 주소로 확인합니다.

```text
http://localhost:4321/
http://localhost:4321/blog/
http://localhost:4321/about/
```

반면 production에서는 다음 주소가 기준입니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

즉, 로컬과 운영의 base path가 다릅니다.

그래서 최종적으로는 환경에 따라 `base`를 분리했습니다.

```js
const isProd = process.env.NODE_ENV === 'production';

export default defineConfig({
	site: 'https://jeongheekevin.github.io',
	base: isProd ? '/kevin-blog/' : '/',
	integrations: [mdx(), sitemap()],
});
```

이렇게 하면 로컬에서는 루트 경로(`/`)를 사용합니다.

```text
http://localhost:4321/blog/
```

반면 production build에서는 GitHub Pages project site 경로를 사용합니다.

```text
https://jeongheekevin.github.io/kevin-blog/blog/
```

여기서 `site`와 `base`의 역할은 다릅니다.

```text
site = 사이트의 도메인
base = 배포되는 하위 경로
```

처음에는 다음처럼 `site`에 전체 경로를 넣으면 된다고 생각할 수 있습니다.

```js
site: 'https://jeongheekevin.github.io/kevin-blog/'
```

하지만 이것만으로는 내부 링크 문제가 해결되지 않았습니다. `site`는 sitemap, RSS, canonical URL 같은 절대 URL 생성에 주로 사용됩니다. 반면 project site에서 실제 asset path와 내부 링크 prefix에 영향을 주는 값은 `base`입니다.

따라서 GitHub Pages project site에서는 `site`와 `base`를 분리하고, 로컬 테스트 편의성을 위해 production 여부에 따라 `base`를 다르게 설정했습니다.

## 빌드와 배포는 정상이었다

이 문제를 혼동했던 이유는 빌드와 배포가 모두 성공했기 때문입니다.

`npm run build`도 성공했고, GitHub Actions의 deploy 단계도 성공했습니다. 실제로 다음 페이지들은 정상적으로 열렸습니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

즉, 문제는 “배포가 안 됐다”가 아니었습니다.

정적 파일은 생성되었고, Pages에도 올라갔습니다. 진짜 문제는 생성된 페이지 안에 들어 있는 링크가 잘못된 경로를 가리킨다는 점이었습니다.

## 실제 원인

문제는 `src/pages/blog/index.astro`의 개별 글 링크였습니다.

기존 코드는 다음과 같았습니다.

```astro
<a href={`/blog/${post.id}/`}>
```

이 링크는 항상 사이트 루트 기준으로 `/blog/{slug}/`를 생성합니다.

로컬에서는 정상처럼 보입니다.

```text
http://localhost:4321/blog/markdown-style-guide/
```

하지만 GitHub Pages project site 운영 환경에서는 사이트가 `/kevin-blog/` 아래에 있습니다. 따라서 운영에서 필요한 링크는 다음입니다.

```text
/kevin-blog/blog/markdown-style-guide/
```

그런데 기존 코드는 `/blog/markdown-style-guide/`를 만들었습니다.

결과적으로 브라우저는 다음 주소로 이동했습니다.

```text
https://jeongheekevin.github.io/blog/markdown-style-guide/
```

이 경로에는 페이지가 없으므로 404가 발생했습니다.

## HeaderLink에 대한 오해

추가로 혼동했던 부분은 `HeaderLink.astro`였습니다.

`HeaderLink.astro`에는 다음과 같은 코드가 있었습니다.

```astro
const pathname = Astro.url.pathname.replace(import.meta.env.BASE_URL, '');
```

이 코드를 보고 처음에는 `HeaderLink`가 base path를 알아서 처리한다고 생각했습니다. 하지만 이 코드는 현재 경로에서 `BASE_URL`을 제거해서 active 상태를 계산하기 위한 코드입니다.

즉, 다음 역할만 합니다.

```text
현재 URL이 어떤 메뉴에 해당하는지 판단한다.
```

하지만 실제 `href` 값을 바꿔주지는 않습니다.

다시 말해 `HeaderLink`에 다음처럼 넘기면:

```astro
<HeaderLink href="/blog">Blog</HeaderLink>
```

운영 환경에서도 링크는 그대로 `/blog`입니다.

따라서 project site에서 안전하게 동작하려면 `href` 자체를 base-aware하게 만들어야 합니다.

```astro
const BASE = import.meta.env.BASE_URL;
```

```astro
<HeaderLink href={`${BASE}blog/`}>Blog</HeaderLink>
```

## 해결

Astro는 `import.meta.env.BASE_URL` 값을 제공합니다. 이 값은 `astro.config.mjs`의 `base` 설정을 반영합니다.

따라서 블로그 목록 페이지에서 다음 변수를 추가했습니다.

```astro
const BASE = import.meta.env.BASE_URL;
```

그리고 개별 글 링크를 다음처럼 수정했습니다.

```astro
<a href={`${BASE}blog/${post.id}/`}>
```

수정 후 구조는 다음과 같습니다.

```astro
const posts = (await getCollection('blog')).sort(
	(a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf(),
);

const BASE = import.meta.env.BASE_URL;
```

```astro
{
	posts.map((post) => (
		<li>
			<a href={`${BASE}blog/${post.id}/`}>
				<h4 class="title">{post.data.title}</h4>
				<p class="date">
					<FormattedDate date={post.data.pubDate} />
				</p>
			</a>
		</li>
	))
}
```

Header도 같은 원칙으로 수정했습니다.

```astro
---
import { SITE_TITLE } from '../consts';
import HeaderLink from './HeaderLink.astro';

const BASE = import.meta.env.BASE_URL;
---

<header>
	<nav>
		<h2><a href={BASE}>{SITE_TITLE}</a></h2>
		<div class="internal-links">
			<HeaderLink href={BASE}>Home</HeaderLink>
			<HeaderLink href={`${BASE}about/`}>About</HeaderLink>
			<HeaderLink href={`${BASE}blog/`}>Blog</HeaderLink>
		</div>
	</nav>
</header>
```

이제 운영 환경에서는 다음 링크가 생성됩니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/about/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/blog/markdown-style-guide/
```

## 로컬 테스트에서 주의할 점

이번 세팅에서는 로컬과 production의 `base`를 다르게 설정했습니다.

```js
const isProd = process.env.NODE_ENV === 'production';

export default defineConfig({
	site: 'https://jeongheekevin.github.io',
	base: isProd ? '/kevin-blog/' : '/',
});
```

이 설정 때문에 로컬 개발 환경에서는 다음 주소를 기준으로 확인합니다.

```text
http://localhost:4321/
http://localhost:4321/blog/
http://localhost:4321/about/
```

반면 production에서는 다음 주소를 기준으로 확인합니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

즉, 로컬에서 `/blog/`가 정상이라고 해서 운영에서도 `/blog/`가 정상인 것은 아닙니다. 운영에서는 `/kevin-blog/blog/`가 정상 경로입니다.

이 차이를 놓치면 로컬에서는 문제가 없는데 운영에서만 404가 발생할 수 있습니다.

특히 `npx serve dist`로 빌드 결과물을 확인할 때도 주의해야 합니다.

```bash
npm run build
npx serve dist
```

이 방식은 `dist` 디렉터리를 로컬 서버의 루트(`/`)에 올려서 보여줍니다. GitHub Pages가 project site를 `/kevin-blog/` 아래에 mount하는 구조를 완전히 재현하지는 않습니다.

따라서 최종 검증은 반드시 운영 URL 기준으로 해야 합니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

## 캐시로 인한 혼동

배포 후에도 이전 글 목록이나 이전 링크가 잠시 보이는 문제가 있었습니다.

이 경우 원인은 코드가 아니라 캐시였습니다. GitHub Pages는 정적 파일을 CDN을 통해 제공하기 때문에 배포 직후에는 이전 HTML이 잠시 보일 수 있습니다.

확인할 때는 캐시를 우회했습니다.

```text
https://jeongheekevin.github.io/kevin-blog/blog/?v=deploy-check
```

또는 브라우저에서 강력 새로고침을 사용했습니다.

```text
Cmd + Shift + R
```

이 문제 때문에 운영 배포 확인 시에는 단순히 브라우저 새로고침만 믿지 않는 편이 안전합니다.

## 검증

수정 후 다음 순서로 확인했습니다.

```bash
npm run build
git add .
git commit -m "Fix GitHub Pages base path links"
git push
```

GitHub Actions가 성공한 뒤 운영에서 다음 경로를 확인했습니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/about/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/blog/astro-github-pages-base-path/
```

확인 기준은 단순히 페이지가 열리는지가 아니었습니다.

다음 두 가지를 함께 확인했습니다.

```text
1. 직접 URL 접근이 되는가
2. 화면에서 링크를 클릭했을 때도 올바른 URL로 이동하는가
```

정적 사이트에서는 첫 번째만 확인하면 부족합니다. 페이지 자체는 존재해도 내부 링크가 잘못 생성될 수 있기 때문입니다.

## 재발 방지

이번 문제를 다시 만들지 않기 위해 다음 규칙을 정했습니다.

- project site에서는 내부 링크를 `/blog/...`처럼 직접 작성하지 않습니다.
- 내부 링크는 `import.meta.env.BASE_URL`을 기준으로 생성합니다.
- `HeaderLink` 같은 컴포넌트가 active state만 처리하는지, 실제 href까지 처리하는지 구분합니다.
- 배포 후에는 직접 URL 접근뿐 아니라 화면 클릭 경로까지 확인합니다.
- 삭제/수정 후 운영 페이지 확인 시 캐시 우회 URL을 사용합니다.

향후 내부 링크가 늘어나면 helper 함수를 두는 편이 낫습니다.

```ts
export function withBase(path: string): string {
	const base = import.meta.env.BASE_URL;
	return `${base}${path.replace(/^\/+/, '')}`;
}
```

그럼 링크 생성 시 매번 `BASE`를 직접 붙이지 않아도 됩니다.

```astro
<a href={withBase(`/blog/${post.id}/`)}>
```

## 정리

이번 문제의 핵심은 GitHub Pages 배포 실패가 아니었습니다.

빌드도 성공했고, 배포도 성공했습니다. 문제는 운영 환경의 mount path를 고려하지 않은 내부 링크였습니다.

GitHub Pages project site는 repository 이름을 포함한 하위 경로에 사이트를 배포합니다.

```text
/kevin-blog/
```

하지만 일부 코드가 루트 기준 링크를 생성하고 있었습니다.

```text
/blog/...
```

이 차이 때문에 운영에서만 404가 발생했습니다.

정적 사이트에서 배포가 성공했다는 사실은 충분하지 않습니다. 실제 페이지 안에 생성된 링크가 운영 URL 기준으로 올바른지도 반드시 확인해야 합니다.