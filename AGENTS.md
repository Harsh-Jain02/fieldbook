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
| Local preview at http://127.0.0.1:8000/fieldbook/ (keeps running until stopped) | `.venv/Scripts/python.exe -m mkdocs serve` |

The build writes to `site/`, which is git-ignored.

Every build prints a boxed "Warning from the Material for MkDocs team" about MkDocs 2.0. It is an announcement, not a build warning: it does not fail `--strict` and needs no action.

`--strict` fails the build on:

- broken links between pages
- `nav` entries that point to missing files
- pages in `docs/` that are missing from `nav`
- links to a heading (`page.md#anchor`) that does not exist

MkDocs reports the last two only as `INFO` by default. The `validation` section of `mkdocs.yml` raises them to warnings so the strict build catches them. Do not remove it.

## Repository layout

```text
mkdocs.yml                    Site config: theme, nav, Markdown extensions
docs/                         Page sources (Markdown), one folder per topic; docs/index.md is the home page
docs/mkdocs/                  Topic: building and publishing this kind of site
docs/coding-agents/           Topic: setups and tricks for AI coding agents
docs/supabase/                Topic: Supabase projects and their databases
docs/stylesheets/extra.css    Custom CSS: makes whole cards clickable
requirements.txt              Pinned Python dependencies (pip freeze output)
SUGGESTIONS.md                Improvement ideas not adopted yet (not part of the site)
.github/workflows/deploy.yml  Builds the site and deploys it to GitHub Pages on push to main
site/                         Build output (git-ignored, never edit)
.venv/                        Local virtual environment (git-ignored)
```

## Pages and navigation

- Pages are Markdown files under `docs/`, grouped by topic. Each topic is a folder under `docs/` (for example `docs/mkdocs/`) and a section of the same name in `nav`.
- `nav` in `mkdocs.yml` is an explicit list, so a new page does not appear in the site navigation until it is added there.
- Each topic folder has an `index.md` topic page, listed first and without a label in its `nav` section. The `navigation.indexes` theme feature attaches it to the section, so clicking the section name in the sidebar opens it.
- A topic can have a short `quick-steps.md` (only what to do) next to a long `full-guide.md` (what to do and why). The quick page repeats commands and config files from the full guide and links to its step headings, so when you change one page, update the other to match.
- You may edit `mkdocs.yml`, including `nav`, theme features and Markdown extensions.
- Link between pages with relative paths to the `.md` file, for example `[Setup](../tools/setup.md)`. MkDocs only checks links written this way.

### Cards on the home page and topic pages

The home page (`docs/index.md`) shows one card per topic, and each topic page shows one card per page in that topic, in the same order as `nav`. Material cannot fill these in automatically, so keep them up to date by hand:

- **New page:** add it to `nav` and add a card for it to its topic page.
- **New topic:** create the folder with an `index.md` topic page, add a `nav` section with the topic page listed first, and add a card for it to the home page.

A card uses Material's card grid. Its title is a bold link whose text is the `nav` label, followed by a rule and a one-line description:

```markdown
<div class="grid cards" markdown>

-   **[Codex Slack notifications](codex-slack-notifications.md)**

    ---

    Get a Slack message when Codex needs approval or finishes.

</div>
```

Home page cards also start with an icon, for example `:material-robot-outline:{ .lg .middle } **[Coding agents](coding-agents/index.md)**`. An icon name works if its SVG file exists in the installed theme, for example `:material-robot-outline:` is `.venv/Lib/site-packages/material/templates/.icons/material/robot-outline.svg`.

`docs/stylesheets/extra.css` stretches the card's link over the whole card, so clicking anywhere on the card opens the page. Keep exactly one link per card: with more, clicks would go to the wrong one.

### Markdown features

Enabled in `mkdocs.yml`:

- `admonition`: callout blocks such as `!!! note` and `!!! warning`
- `attr_list`: attributes in curly braces, such as `{ .lg .middle }` on card icons
- `md_in_html`: Markdown inside HTML blocks that have the `markdown` attribute, used by card grids (`<div class="grid cards" markdown>`)
- `tables`
- `toc` with `permalink: true`: a link anchor on every heading
- `pymdownx.details`: collapsible blocks (`??? note "Title"` starts collapsed, `???+` starts open)
- `pymdownx.emoji` with Material's icon set: icons such as `:material-robot-outline:`. VS Code's YAML extension reports "Unresolved tag" on its two `!!python/name:` lines in `mkdocs.yml`. MkDocs reads them correctly, so leave them as they are.
- `pymdownx.highlight`: syntax highlighting in code blocks
- `pymdownx.superfences`: fenced code blocks, including inside list items, admonitions and content tabs. It replaces Python-Markdown's `fenced_code`, so do not add `fenced_code` back.
- `pymdownx.tabbed` with `alternate_style: true`: content tabs (`=== "Label"`, content indented by four spaces)

Theme features: `navigation.indexes` attaches each topic's `index.md` to its `nav` section. `content.code.copy` adds a copy button to every code block. `content.tabs.link` switches every tab with the same label across the site when the reader picks one, so tabs only stay in sync if their labels match exactly. Operating-system tabs use the labels `Windows`, `macOS` and `Linux`.

Other `pymdown-extensions` syntax is not enabled and will not render, for example task lists (`- [ ]`, `pymdownx.tasklist`) and keyboard keys (`++ctrl+c++`, `pymdownx.keys`). To use one, add it under `markdown_extensions` in `mkdocs.yml`. The package is already installed, so this does not need an install.

## Dependencies

`requirements.txt` is the pinned output of `pip freeze`, and CI installs from it. Do not edit it. If a change needs a new or upgraded package, ask the owner (see rule 2).

## Deployment

Every push to `main` runs `.github/workflows/deploy.yml`, which builds the site and publishes it to GitHub Pages. Anything committed under `docs/` ends up on the published site, so never put secrets, tokens, passwords or private information in pages.

CI runs `mkdocs build` without `--strict`, so a page with broken links can still deploy. The local strict build is the only check.

## Instruction files

This file is the single source of agent instructions. Codex reads `AGENTS.md` directly. Claude Code reads `CLAUDE.md`, which imports this file. Claude Code also has `.claude/settings.json`, which blocks git write and install commands. Add new instructions here, not in `CLAUDE.md`.
