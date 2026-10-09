# Fieldbook

Fieldbook is a personal knowledge base. Its owner records things they have learned and tricks they found useful, across many topics, so that when they need to do something again they know where to look. Pages should be findable later, both through the navigation and through the site search.

The site is built with [MkDocs](https://www.mkdocs.org/) and the [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme, and published at <https://harsh-jain02.github.io/fieldbook/>.

Start with this file. Then:

- To add or change pages, read [WRITING-GUIDE.md](WRITING-GUIDE.md).
- Coding agents also follow [AGENTS.md](AGENTS.md).
- Ideas for the site that have not been adopted yet are in [SUGGESTIONS.md](SUGGESTIONS.md).

None of these files are part of the published site: MkDocs only builds the `docs/` folder.

## Set up

The project is set up on Windows, with Python 3.12. The commands in this file work in PowerShell and Git Bash, run from the repository root. They call the virtual environment's Python directly as `.venv/Scripts/python.exe`, so they work whether or not the environment is activated.

To set up the project on a new computer, clone the repository, then create the virtual environment and install the pinned packages:

```bash
python -m venv .venv
.venv/Scripts/python.exe -m pip install -r requirements.txt
```

For more detail, and for fixes to common problems, see [Set up the project on another computer](docs/mkdocs/full-guide.md#set-up-the-project-on-another-computer) and [Troubleshooting](docs/mkdocs/full-guide.md#troubleshooting) in the MkDocs full guide.

## Build and preview

| Task | Command |
|---|---|
| Build and check | `.venv/Scripts/python.exe -m mkdocs build --strict` |
| Local preview at http://127.0.0.1:8000/fieldbook/ (keeps running until stopped) | `.venv/Scripts/python.exe -m mkdocs serve` |

The build writes to `site/`, which is git-ignored.

Every build prints a boxed "Warning from the Material for MkDocs team" about MkDocs 2.0. It is an announcement, not a build warning: it does not fail `--strict` and needs no action.

`--strict` fails the build on:

- broken links between pages
- `nav` entries that point to missing files
- pages in `docs/` that are missing from `nav`
- links to a heading (`page.md#anchor`) that does not exist

MkDocs reports the last two only as `INFO` by default. The `validation` section of `mkdocs.yml` raises them to warnings so the strict build catches them. Do not remove it.

The deployment workflow runs `mkdocs build` without `--strict`, so a page with broken links still deploys. The local strict build is the only check, so run it before every push.

## Repository layout

```text
README.md                     This file: start here
WRITING-GUIDE.md              How to add and change pages, and the page conventions
AGENTS.md                     Rules for coding agents (Claude Code, OpenAI Codex)
CLAUDE.md                     Makes Claude Code read AGENTS.md
SUGGESTIONS.md                Improvement ideas not adopted yet
mkdocs.yml                    Site config: theme, nav, Markdown extensions, link checks
docs/                         Page sources (Markdown), one folder per topic; docs/index.md is the home page
docs/mkdocs/                  Topic: building and publishing this kind of site
docs/coding-agents/           Topic: setups and tricks for AI coding agents
docs/supabase/                Topic: Supabase projects and their databases
docs/stylesheets/extra.css    Custom CSS: makes whole cards clickable
requirements.txt              Pinned Python dependencies (pip freeze output)
.github/workflows/deploy.yml  Builds the site and deploys it to GitHub Pages on push to main
.claude/settings.json         Claude Code permissions: blocks git write and install commands (git-ignored)
site/                         Build output (git-ignored, never edit)
.venv/                        Local virtual environment (git-ignored)
```

## Dependencies

`requirements.txt` is the pinned output of `pip freeze`. The deployment workflow installs from it, so the published site is built with the same package versions as the local preview.

To add or upgrade a package, install it, save the versions again and commit `requirements.txt`. In Git Bash:

```bash
.venv/Scripts/python.exe -m pip install --upgrade <package>
.venv/Scripts/python.exe -m pip freeze > requirements.txt
```

In Windows PowerShell 5.1, save the versions with `.venv/Scripts/python.exe -m pip freeze | Out-File -Encoding utf8 requirements.txt` instead, so the file is not saved as UTF-16.

Coding agents never install packages or edit `requirements.txt`. They ask the owner instead (see [AGENTS.md](AGENTS.md)).

## Deployment

Every push to `main` runs `.github/workflows/deploy.yml`, which installs the packages in `requirements.txt` with the latest Python 3, builds the site and publishes it to GitHub Pages. There is no review step, so whatever is pushed to `main` goes live.

Anything committed under `docs/` ends up on the published site, so never put secrets, tokens, passwords or private information in pages.
