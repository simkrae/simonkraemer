---
title: "CI/CD Infrastructure for Hugo on GitHub Pages"
description: "Technical implementation of an automated deployment and integration of local instruction layers for Gemini."
date: 2026-02-22T17:41:00+01:00
categories: ["Journal"]
tags: ["Hugo", "GitHub Actions", "CI/CD", "Gemini", "DevOps"]
showToc: true
draft: false
slug: "hugo-github-pipeline"
---

**Automating a static blog requires a decoupled architecture. By using GitHub Actions, the build process is moved from the local machine to an isolated environment. In combination with a structured set of instructions for Gemini, the entire workflow from content creation to going live can be standardized.**

## Project Initialization and Submodules

The **PaperMod** theme is included as a Git submodule. This ensures that the theme files are not part of the main repository, but function as a reference to the upstream version.

```bash
hugo new site tech-journal
cd tech-journal
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

In `hugo.toml`, the theme engine is explicitly defined:

```yaml
baseURL: 'https://username.github.io/'
theme: 'PaperMod'
```

## GitHub Actions: Recursive Checkout

Deployment is done via GitHub Actions. The challenge lies in the Git checkout: by default, submodules are not loaded. This leads to empty builds. The `with: submodules: recursive` statement resolves this bottleneck.

```yaml
# .github/workflows/hugo.yml
name: Deploy
on:
  push:
    branches: ["main"]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      - uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.126.0'
          extended: true
      - run: hugo --minify
```

## Hybrid Prompt Architecture

To use Gemini as a tool for creating Hugo Page Bundles, the context must be separated. Global instructions in the Google account define the writing style (sobriety, no metaphors), while the local `GEMINI.md` handles the file system logic.

```text
ROOT/
├── GEMINI.md       # Definition of paths, frontmatter & skills
├── .aiexclude      # Filters .env and /public for the LLM
├── hugo.toml       # Site configuration
└── content/        # Target directory for Page Bundles
```

### Configuration of GEMINI.md
This file is located in the root directory and extends Gemini's context in VS Code. It specifies:

- **Paths:** Definition of the format `content/posts/YYYY/MM/slug/index.md`.
- **Frontmatter:** Specification of the required YAML keys (e.g., `date`, `tags`, `showToc`).
- **Commands:** Creation of PowerShell commands like `New-Item -Path "..."`.
- **Triggers:** Definition of keywords like **"journal"** to activate this specific workflow.

## Isolation through .aiexclude

The `.aiexclude` file is necessary to prevent the upload of sensitive or irrelevant data into the LLM context. Unlike `.gitignore`, it exclusively controls which files are visible to the AI.

- **Secrets:** Files like `.env` or `credentials.json` are isolated.
- **Artifacts:** The `public/` folder is excluded to avoid filling the context window with redundant build data.
