# 기술블로그 작성 가이드 (AI 에이전트용)

이 문서는 **AI 에이전트가 이 블로그에 글을 작성·수정할 때 반드시 따르는 규칙**이다.
새 글을 쓰기 전, 그리고 발행 전에 이 문서를 기준으로 검토한다.

- **스택**: Jekyll + GitHub Pages (사용자 페이지: `https://opaqe.github.io/`)
- **주제**: 웹 / 프론트엔드 (JavaScript, TypeScript, React, CSS, 브라우저, 빌드 도구 등)
- **언어 정책**: 본문은 **한국어 중심**, 기술 용어·코드·고유명사는 **영어 원문 유지** (아래 [언어 정책](#71-언어-정책) 참고)
- **대상 독자**: 같은 분야의 실무 개발자. "왜"와 "어떻게"를 설명하되, 기초 개념은 과하게 늘어놓지 않는다.

---

## 1. 저장소 구조

```
ayden.github.io/
├── _config.yml            # 사이트 전역 설정 (수정 시 jekyll 재시작 필요)
├── Gemfile                # 의존성 (github-pages gem)
├── index.md               # 홈(글 목록) 페이지
├── about.md               # 소개 페이지
├── _posts/                # ✅ 일반·프론트엔드 글 (categories는 front matter)
│   └── YYYY-MM-DD-title.md
├── spring/
│   └── _posts/            # ✅ Spring 글 → 폴더가 categories: spring 자동 부여
│       └── YYYY-MM-DD-title.md
├── _drafts/               # 초안 (날짜 없음, 빌드 기본 제외)
│   └── post-template.md   # 새 글 복사용 템플릿
├── assets/
│   └── img/               # 이미지. 글마다 하위 폴더 권장
│       └── <post-slug>/
└── AGENTS.md              # (이 문서)
```

일반 글은 `_posts/`에, **Spring 글은 `spring/_posts/`에** 둔다(폴더명이 카테고리가 됨). 작성 중인 초안은 `_drafts/`에 둔다.

---

## 2. 새 글 작성 워크플로우 (가장 중요)

1. **주제 확정** — 한 글은 하나의 핵심 주제만 다룬다.
2. **파일 생성** — 일반 글은 `_posts/YYYY-MM-DD-<slug>.md`, **Spring 글은 `spring/_posts/YYYY-MM-DD-<slug>.md`** ([네이밍 규칙](#3-파일-이름-규칙))
   - 초안 단계면 `_drafts/post-template.md`를 복사해 `_drafts/<slug>.md`로 시작.
3. **Front Matter 작성** — [5장 템플릿](#5-front-matter)을 채운다. `date`는 작성 시점 기준.
4. **본문 작성** — [7장 스타일 가이드](#7-글쓰기-스타일-가이드)를 따른다.
5. **로컬 미리보기** — [10장 명령](#10-로컬-미리보기)으로 렌더링 확인.
6. **[발행 전 체크리스트](#9-발행-전-체크리스트) 검토.**
7. **커밋** — 한 글 = 의미 있는 단위로 커밋. 메시지 예: `post: React 동시성 렌더링 정리`.

> 초안을 발행할 때는 `_drafts/`에서 `_posts/`로 옮기고 파일명 앞에 `YYYY-MM-DD-`를 붙인다.

---

## 3. 파일 이름 규칙

```
_posts/YYYY-MM-DD-slug-in-kebab-case.md
```

- 날짜는 **발행일**, `kebab-case`(소문자+하이픈).
- slug는 **영문**으로. 한글 파일명은 URL이 깨질 수 있어 금지.
- 예: `2026-06-19-react-useeffect-cleanup.md`, `2026-07-02-css-container-queries.md`

URL은 `_config.yml`의 `permalink: /:year/:month/:day/:title/` 규칙을 따른다.

- **카테고리 폴더**: Spring 글은 `spring/_posts/`에 두면 폴더명이 `categories: spring`으로 자동 지정된다. 일반 글은 `_posts/`. ⚠️ `_posts/spring/`처럼 `_posts` **안에** 넣으면 카테고리가 안 붙는다 — 반드시 `spring/_posts/`(카테고리 폴더가 `_posts` 위).

---

## 4. 카테고리 & 태그

**categories(대분류, 1개)** + **tags(세부 키워드, 2~5개)** 조합을 사용한다. 모두 **소문자**.

| 구분 | 사용할 값 (이 중에서 고른다) |
|---|---|
| **categories** (front matter) | `frontend` · `javascript` · `css` · `web` · `backend` · `tooling` · `retrospective` |
| **폴더 자동 카테고리** | `spring/_posts/` → `spring` (front matter에 `categories`를 쓰지 않음) |
| **tags** (예시) | `react` `vue` `typescript` `vite` `webpack` `tailwind` `dom` `browser` `performance` `accessibility` `http` `testing` `state-management` |

- 새 태그를 만들기 전에 위 목록·기존 글에서 재사용 가능한지 먼저 확인한다 (난립 방지).
- categories는 가능하면 **1개**로 유지(URL·분류 단순화).
- **Spring 글은 폴더로 분류**: `spring/_posts/`에 두면 자동으로 `spring` 카테고리가 붙는다. front matter에 `categories: spring`을 **추가로 쓰지 않는다**(중복 부여 방지).
- **메뉴 연동**: 상단 `Spring` 메뉴(`spring.md`)는 `spring` 카테고리 글을, `블로그` 메뉴(`blog.md`)는 전체 글을 모은다. 메뉴 구성은 `_config.yml`의 `header_pages`.

---

## 5. Front Matter

모든 글 최상단에 YAML front matter를 둔다. 복붙용 템플릿:

```yaml
---
layout: post
title: "핵심 키워드를 담은 한 줄 제목"
date: 2026-06-19 14:00:00 +0900
categories: frontend          # Spring 글이면 이 줄 빼고 파일을 spring/_posts/ 에 저장
tags: [react, performance]
description: "검색 결과·SNS 공유에 노출되는 1~2문장 한국어 요약."
# image: /assets/img/2026-06-19-slug/cover.png   # OG 썸네일(선택)
---
```

| 필드 | 필수 | 규칙 |
|---|---|---|
| `layout` | ✅ | 항상 `post` |
| `title` | ✅ | 따옴표로 감싼다. 콜론(`:`)·따옴표 포함 시 특히 필수 |
| `date` | ✅ | `YYYY-MM-DD HH:MM:SS +0900` (KST) |
| `categories` | 조건부 | 일반 글은 [4장](#4-카테고리--태그)에서 1개. **Spring 글은 `spring/_posts/` 폴더가 부여하므로 생략** |
| `tags` | ✅ | [4장](#4-카테고리--태그)에서 2~5개, 배열 |
| `description` | ✅ | SEO 요약. 한국어 1~2문장. (jekyll-seo-tag가 사용) |
| `image` | 선택 | 대표/OG 이미지 경로 |

---

## 6. 본문 구조 템플릿

```markdown
한 줄 도입: 이 글에서 무엇을, 왜 다루는지. (front matter 바로 아래, 제목 없이)

## 배경 / 문제 상황
무엇이 문제였는지, 어떤 맥락인지.

## 핵심 내용
설명 + 코드. 필요하면 소제목(###)으로 쪼갠다.

## 정리
요점 3줄 또는 불릿. 한계나 다음 단계가 있으면 덧붙인다.

## 참고
- [출처 제목](https://example.com)
```

- 본문 최상위 제목은 **`##`(h2)부터** 시작한다. `#`(h1)은 글 제목(front matter)이 차지하므로 본문에서 쓰지 않는다.
- 제목은 계층을 건너뛰지 않는다 (`##` → `###` → `####`).

---

## 7. 글쓰기 스타일 가이드

### 7.1 언어 정책
- 서술 문장은 **한국어**, 어미는 평서체(`~다`)로 통일.
- 기술 용어·API·라이브러리·코드 식별자는 **영어 원문** 유지: "리액트"(X) → "React"(O), `useEffect`, `Promise`, hydration 등.
- 처음 등장하는 핵심 개념은 `리페인트(repaint)`처럼 **한국어(영어)** 병기 1회 후 이후 통일.
- 직역투·번역기 말투 금지. 자연스러운 한국어 문장으로.

### 7.2 코드 블록
- 항상 **언어를 명시**한 펜스 코드 블록: ` ```tsx `, ` ```js `, ` ```css `, ` ```bash `, ` ```html `.
- 실행 가능하거나 최소 재현 가능한 스니펫을 우선. 핵심과 무관한 줄은 `// ...`로 생략.
- 터미널 명령은 ` ```bash `, 출력과 섞지 않는다.
- 인라인 코드는 백틱: `npm run build`, `className`.

### 7.3 이미지
- 경로: `/assets/img/<post-slug>/<name>.png`. 글마다 하위 폴더로 분리.
- 반드시 **의미 있는 alt 텍스트**: `![렌더링 성능 비교 그래프](/assets/img/.../bench.png)`.
- 스크린샷은 불필요한 여백·민감정보 제거 후 첨부.

### 7.4 링크 · 강조
- 외부 출처는 본문 끝 **참고** 섹션에 모으거나 인라인 링크.
- 강조는 `**굵게**`만 절제해서. 색상·과도한 이탤릭 지양.
- 인용/주의는 `>` 블록인용 사용.

### 7.5 분량·문단
- 한 문단은 2~4문장. 화면에서 읽기 좋게 짧게 끊는다.
- 결론을 먼저(두괄식). "왜 중요한가"를 빼먹지 않는다.

---

## 8. Jekyll / Markdown 주의사항

- **Liquid 충돌**: 본문에 `{{`, `}}`, `{%`, `%}` 같은 문자를 그대로 쓰면 Jekyll이 템플릿으로 해석한다.
  Vue/Angular 보간법, JS 템플릿 리터럴 등을 보여줄 때는 해당 부분을 `{% raw %} ... {% endraw %}`로 감싼다.
- **마크다운 엔진**: kramdown. 표·각주·코드펜스 지원. 줄바꿈은 빈 줄로.
- **수식·HTML**: 필요 시 인라인 HTML 허용되나 최소화.
- `_config.yml`을 바꾸면 로컬 서버를 **재시작**해야 반영된다 (글 수정은 자동 반영).

---

## 9. 발행 전 체크리스트

- [ ] 파일 위치가 올바름 — 일반 글 `_posts/`, Spring 글 `spring/_posts/`
- [ ] 파일명이 `YYYY-MM-DD-slug.md` 형식 (slug 영문 kebab-case)
- [ ] Front matter에 `layout/title/date/tags/description` 존재 (categories는 일반 글만; Spring 글은 폴더가 부여)
- [ ] `categories`/`tags`가 [4장 목록](#4-카테고리--태그) 기준
- [ ] 본문 제목이 `##`부터 시작, 계층 안 건너뜀
- [ ] 모든 코드 블록에 언어 명시
- [ ] 모든 이미지에 alt 텍스트, 경로 정상
- [ ] `description`이 내용을 한 줄로 잘 요약
- [ ] 로컬 미리보기에서 렌더링·링크·이미지 깨짐 없음
- [ ] 맞춤법·번역투 점검

---

## 10. 로컬 미리보기

```bash
bundle install            # 최초 1회 (Ruby + Bundler 필요)
bundle exec jekyll serve  # http://localhost:4000
bundle exec jekyll serve --drafts   # 초안(_drafts)까지 포함해 미리보기
```

---

## 11. 하지 말 것 (Don'ts)

- ❌ 한 글에 여러 주제를 욱여넣기
- ❌ 한글·공백·대문자가 들어간 파일명/slug
- ❌ 언어 없는 코드 블록, alt 없는 이미지
- ❌ AI 티 나는 군더더기 문장("이번 포스팅에서는 ~에 대해 알아보도록 하겠습니다" 류 상투구)
- ❌ 출처 없이 수치·인용 단정
- ❌ `_config.yml`의 `url`/`baseurl` 등 사이트 설정을 글 작성하면서 임의 변경
