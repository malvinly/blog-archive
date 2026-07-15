# Building and publishing

## Publishing

Every push to `main` triggers the GitHub Actions workflow ([.github/workflows/jekyll.yml](.github/workflows/jekyll.yml)), which builds the site and deploys it to GitHub Pages. No local toolchain required.

## Building locally

One-time setup:

1. Install Ruby with DevKit (3.3.x to match the Actions workflow):
   ```
   winget install RubyInstallerTeam.RubyWithDevKit.3.3
   ```
   Let the post-install `ridk install` step run. It sets up the toolchain needed to compile native gems.
2. From the repo root, install dependencies:
   ```
   bundle install
   ```

Preview the site:

```
bundle exec jekyll serve
```

Then open **http://localhost:4000/blog-archive/**. Note the `/blog-archive/` path; the root of `localhost:4000` will 404 because of the `baseurl` setting.

Useful flags:

- `--drafts`: include posts from `_drafts/` in the preview
- `--livereload`: auto-refresh the browser on rebuild
- `bundle exec jekyll build`: build into `_site/` without serving

`_site/` and `Gemfile.lock` are gitignored, so local builds won't dirty the repo.

## Layout

- `_posts/`: published posts, named `YYYY-MM-DD-title-slug.md`; the date and slug become the URL (`/:year/:month/:day/:title/`, matching the old WordPress permalinks)
- `_drafts/`: unpublished drafts (no date prefix); only rendered locally with `--drafts`
- `assets/images/<year>/<month>/`: post images, organized by the referencing post's date
