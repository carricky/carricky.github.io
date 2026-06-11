# siyuangao.com

Personal academic website of Siyuan Gao, built with [al-folio](https://github.com/alshedivat/al-folio) (Jekyll).

## Local development

```bash
bundle install
bundle exec jekyll serve
```

## Content

- Bio / homepage: `_pages/about.md`
- News: `_news/`
- Publications: `_bibliography/papers.bib`
- Blog posts: `_posts/`
- Site settings: `_config.yml`, `_data/socials.yml`

## Deployment

Pushing to `master` triggers `.github/workflows/deploy.yml`, which builds the site
and publishes it to the `gh-pages` branch (served by GitHub Pages at
[siyuangao.com](https://siyuangao.com)).
