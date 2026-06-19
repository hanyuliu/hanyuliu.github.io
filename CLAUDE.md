# Claude Instructions for hanyuliu.github.io

This is a Jekyll-based GitHub Pages blog.

## After Making Any Change

1. **Check if the dev server is running:**
   ```bash
   pgrep -f "jekyll serve" > /dev/null 2>&1
   ```
   If it is NOT running, start it:
   ```bash
   scripts/start-server
   ```
   Wait a few seconds for Jekyll to build, then confirm the server is up before proceeding.

2. **Open the default browser** to preview the change:
   ```bash
   open http://localhost:4000
   ```
   If you know the specific page that changed (e.g., a post or a tab), open that URL directly instead.

## Dev Server Scripts

These scripts live in `scripts/` and are the canonical way to manage the local server:

- `scripts/start-server` — starts `bundle exec jekyll serve --livereload` in the background, writes the PID to `.server.pid` (git-ignored), and logs output to `/tmp/jekyll-serve.log`.
- `scripts/stop-server` — reads `.server.pid`, kills the process, removes the PID file, and cleans Jekyll cache files. Falls back to `pkill -f "jekyll serve"` if the PID file is missing.

Never start Jekyll manually with `bundle exec jekyll serve` directly; always use `scripts/start-server` so the PID is tracked.

## Project Structure

- `_posts/` — blog posts (Markdown)
- `_tabs/` — main navigation pages
- `_layouts/`, `_includes/` — theme templates
- `_data/` — site data files
- `assets/` — CSS, JS, images
- `_config.yml` — Jekyll configuration
