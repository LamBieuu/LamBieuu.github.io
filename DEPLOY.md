# Deploy this site to GitHub Pages

## 1. Repository name

For the root personal site, name the repository exactly:

`YOUR_GITHUB_USERNAME.github.io`

Example: username `alice` -> repository `alice.github.io`.

## 2. Repository visibility

If your GitHub account uses GitHub Free, make the repository **Public** for GitHub Pages.

GitHub Pro / Team / Enterprise plans can use a private repository for Pages, but the resulting Pages website is still public unless you have an Enterprise configuration that supports private Pages access.

## 3. Upload/push

The files in this package must be at the repository root, not inside another nested folder.

Example:

```
YOUR_GITHUB_USERNAME.github.io/
├── .github/
├── _data/
├── _plugins/
├── _posts/
├── _tabs/
├── assets/
├── _config.yml
├── Gemfile
└── index.html
```

Then commit and push to `main`.

## 4. Enable Pages

On GitHub:

`Repository -> Settings -> Pages -> Build and deployment -> Source -> GitHub Actions`

Then open the **Actions** tab and wait for `Build and Deploy` to finish.

Your site will normally be:

`https://YOUR_GITHUB_USERNAME.github.io`

## 5. Set your final URL afterward

Once you know the final address, edit `_config.yml`:

```yaml
url: "https://YOUR_GITHUB_USERNAME.github.io"

github:
  username: YOUR_GITHUB_USERNAME
```

You may also uncomment the GitHub contact item in `_data/contact.yml`.

## Writing posts

Create a file such as:

`_posts/2026-09-01-introduction-to-zkp.md`

with:

```yaml
---
title: "Introduction to Zero-Knowledge Proofs"
date: 2026-09-01 15:00:00 +0700
categories: [Cryptography, Zero-Knowledge]
tags: [zkp, cryptography]
math: true
---
```

Then write normal Markdown below the front matter.

## Local preview

With Ruby and Bundler installed:

```bash
bundle install
bundle exec jekyll serve
```

Open `http://127.0.0.1:4000/`.
