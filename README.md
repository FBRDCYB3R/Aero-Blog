# Aerospace Blog — Jekyll Site

Runs on GitHub Pages with zero setup. Write posts in Markdown, commit, done.

---

## Setup (one time)

1. Create a GitHub repo named `yourusername.github.io`
2. Upload everything in this folder to that repo
3. Go to **Settings → Pages → Source: Deploy from branch → main / root**
4. Edit `_config.yml` — update your name, email, GitHub username, LinkedIn handle, and URL

Your site is live. Every time you push a commit, GitHub rebuilds it automatically.

---

## Writing a blog post

1. Go to your repo on GitHub.com
2. Navigate to the `_posts/` folder
3. Click **Add file → Create new file**
4. Name it: `YYYY-MM-DD-your-post-title.md`
   - Example: `2025-04-01-how-fins-work.md`
5. Paste this at the top, fill it in, then write below it:

```
---
title: "Your Post Title"
date: 2025-04-01
tags: [Rocketry]
description: "One sentence shown under the title."
read_time: 5
---

Your post content here in Markdown.
```

6. Click **Commit changes** — the post appears on the blog automatically.

**The blog post counter on the home page updates automatically** — it's powered by Jekyll, not hardcoded.

---

## Adding a project

Same idea, but in the `_projects/` folder. No date needed in the filename.

1. Navigate to `_projects/`
2. **Add file → Create new file**
3. Name it: `your-project-slug.md` (e.g. `helios-rocket.md`)
4. Use this front matter:

```
---
title: "Project Helios"
date: 2025-03-01
category: Rocketry
description: "Short description for the listing page."
status: Complete
tools: [OpenRocket, Python, MATLAB]
github: "https://github.com/you/repo"
---

Project write-up here.
```

---

## Adding images

1. Upload your image to `assets/images/` in the repo
2. Reference it in any post or project:

```markdown
![Description of image](/assets/images/your-photo.jpg)
```

---

## Updating your resume

Edit `resume.html` directly — it's the one page that stays as HTML since it has a specific layout. Just find the placeholder text and swap it out.

---

## Markdown cheat sheet

```markdown
## Heading
**bold**  *italic*  `code`

[Link text](https://url.com)
![Image alt](path/to/image.jpg)

- bullet list
1. numbered list

> blockquote

    code block (indent 4 spaces or use triple backticks)
```
