---
title: "GitHub Pages에 Astro 블로그를 세팅하며 겪은 path와 cache 문제"
description: "Astro 블로그를 GitHub Pages project site에 배포하면서 로컬과 production의 path 차이, BASE_URL 처리, GitHub Pages 캐시 문제를 정리했습니다."
pubDate: "Jun 03 2026"
----------------------

## TL;DR

Astro 블로그를 GitHub Pages project site에 배포할 때 핵심 문제는 빌드나 배포 자체가 아니라 **경로 기준**이었습니다.

GitHub Pages project site는 사이트를 루트(`/`)가 아니라 repository 이름 하위 경로에 배포합니다.

```text
https://jeongheekevin.github.io/kevin-blog/
```

따라서 로컬에서는 정상처럼 보이는 링크가 production에서는 깨질 수 있습니다.

```text
로컬
/blog/

production
/kevin-blog/blog/
```

이번 문제의 직접 원인은 내부 링크를 루트 기준으로 작성한 것이었습니다.

```astro
<a href={`/blog/${post.id}/`}>
```

production에서는 `import.meta.env.BASE_URL`을 기준으로 링크를 생성해야 했습니다.

```astro
<a href={`${BASE}blog/${post.id}/`}>
```

또한 배포 후 삭제한 글이 계속 보이는 문제가 있었는데, 실제 repository나 build 문제가 아니라 GitHub Pages/CDN/브라우저 캐시로 인한 혼동이었습니다.

이번 세팅에서 정리한 기준은 다음과 같습니다.

* Astro는 Markdown / MDX 기반 글 관리를 위해 선택했습니다.
* GitHub Actions는 `npm run build` 결과물인 `dist`를 생성하고 배포하기 위해 사용했습니다.
* GitHub Pages는 정적 파일을 제공하는 hosting 역할만 담당합니다.
* project site에서는 `base` 설정과 내부 링크 생성 방식을 반드시 확인해야 합니다.
* 배포 검증 시 직접 URL 접근과 화면 클릭 경로를 모두 확인해야 합니다.
* 배포 직후에는 GitHub Pages/CDN/브라우저 캐시 때문에 이전 HTML이 보일 수 있습니다.

## 1. 세팅 목표

개인 기술 블로그와 포트폴리오를 운영하기 위해 Astro 기반 블로그를 GitHub Pages에 배포했습니다.

목표는 단순히 정적 페이지 하나를 올리는 것이 아니었습니다. 앞으로 기술 글을 계속 누적할 수 있는 구조가 필요했습니다.

이번 세팅에서 원한 조건은 다음과 같았습니다.

* Markdown / MDX 기반으로 글을 작성할 수 있어야 했습니다.
* 글 목록이 자동으로 생성되어야 했습니다.
* 공통 레이아웃, header, footer를 재사용할 수 있어야 했습니다.
* GitHub repository에 source code만 유지하고 싶었습니다.
* build 결과물은 직접 commit하지 않고 자동으로 배포하고 싶었습니다.
* 최종 사이트는 GitHub Pages로 외부에 공개하고 싶었습니다.

최종 배포 주소는 다음 형태였습니다.

```text
https://jeongheekevin.github.io/kevin-blog/
```

## 2. 기술 선택과 배포 구조

### 2.1 왜 순수 HTML 대신 Astro를 선택했는가

기술 블로그는 순수 HTML, CSS, JavaScript만으로도 만들 수 있습니다.

GitHub Pages는 정적 파일을 그대로 배포할 수 있기 때문에, 단순한 페이지 몇 개라면 별도의 빌드 도구 없이도 충분합니다.

예를 들면 다음과 같은 구조도 가능합니다.

```text
index.html
about.html
blog/
├── first-post.html
└── second-post.html
style.css
```

이 방식의 장점은 명확합니다.

* build step이 없습니다.
* GitHub Pages의 `Deploy from a branch` 방식으로 바로 배포할 수 있습니다.
* 설정할 것이 적습니다.
* 경로 구조도 상대적으로 단순합니다.

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

처음에는 문제가 없어 보이지만 글이 늘어나면 반복 작업이 많아집니다.

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

순수 HTML 방식은 단순하지만, 글이 늘어날수록 콘텐츠 관리 비용이 커집니다.

