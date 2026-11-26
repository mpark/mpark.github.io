# Michael Park's site

Source for [mpark.github.io](https://mpark.github.io).

This site loads [So Simple](https://github.com/mmistakes/so-simple-theme) as a
GitHub Pages remote theme, pinned to upstream commit `0128355ab62c20740a17775ee1df5382db472cb3`.
The theme files are downloaded during the build.

## Local preview

```sh
bundle install
bundle exec jekyll serve
```

The Gemfile includes GitHub Pages' dependencies so local builds use its Jekyll
version. The `jekyll-remote-theme` plugin downloads the pinned theme; local
template overrides take precedence. Build with
`JEKYLL_ENV=production bundle exec jekyll build` to enable production settings.

## Posts and drafts

Posts live in `_posts/programming/` and `_posts/trip-report/`, with dated filenames.
Existing dated post URLs and `/feed.xml` are preserved.

- `published: false` excludes a post from a normal build. Preview it with
  `bundle exec jekyll serve --unpublished`.
- `draft: true` renders a shareable public URL while omitting the post from the
  homepage, category lists, and normal feed. It still appears in search, tags,
  the sitemap, and previous/next links.
- Files in `_drafts/` are excluded unless you use `bundle exec jekyll serve --drafts`.
  The feed plugin includes drafts when previewing with `--drafts`.
- `share: true` enables sharing links. Sharing is disabled by default.

## Theme customization

`_config.yml` uses the current theme's settings. The homepage, category archives,
tag archive, and search page use upstream layouts. Search and the feed use
upstream implementations; there is no separate JSON search or custom Atom feed.

The template changes relative to upstream are favicon links in
`_includes/head-custom.html` and public-draft filtering in
`_includes/posts-limit.html` and `_includes/posts-category.html`.
The local `_layouts/default.html` adds the pinned theme revision to the CSS and
JavaScript URLs so cached files from older theme versions do not break the layout.
To update the theme, change the revision in `remote_theme` in `_config.yml`, then
build and check the site. Theme updates no longer require rebasing the site.
See the [upstream documentation](https://github.com/mmistakes/so-simple-theme#readme)
for available theme settings.
