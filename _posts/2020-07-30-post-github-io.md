---
layout: post
title: GitHub.io 구축하기
hide_title: false
color: rgb(0,0,0)
bootstrap: true
excerpt_separator: <!--more-->
tags: [github.io, Jekyll]
---

GitHub Pages와 Jekyll 테마로 이 블로그를 만든 과정, 그리고 이후 기술 검토 문서를 올리면서 더한 구성입니다.

<!--more-->

2020년에 무작정 시작하며 남긴 기록을 정리하고, 이후 긴 기술 검토 문서를 올리기 위해 더한 구성(문서용 스타일 컴포넌트)을 덧붙였습니다. 당시 "확인해 보자"고 남긴 저장소 이름과 대소문자 문제도 확인했습니다.

원문 2020-07-30 · 보강 2026-10-21 · Jekyll · [Type on Strap](https://github.com/sylhare/Type-on-Strap) (remote theme)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **사용자 사이트는 저장소 이름이 `<사용자명>.github.io`여야 합니다.** 그러면 `https://<사용자명>.github.io`로 게시됩니다.
2. **대소문자는 문제가 아니었습니다.** GitHub 사용자명은 대소문자를 구분하지 않고, Pages 주소는 항상 소문자로 쓰입니다. 당시 실패의 원인은 저장소 이름이 사용자명과 달랐던 것으로 보입니다.
3. **테마는 `remote_theme`으로 받아 씁니다.** 테마 파일을 복사하지 않아도 되고, 필요한 스타일만 `_sass`에 더해 덮어씁니다.
4. **긴 검토 문서를 위한 컴포넌트를 더했습니다.** 요약 카드, 목차, 표 스크롤, 콜아웃, 상태 배지를 kramdown 속성으로 씁니다.

</div>

## 1. 저장소 만들기 {#repo}

GitHub Pages 사용자 사이트를 만들려면 public 저장소를 하나 만듭니다.

{% include aligner.html images="Screen Shot 2020-07-30 at 6.19.08 PM.png" column=1 %}

- 저장소 이름은 **`<사용자명>.github.io`** 형식이어야 합니다. 사용자명이 `caimask`이므로 `caimask/caimask.github.io`입니다.
- 이름이 이 형식이 아니면 사용자 사이트가 아니라 프로젝트 사이트(`https://<사용자명>.github.io/<저장소명>/`)로 취급됩니다.

> **당시 남긴 질문: 대소문자가 섞이면 안 되는가**
>
> 처음에는 사용자명이 대소문자가 섞인 `MyungSikChae`였고, 몇 번 실패한 끝에 소문자 `caimask`로 바꿨습니다. GitHub 문서 기준으로 Pages 주소는 사용자명을 소문자로 바꿔 쓰고, 사용자명 자체는 대소문자를 구분하지 않습니다. 그래서 대소문자가 섞인 사용자명도 저장소 이름만 `<사용자명>.github.io`로 맞추면 동작합니다. 당시 실패는 저장소 이름이 사용자명과 일치하지 않았던 것이 원인이었을 가능성이 큽니다.
{: .callout .info}

## 2. 테마 적용 {#theme}

`_config.yml`에서 `remote_theme`으로 GitHub에 있는 테마를 지정합니다. GitHub Pages가 빌드할 때 테마를 받아 오므로 테마 파일을 저장소에 둘 필요가 없습니다.

```yaml
remote_theme: sylhare/Type-on-Strap
plugins: [jekyll-paginate, jekyll-seo-tag, jekyll-feed]
```

글은 `_posts/YYYY-MM-DD-제목.md` 형식으로 두고, 파일 맨 위에 front matter를 씁니다. front matter가 없는 파일은 글로 처리되지 않습니다.

```yaml
---
layout: post
title: Apache Kafka
tags: [Kafka, Kinesis]
---
```

## 3. 검토 문서용 컴포넌트 {#components}

2026년에 Iceberg 적재 구조를 검토한 긴 문서들을 올리면서, 위에서 아래로 읽기 쉽도록 몇 가지 스타일을 더했습니다. 테마의 색상 변수를 그대로 써서 라이트·다크 모드를 함께 따릅니다.

```
_sass/_review.scss       ← 컴포넌트 스타일
assets/css/main.scss     ← @import "type-on-strap"; @import "review";
```

<div class="table-wrap" markdown="1">

| 컴포넌트 | 쓰는 법 | 용도 |
| --- | --- | --- |
| 메타 줄 | 문단 뒤에 `{: .meta}` | 작성일, 버전, 연관 문서 |
| 요약 카드 | `<div class="summary" markdown="1">` | 문서 맨 앞의 결론 |
| 목차 | `<div class="toc" markdown="1">` | 장 이동 |
| 질문 | 문단 뒤에 `{: .q}` | 검토 중에 받은 질문 |
| 콜아웃 | 인용문 뒤에 `{: .callout .warn}` (`ok`, `warn`, `bad`, `info`) | 주의, 결론 |
| 표 | `<div class="table-wrap" markdown="1">`로 감쌈 | 넓은 표를 가로 스크롤 |
| 상태 배지 | `<span class="pill ok">양호</span>` | 평가 결과 표시 |

</div>

`markdown="1"`은 kramdown에게 HTML 블록 안의 내용도 마크다운으로 처리하라고 알려 줍니다. 빠뜨리면 블록 안의 표와 목록이 그대로 텍스트로 나옵니다.

* * *

## 4. 참고 {#refs}

- [GitHub Pages: Creating a GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site)
- [About GitHub Pages: Types of sites](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#types-of-github-pages-sites)
- 테마 적용 방법 - [jekyll remote theme](https://dreamgonfly.github.io/blog/jekyll-remote-theme/)
- 단계적 적용 방법 - [Github.io 만들기](https://blog.naver.com/tty4032/221493608079)