반면 Astro는 Markdown / MDX로 글을 작성하고, build 시 정적 HTML로 변환할 수 있습니다.

```text
Markdown / MDX
→ Astro build
→ HTML
→ GitHub Pages
```

최종 결과물은 여전히 정적 사이트지만, 작성과 관리 방식은 블로그 운영에 더 적합했습니다.

그래서 이번 블로그는 순수 HTML이 아니라 Astro로 구성했습니다.

### 2.2 왜 GitHub Actions로 배포했는가

GitHub Pages 설정에는 `Deploy from a branch` 방식도 있습니다.

단순 HTML, CSS, JavaScript만 있는 정적 사이트라면 특정 branch의 `/root` 또는 `/docs` 디렉터리를 그대로 배포해도 충분합니다.

하지만 Astro는 source code를 그대로 배포하는 구조가 아닙니다. 실제로 배포해야 하는 것은 `npm run build` 결과물인 `dist` 디렉터리입니다.

```text
Astro source
→ npm run build
→ dist
→ GitHub Pages
```

물론 `dist` 결과물을 별도 branch에 올리는 방식도 가능합니다.

```text
main branch
→ Astro source

gh-pages branch
→ dist output
```

하지만 이 방식은 관리 포인트가 늘어납니다.

* source branch와 deploy branch를 따로 관리해야 합니다.
* `dist` 결과물을 직접 commit하거나 별도 배포 도구를 사용해야 합니다.
* build 결과물이 source history에 섞일 가능성이 있습니다.
* 배포 실패 시 어느 단계에서 실패했는지 추적이 불편합니다.

그래서 GitHub Actions에서 build를 수행하고, 생성된 artifact를 GitHub Pages에 배포하는 방식으로 구성했습니다.

### 2.3 Source, Artifact, Hosting 책임 분리

이번 구조에서는 역할을 다음처럼 분리했습니다.

```text
GitHub Repository = source code 저장소
GitHub Actions    = build/deploy runner
dist              = build artifact
GitHub Pages      = static hosting
```

여기서 GitHub Actions는 웹 요청을 처리하는 서버가 아닙니다.

GitHub Actions는 push 이후 정해진 workflow를 실행하는 CI runner입니다. 실제 정적 파일을 제공하는 것은 GitHub Pages입니다.

즉, GitHub Actions를 사용한 이유는 서버처럼 쓰기 위해서가 아니라, source code와 build artifact를 분리하기 위해서였습니다.

배포 흐름은 다음과 같습니다.

```text
1. main branch push
2. GitHub Actions runner 실행
3. dependency 설치
4. npm run build 실행
5. dist 디렉터리 생성
6. Pages artifact 업로드
7. GitHub Pages에 배포
```

이 구조를 사용하면 repository에는 source code만 남기고, build output은 CI 환경에서 재현 가능하게 생성할 수 있습니다.

## 3. GitHub Pages project site의 경로 구조

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

즉, project site에서는 내부 링크를 루트 기준으로 작성하면 production에서 깨질 수 있습니다.

이 차이가 이번 문제의 핵심이었습니다.

## 4. 문제 1: 로컬에서는 되는데 production에서는 404가 났다

### 4.1 로컬 경로와 production 경로가 달랐다

로컬 개발 서버에서는 다음 경로가 정상적으로 동작했습니다.

```text
http://localhost:4321/
http://localhost:4321/blog/
http://localhost:4321/about/
```

하지만 production에서는 다음 경로가 기준입니다.

```text
https://jeongheekevin.github.io/kevin-blog/
https://jeongheekevin.github.io/kevin-blog/blog/
https://jeongheekevin.github.io/kevin-blog/about/
```

즉, 로컬에서는 `/blog/`가 맞지만 production에서는 `/kevin-blog/blog/`가 맞습니다.

문제는 일부 내부 링크가 이 production mount path를 반영하지 못했다는 점이었습니다.

### 4.2 root-relative link가 문제였다

블로그 목록 페이지에서 개별 글 링크는 다음처럼 생성되고 있었습니다.

```astro
<a href={`/blog/${post.id}/`}>
```

이 코드는 항상 사이트 루트 기준으로 `/blog/{slug}/`를 생성합니다.

로컬에서는 정상처럼 보였습니다.

```text
http://localhost:4321/blog/markdown-style-guide/
```

