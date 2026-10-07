# AGENTS.md

Instructions for coding agents (Claude Code, OpenAI Codex) working in this repository.

## About this project

Fieldbook is a personal knowledge base built with MkDocs and the Material theme. The owner records things they have learned and tricks they found useful, across many topics, so that when they need to do something again they know where to look. Pages should be findable later, both through the navigation and through the site search.

Writing conventions (page structure, file naming, tone) are not decided yet. Until they are written down here, follow the style of existing pages and ask the owner before introducing structure that affects the whole site.

## Rules

1. **No git write operations.** Do not run any git command that changes the repository, index, branches, tags, remotes, config or history: `add`, `commit`, `push`, `pull`, `fetch`, `merge`, `rebase`, `reset`, `checkout`, `switch`, `restore`, `stash`, `branch`, `tag`, `rm`, `mv`, `clean`, and so on. This includes `mkdocs gh-deploy` and `ghp-import`, which commit and push to a `gh-pages` branch. Read-only commands such as `git status`, `git diff`, `git log` and `git show` are fine. When something needs to be committed or pushed, tell the owner.
2. **No installing anything.** Do not install, upgrade or uninstall packages (pip, uv, npm, winget, etc.), and do not create or recreate the virtual environment. If something needs to be installed, stop and ask the owner to do it, giving the exact command.
3. **Run the strict build before finishing.** After any change under `docs/` or to `mkdocs.yml`, run the build command below and fix every warning it reports.

## Environment

- Windows only. The commands below work in both PowerShell and Git Bash, run from the repository root.
- Python 3.12 with a virtual environment in `.venv/`. Your shell may not have it activated, so call its Python directly as `.venv/Scripts/python.exe`.
- If `.venv/` is missing or broken, ask the owner to set it up (see rule 2).

## Commands

| Task | Command |
|---|---|
| Build and check (required before finishing) | `.venv/Scripts/python.exe -m mkdocs build --strict` |
| Local preview at http://127.0.0.1:8000 (keeps running until stopped) | `.venv/Scripts/python.exe -m mkdocs serve` |

The build writes to `site/`, which is git-ignored.

Every build prints a boxed "Warning from the Material for MkDocs team" about MkDocs 2.0. It is an announcement, not a build warning: it does not fail `--strict` and needs no action.

`--strict` fails the build on broken links between pages and on `nav` entries that point to missing files. It does **not** fail on a page that exists in `docs/` but is missing from `nav`; MkDocs reports that only as an `INFO` line ("The following pages exist in the docs directory, but are not included in the "nav" configuration"). Check the build output for that line.

## Repository layout

```text
mkdocs.yml                    Site config: theme, nav, Markdown extensions
docs/                         Page sources (Markdown); docs/index.md is the home page
requirements.txt              Pinned Python dependencies (pip freeze output)
SUGGESTIONS.md                Improvement ideas not adopted yet (not part of the site)
.github/workflows/deploy.yml  Builds the site and deploys it to GitHub Pages on push to main
site/                         Build output (git-ignored, never edit)
.venv/                        Local virtual environment (git-ignored)
```

## Pages and navigation

- Pages are Markdown files under `docs/`.
- `nav` in `mkdocs.yml` is an explicit list, so a new page does not appear in the site navigation until it is added there.
- You may edit `mkdocs.yml`, including `nav`, theme features and Markdown extensions.
- Link between pages with relative paths to the `.md` file, for example `[Setup](../tools/setup.md)`. MkDocs only checks links written this way.

### Markdown features

Enabled in `mkdocs.yml`:

- `admonition`: callout blocks such as `!!! note` and `!!! warning`
- `tables`
- `toc` with `permalink: true`: a link anchor on every heading
- `pymdownx.highlight`: syntax highlighting in code blocks
- `pymdownx.superfences`: fenced code blocks, including inside list items, admonitions and content tabs. It replaces Python-Markdown's `fenced_code`, so do not add `fenced_code` back.
- `pymdownx.tabbed` with `alternate_style: true`: content tabs (`=== "Label"`, content indented by four spaces)

Theme features: `content.code.copy` adds a copy button to every code block. `content.tabs.link` switches every tab with the same label across the site when the reader picks one, so tabs only stay in sync if their labels match exactly. Operating-system tabs use the labels `Windows`, `macOS` and `Linux`.

Other `pymdown-extensions` syntax is not enabled and will not render, for example collapsible blocks (`???`, `pymdownx.details`), task lists (`- [ ]`, `pymdownx.tasklist`) and keyboard keys (`++ctrl+c++`, `pymdownx.keys`). To use one, add it under `markdown_extensions` in `mkdocs.yml`. The package is already installed, so this does not need an install.

## Dependencies

`requirements.txt` is the pinned output of `pip freeze`, and CI installs from it. Do not edit it. If a change needs a new or upgraded package, ask the owner (see rule 2).

## Deployment

Every push to `main` runs `.github/workflows/deploy.yml`, which builds the site and publishes it to GitHub Pages. Anything committed under `docs/` ends up on the published site, so never put secrets, tokens, passwords or private information in pages.

CI runs `mkdocs build` without `--strict`, so a page with broken links can still deploy. The local strict build is the only check.

## Instruction files

This file is the single source of agent instructions. Codex reads `AGENTS.md` directly. Claude Code reads `CLAUDE.md`, which imports this file. Claude Code also has `.claude/settings.json`, which blocks git write and install commands. Add new instructions here, not in `CLAUDE.md`.
