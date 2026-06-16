# How to Write a New Blog Post

## 1. Create the post file

Create a new Markdown file in `_posts/` following this naming convention:

```
_posts/YYYY-MM-DD-your-post-title.md
```

Example: `_posts/2024-06-15-my-new-post.md`

## 2. Add front matter

Every post must start with front matter (YAML between `---` delimiters):

```markdown
---
title: Your Post Title
date: 2024-06-15 10:00:00 -0800
categories: [TopCategory, SubCategory]
tags: [tag1, tag2, tag3]
---

Your post content starts here.
```

### Front matter fields

| Field        | Required | Notes |
|--------------|----------|-------|
| `title`      | Yes      | Displayed as the post heading |
| `date`       | Yes      | Format: `YYYY-MM-DD HH:MM:SS ±HHMM` (timezone offset) |
| `categories` | No       | 1–2 levels, e.g. `[Tech, Android]` |
| `tags`       | No       | Lowercase, any number of tags |
| `image`      | No       | Preview image shown on the post card (see below) |
| `toc`        | No       | Set `false` to hide the table of contents |
| `comments`   | No       | Set `false` to disable comments on a specific post |
| `pin`        | No       | Set `true` to pin the post to the top of the home page |

### Category rules
- Categories are hierarchical: `[TopLevel]` or `[TopLevel, SubLevel]`
- Each post should have at most 2 category levels
- Capitalization is preserved as-is

### Tag rules
- Tags should always be **lowercase**
- Tags appear in the Tags page and are used for filtering

## 3. Add images

Put images anywhere in the repo (e.g. `assets/img/posts/YYYY-MM-DD/`).

Reference them in Markdown:

```markdown
![Alt text](/assets/img/posts/2024-06-15/screenshot.png)
```

To set a preview image for the post card (shown on the home page):

```yaml
image:
  path: /assets/img/posts/2024-06-15/cover.png
  alt: Image description
```

## 4. Write the content

Standard Markdown plus these chirpy extras:

### Code blocks with syntax highlighting

````markdown
```python
def hello():
    print("Hello, World!")
```
````

### Callout boxes

```markdown
> This is a tip.
{: .prompt-tip }

> This is a warning.
{: .prompt-warning }

> This is a danger alert.
{: .prompt-danger }

> This is an info note.
{: .prompt-info }
```

### Table of contents

The TOC is generated automatically from your `##` and `###` headings. To disable it for a post, add `toc: false` to the front matter.

## 5. Preview locally (requires Ruby >= 3.0)

```bash
bundle install
bundle exec jekyll serve
```

Then open `http://localhost:4000` in your browser. Changes to posts hot-reload automatically.

## 6. Publish

Commit and push to `master`:

```bash
git add _posts/2024-06-15-my-new-post.md
git commit -m "add post: My New Post"
git push
```

GitHub Actions will automatically build and deploy the site. Check the **Actions** tab in your GitHub repo to monitor the build.

---

## One-time GitHub Pages setup

If you haven't already done this after migrating to chirpy, go to your repo's **Settings → Pages** and set the source to **GitHub Actions** (not "Deploy from a branch").