하지만 GitHub Pages project site에서는 사이트가 `/kevin-blog/` 아래에 배포됩니다.

따라서 production에서 필요한 링크는 다음입니다.

```text
/kevin-blog/blog/markdown-style-guide/
```

그런데 기존 코드는 다음 경로를 만들었습니다.

```text
/blog/markdown-style-guide/
```

결과적으로 브라우저는 다음 주소로 이동했습니다.

```text
https://jeongheekevin.github.io/blog/markdown-style-guide/
```

이 경로에는 페이지가 없으므로 404가 발생했습니다.

즉, build와 deploy는 성공했지만, 생성된 HTML 안의 링크가 production mount path를 무시하고 있었습니다.

### 4.3 import.meta.env.BASE_URL로 링크를 생성했다

Astro는 `import.meta.env.BASE_URL` 값을 제공합니다.

이 값은 `astro.config.mjs`의 `base` 설정을 반영합니다.

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

이제 production에서는 다음처럼 정상 링크가 생성됩니다.

```text
https://jeongheekevin.github.io/kevin-blog/blog/markdown-style-guide/
```

### 4.4 HeaderLink는 href를 보정하지 않았다

추가로 혼동했던 부분은 `HeaderLink.astro`였습니다.

`HeaderLink.astro`에는 다음과 같은 코드가 있었습니다.

```astro
const pathname = Astro.url.pathname.replace(import.meta.env.BASE_URL, '');
```

이 코드를 보고 처음에는 `HeaderLink`가 base path를 알아서 처리한다고 생각했습니다.

하지만 이 코드는 현재 경로에서 `BASE_URL`을 제거해서 active 상태를 계산하기 위한 코드입니다.

즉, 다음 역할만 합니다.

```text
현재 URL이 어떤 메뉴에 해당하는지 판단한다.
```

하지만 실제 `href` 값을 바꿔주지는 않습니다.

다시 말해 `HeaderLink`에 다음처럼 넘기면:

```astro
<HeaderLink href="/blog">Blog</HeaderLink>
```

production에서도 링크는 그대로 `/blog`입니다.

따라서 `href` 자체를 base-aware하게 만들어야 했습니다.

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

이제 Header 메뉴 클릭 시에도 `/kevin-blog/` 하위 경로로 이동합니다.

## 5. 문제 2: 삭제한 글이 계속 보였다

### 5.1 repository에서는 삭제됐지만 운영 페이지에는 남아 있었다

샘플 글 Markdown 파일을 삭제했는데도 production의 `/blog/` 페이지에는 이전 글 목록이 계속 보였습니다.

처음에는 삭제가 commit되지 않았거나, build가 이전 파일을 참조하고 있다고 의심했습니다.

하지만 GitHub repository의 `src/content/blog`에는 새 글 하나만 남아 있었습니다.

로컬 build 결과도 확인했습니다.

```bash
npm run build
find dist/blog -maxdepth 2 -type f | sort
```

기대했던 결과는 다음과 같았습니다.

```text
dist/blog/astro-github-pages-base-path/index.html
dist/blog/index.html
```

repository와 build output이 정상이라면 코드 문제는 아닙니다.

### 5.2 원인은 GitHub Pages/브라우저 캐시였다

실제 원인은 캐시였습니다.

GitHub Pages는 정적 파일을 CDN을 통해 제공합니다. 또한 브라우저도 HTML을 캐시할 수 있습니다.

그 결과 배포 직후에는 repository와 build output이 바뀌었더라도, 사용자는 이전 HTML을 잠시 볼 수 있습니다.

이번 경우에도 삭제된 Markdown 파일이 다시 생성된 것이 아니라, 이전 `/blog/` HTML을 보고 있었던 것이었습니다.

### 5.3 캐시 우회 URL로 확인했다

배포 직후에는 캐시를 우회해서 확인했습니다.

```text
https://jeongheekevin.github.io/kevin-blog/blog/?v=deploy-check
```

브라우저에서는 강력 새로고침도 사용했습니다.

```text
Cmd + Shift + R
```

이후 최신 HTML이 반영된 것을 확인했습니다.

정적 사이트 배포에서는 캐시 때문에 다음 두 가지를 구분해야 합니다.

```text
repository 상태
build output 상태
사용자가 보고
```
