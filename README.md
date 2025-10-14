# MKDocs Setup

1. Create a directory to put docs into (repository ideally)
```bash
mkdir -p $TOP_LEVEL_DOCS_FOLDER
```
2. cd into directory
```bash
cd $TOP_LEVEL_DOCS_FOLDER
```
3. Creaet a virtual python environment (for building)
```bash
python3 -m venv venv
```
4. Activate the environment
```bash
source ./venv/bin/activate
```
5. Install mkdocs with material theme
```bash
pip3 install mkdocs-material
```
6. Test Installation
```bash
mkdocs serve
```
cntl + c to stop server
7. Customizations
```bash
nano mkdocs.yml
```
```yml
site_name: Seawolf Marketplace
site_description: "Official documentation for the Seawolf Marketplace project"
site_author: "Seawolf Development Team"

theme:
  name: material
  palette:
    scheme: slate          # <- enables dark mode by default
    primary: custom	   # <- adjust brand color here
    accent: custom         # <- accent color
  logo: assets/logo.png    # optional logo (put under docs/assets)
  favicon: assets/favicon.png

extra_css:
  - stylesheets/uaa.css

nav:
  - Home: index.md
  - Getting Started: getting-started.md
  - Features: features.md
  - API Reference: api.md
  - FAQ: faq.md

plugins:
  - search
  # optional quality-of-life plugins
  # - mkdocstrings
  # - tags

markdown_extensions:
  - admonition
  - codehilite
  - footnotes
  - toc:
      permalink: true
  - pymdownx.details
  - pymdownx.superfences
  - pymdownx.highlight
  - pymdownx.inlinehilite
  - pymdownx.emoji:
      emoji_index: !!python/name:material.extensions.emoji.twemoji
      emoji_generator: !!python/name:material.extensions.emoji.to_svg

```
```bash
nano docs/index.md
```
```yaml
# Welcome to Seawolf Marketplace

Welcome to the **Seawolf Marketplace** documentation.

Here you’ll find everything you need to understand, deploy, and extend Seawolf Marketplace — our unified platform for digital goods and resource exchange.

---

## 🔹 Quick Links

- [Getting Started](getting-started.md)
- [API Reference](api.md)
- [FAQ](faq.md)

```
```bash
nano docs/getting-started.md
```
```yml
# Getting Started

To get started with **Seawolf Marketplace**, you’ll need:

1. A valid Seawolf account
2. Access credentials (provided by your administrator)
3. Docker or Python 3.12+ installed on your machine

Run the following command to launch your environment:

```bash
git clone https://github.com/yourorg/seawolf-marketplace.git
cd seawolf-marketplace
make setup
```
8. See Results
```bash
mkdocs serve
```