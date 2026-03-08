# Jekyll Conversion Guidelines

This website has been converted to Jekyll. Here are some important files:

## Key Files and Directories

- `_config.yml` - Main Jekyll configuration
- `_layouts/default.html` - Base layout template
- `_includes/` - Reusable components (if needed)
- `assets/css/style.css` - Stylesheet
- `index.html` - English home page (now with Jekyll front matter)
- `it/index.html` - Italian home page (now with Jekyll front matter)

## Front Matter

Each page now uses YAML front matter to specify the layout and metadata:

```yaml
---
layout: default
lang: en
title: Luca Crema
---
```

## Local Testing

To test Jekyll locally:

```bash
gem install bundler jekyll
bundle exec jekyll serve
```

Then visit http://localhost:4000

## Publishing

GitHub Pages will automatically build and deploy your site when you push to the repository.
