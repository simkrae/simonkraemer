---
title: "CI/CD-Infrastruktur für Hugo auf GitHub Pages"
description: "Technische Implementierung eines automatisierten Deployments und Integration lokaler Instruktions-Layer für Gemini."
date: 2026-02-22T17:41:00+01:00
categories: ["Journal"]
tags: ["Hugo", "GitHub Actions", "CI/CD", "Gemini", "DevOps"]
showToc: true
draft: false
slug: "hugo-github-pipeline"
translationKey: "hugo-pipeline-2026"
---

**Die Automatisierung eines statischen Blogs erfordert eine entkoppelte Architektur. Durch die Nutzung von GitHub Actions wird der Build-Prozess vom lokalen Rechner in eine isolierte Umgebung verschoben. In Kombination mit einem strukturierten Instruktions-Set für Gemini lässt sich der gesamte Workflow von der Content-Erstellung bis zum Live-Gang standardisieren.**

## Projekt-Initialisierung und Submodule

Das Theme **PaperMod** wird als Git Submodule eingebunden. Dies stellt sicher, dass die Theme-Dateien nicht Teil des Haupt-Repositories sind, sondern als Referenz auf den Upstream-Stand fungieren.

```bash
hugo new site tech-journal
cd tech-journal
git init
git submodule add [https://github.com/adityatelange/hugo-PaperMod.git](https://github.com/adityatelange/hugo-PaperMod.git) themes/PaperMod
```

In der `hugo.yml` wird die Theme-Engine explizit definiert:

```yaml
baseURL: '[https://username.github.io/](https://username.github.io/)'
theme: 'PaperMod'
```

## GitHub Actions: Rekursiver Checkout

Das Deployment erfolgt über GitHub Actions. Die Herausforderung liegt im Git-Checkout: Standardmäßig werden Submodule nicht geladen. Dies führt zu leeren Builds. Die `with: submodules: recursive` Anweisung behebt diesen Engpass.

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

## Hybride Prompt-Architektur

Um Gemini als Werkzeug für die Erstellung von Hugo Page Bundles zu nutzen, muss der Kontext getrennt werden. Globale Instruktionen im Google-Konto definieren den Schreibstil (Nüchternheit, Verzicht auf Metaphern), während die lokale `GEMINI.md` die Dateisystem-Logik übernimmt.

```text
ROOT/
├── GEMINI.md       # Definition von Pfaden, Frontmatter & Skills
├── .aiexclude      # Filtert .env und /public für das LLM
├── hugo.yml        # Site-Konfiguration
└── content/        # Zielverzeichnis für Page Bundles
```

### Konfiguration der GEMINI.md
Diese Datei liegt im Stammverzeichnis und erweitert den Kontext von Gemini in VS Code. Sie spezifiziert:

- **Pfade:** Definition des Formats `content/posts/YYYY/MM/slug/index.md`.
- **Frontmatter:** Vorgabe der erforderlichen YAML-Keys (z.B. `date`, `tags`, `showToc`).
- **Commands:** Erstellung von PowerShell-Befehlen wie `New-Item -Path "..."`.
- **Trigger:** Definition von Keywords wie **"journal"**, um diesen spezifischen Workflow zu aktivieren.

## Isolation durch .aiexclude

Die Datei `.aiexclude` ist notwendig, um den Upload sensibler oder irrelevanter Daten in den LLM-Kontext zu verhindern. Im Gegensatz zur `.gitignore` steuert sie ausschließlich, welche Dateien für die KI sichtbar sind.

- **Secrets:** Dateien wie `.env` oder `credentials.json` werden isoliert.
- **Artefakte:** Der Ordner `public/` wird ausgeschlossen, um Kontext-Fenster nicht mit redundanten Build-Daten zu füllen.

