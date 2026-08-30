# Shashank Yadav - Personal Website

[![Deploy Hugo site to GitHub Pages](https://github.com/yshashanky/yshashanky.github.io/actions/workflows/hugo.yml/badge.svg)](https://github.com/yshashanky/yshashanky.github.io/actions/workflows/hugo.yml)

Personal website and blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Deployed automatically to GitHub Pages.

🔗 **Live Site**: [https://yshashanky.github.io](https://yshashanky.github.io)

## About

This site serves as my professional portfolio and technical blog where I share insights about:
- Java backend engineering
- Distributed systems & microservices
- Apache Kafka & event-driven architecture
- Performance optimization & scalability
- Software engineering best practices

## Tech Stack

- **Static Site Generator**: [Hugo v0.146.0](https://gohugo.io/)
- **Theme**: [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- **Deployment**: GitHub Actions → GitHub Pages
- **CI/CD**: Automated deployment on push to `main` branch

## Getting Started

### Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) v0.146.0 or later
- Git

### Local Development

1. **Clone the repository**
   ```bash
   git clone https://github.com/yshashanky/yshashanky.github.io.git
   cd yshashanky.github.io
   ```

2. **Initialize theme submodule** (if not already done)
   ```bash
   git submodule update --init --recursive
   ```

3. **Start the development server**
   ```bash
   hugo server -D
   ```

4. **View the site**
   Open your browser to [http://localhost:1313](http://localhost:1313)

### Building for Production

```bash
hugo --minify
```

The static files will be generated in the `public/` directory.

## Content Management

### Creating a New Blog Post

```bash
hugo new content/blog/my-new-post.md
```

Edit the generated markdown file with your content. Front matter structure:

```yaml
---
title: "Your Post Title"
date: 2026-08-30
draft: false
tags: ["tag1", "tag2"]
description: "Brief description of your post"
---
```

### Project Structure

```
.
├── .github/workflows/    # GitHub Actions workflow for deployment
├── content/             # Content files (markdown)
│   ├── _index.md       # Homepage content
│   └── blog/           # Blog posts
├── layouts/            # Custom layout overrides
├── static/             # Static assets (images, files)
├── themes/PaperMod/    # Theme (as git submodule)
└── hugo.yaml          # Hugo configuration
```

## Deployment

The site automatically deploys to GitHub Pages on every push to the `main` branch via GitHub Actions.

**Deployment workflow:**
1. Push changes to `main` branch
2. GitHub Actions triggers the workflow
3. Hugo builds the site with `--minify`
4. Static files are deployed to GitHub Pages
5. Site is live at `https://yshashanky.github.io`

### Manual Deployment Trigger

You can manually trigger a deployment from the [Actions tab](https://github.com/yshashanky/yshashanky.github.io/actions) by running the workflow.

## License

This project is open source and available under the [MIT License](LICENSE).

## Contact

- **Email**: [shashankyadav.1299@gmail.com](mailto:shashankyadav.1299@gmail.com)
- **Website**: [yshashanky.github.io](https://yshashanky.github.io)
- **GitHub**: [@yshashanky](https://github.com/yshashanky)
- **LinkedIn**: [yshashanky](https://linkedin.com/in/yshashanky)
