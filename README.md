# Orion Platform documentation (MkDocs example)

An example, **entirely fictional** documentation set for the product "Orion
Platform 4.2 LTS", prepared as material for building a static site with
[MkDocs](https://www.mkdocs.org/) and the
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme.

The set contains 56 pages across four navigation levels and uses the Markdown
extensions typical for the Material theme: admonitions, content tabs, tables,
definition lists, task lists, footnotes, code line highlighting, and Mermaid
diagrams.

## Structure

```text
.
├── mkdocs.yml              # MkDocs and Material theme configuration
├── requirements.txt        # dependencies needed to build the docs
├── overrides/              # theme template overrides (announcement bar)
│   └── main.html
└── docs/                   # documentation sources in Markdown
    ├── index.md
    ├── faq.md
    ├── changelog.md
    ├── support.md
    ├── stylesheets/extra.css
    ├── introduction/
    ├── getting-started/
    │   ├── installation/
    │   │   └── kubernetes/
    │   └── configuration/
    ├── guides/
    │   ├── authentication/
    │   ├── events/
    │   └── migrations/
    ├── api/
    │   ├── rest/
    │   │   └── resources/
    │   ├── webhooks/
    │   └── sdk/
    └── operations/
        ├── monitoring/
        └── runbooks/
```

## Requirements

- Python 3.9 or newer (verified on 3.12)
- `pip`

## Installing dependencies

Windows (PowerShell):

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Linux / macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Instead of `requirements.txt`, the theme alone is enough, since it pulls in
MkDocs and the extensions:

```bash
pip install mkdocs-material
```

## Live preview

```bash
mkdocs serve
```

The documentation is served at <http://127.0.0.1:8000>. The server rebuilds the
site after every file save. Useful variants:

```bash
mkdocs serve -a 0.0.0.0:8001    # different address and port
mkdocs serve --dirtyreload      # faster rebuilds for a large page set
mkdocs serve --strict           # treat warnings as errors
```

## Building the static site

```bash
mkdocs build --strict
```

The output lands in the `site/` directory, ready to be served by any static file
server. Additional options:

```bash
mkdocs build --clean            # remove the previous contents of site/
mkdocs build -d ../public       # different output directory
mkdocs build --no-directory-urls  # .html addresses (for opening from disk)
```

Checking the configuration without generating pages:

```bash
mkdocs --version
mkdocs get-deps                 # packages required by mkdocs.yml
```

## Publishing to GitHub Pages

```bash
mkdocs gh-deploy --force
```

## Material theme: what is configured

Two things enable the theme: the `mkdocs-material` package in the dependencies
and the `theme.name: material` entry in `mkdocs.yml`. This project additionally
configures:

- an English interface (`theme.language: en`) and English search,
- three color palettes with a toggle (auto / light / dark),
- a vertical navigation tree in the left sidebar, collapsed to the first level
  (no `navigation.tabs`, `navigation.sections`, or `navigation.expand`), with
  section overview pages (`navigation.indexes`),
- code copying, links between content tabs, tooltips,
- the `pymdownx.*` extensions required by Material features (collapsible
  admonitions, tabs, Mermaid diagrams, task lists, keyboard keys),
- a custom stylesheet `docs/stylesheets/extra.css` and the overridden
  `overrides/main.html` template.

## See also

- MkDocs documentation: <https://www.mkdocs.org/>
- Material for MkDocs documentation: <https://squidfunk.github.io/mkdocs-material/>
