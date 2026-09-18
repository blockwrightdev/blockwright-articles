# blockwright-articles

Articles for **blockwright.dev**, as plain Markdown. The site reads this repository and renders them; nothing here is built or deployed.

Publishing an article is a commit to this repository. No site build, no deploy, no version bump.

## Layout

```
ru/
  index.json          the menu: which articles exist, in what order
  <slug>.md           one article
en/
  index.json
  <slug>.md
```

One directory per language. An article exists in a language or it does not — there is no fallback and no machine translation. A language with no articles renders an empty state, not someone else's language.

## Adding an article

1. Write `<lang>/<slug>.md` with the front matter below.
2. Add an entry at the **top** of `<lang>/index.json` — the list is newest first.
3. Commit and push. That is the whole publication process.

### Front matter

```yaml
---
title: Тестируем смарт-контракты на Hardhat 3
slug: hardhat-3-testing
lang: ru
date: '2026-09-15'
description: One or two sentences. Shown on the card and used as the meta description.
canonical: https://habr.com/ru/articles/1082316/
---
```

| Field | Required | Notes |
| --- | --- | --- |
| `title` | yes | What the reader sees. Not the slug |
| `slug` | yes | Must equal the filename without `.md` |
| `lang` | yes | `ru` or `en`, matching the directory |
| `date` | yes | `YYYY-MM-DD`, quoted. The publication date, not the day you imported it |
| `description` | yes | Card text and meta description. Keep under ~200 characters |
| `canonical` | no | Where it was **first** published. Set it when the article also lives on Habr or Medium — see below |

### The menu file

```json
{
  "version": 1,
  "lang": "ru",
  "articles": [
    {
      "slug": "hardhat-3-testing",
      "title": "Тестируем смарт-контракты на Hardhat 3",
      "date": "2026-09-15",
      "description": "…"
    }
  ]
}
```

The list is ordered, newest first, and that order is what the site shows — it does not sort by date on your behalf. An article whose file exists but is missing from `index.json` is not published; that is the intended way to stage a draft.

Title, date and description are repeated here on purpose: the list costs one request instead of one per article. When you edit an article's title, edit both.

## Slugs

Lowercase Latin letters, digits and hyphens. No language prefix — the directory already says the language. A slug is permanent: it is the URL, and changing it breaks every link to the article.

The same slug may exist in both languages when it is the same article translated; it does not have to.

## Markdown

Headings from `##` down — `#` is the title and comes from the front matter, not from the body. Paragraphs, **bold**, _italic_, `inline code`, fenced code blocks, links, images, bullet and numbered lists, tables, horizontal rules.

Keep images as absolute URLs. This repository holds text.

## Canonical, and why it matters

Three of the Russian articles were first published on Habr and three of the English ones on Medium. When the same text lives in two places, search engines pick one and may pick the wrong one. `canonical` tells them which is the original.

Set `canonical` to the external URL while the external copy exists. Remove it for articles written for the site first — those are canonical here.
