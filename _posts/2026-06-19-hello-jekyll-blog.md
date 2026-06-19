---
layout: post
title: "블로그를 시작하며: Jekyll + GitHub Pages 세팅"
date: 2026-06-19 14:00:00 +0900
categories: web
tags: [jekyll, github-pages]
description: "Jekyll과 GitHub Pages로 웹·프론트엔드 기술 블로그를 세팅하고, 글 작성 규칙을 정리했다."
---

웹·프론트엔드 개발 과정에서 배운 것들을 기록하려고 이 블로그를 만들었다. 첫 글은 사용한 스택과 앞으로의 작성 규칙을 짧게 정리한다.

## 스택

- **Jekyll** — 마크다운을 정적 사이트로 빌드하는 생성기
- **GitHub Pages** — `username.github.io` 저장소에 푸시하면 자동 배포
- **Chirpy** — 사이드바·다크모드·TOC·카테고리/태그를 갖춘 기술블로그 테마

## 글은 이렇게 쓴다

새 글은 `_posts/YYYY-MM-DD-slug.md` 형식으로 추가하고, 상단 front matter에 제목·날짜·분류·요약을 채운다. 코드 블록에는 항상 언어를 명시한다.

```bash
bundle exec jekyll serve
# http://localhost:4000 에서 미리보기
```

자세한 작성 규칙은 저장소의 `AGENTS.md`에 정리해 두었다.

## 정리

- Jekyll + GitHub Pages로 별도 서버 없이 블로그를 운영한다.
- 글 작성 규칙은 `AGENTS.md` 한 곳에서 관리한다.
- 다음 글부터는 실제 웹·프론트엔드 주제를 다룬다.
