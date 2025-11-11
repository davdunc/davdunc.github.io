# David Duncan's Technical Blog

A Jekyll-powered blog covering trading framework development, Linux engineering, and software development.

## Topics

- **Trading Framework Development**: Building robust trading systems and algorithmic strategies
- **Linux Development**: Kernel work, package maintenance, and system programming
- **Software Engineering**: Best practices and real-world project experiences

## Local Development

To run this blog locally:

```bash
# Install dependencies
bundle install

# Run Jekyll server
bundle exec jekyll serve

# View at http://localhost:4000
```

## Writing Posts

Posts are written in Markdown and stored in `_posts/` with the naming convention:

```
YYYY-MM-DD-title-of-post.md
```

Example frontmatter:

```yaml
---
layout: post
title: "Your Post Title"
date: 2025-11-11 10:00:00 -0500
categories: [trading, linux]
tags: [python, kernel, performance]
---
```

## Site Structure

- `_layouts/` - Page templates
- `_posts/` - Blog posts
- `_drafts/` - Unpublished drafts
- `assets/` - CSS, JS, and images
- `index.md` - Homepage
- `about.md` - About page
- `archive.html` - Post archive

## License

Content is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Code snippets are licensed under MIT.

## Contact

- GitHub: [@davdunc](https://github.com/davdunc)
- Blog: [blog.davidduncan.org](https://blog.davidduncan.org)
