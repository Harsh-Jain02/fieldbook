# Writing guide

How to add and change pages in Fieldbook. Read [README.md](README.md) first for setup and the build command.

The conventions in this guide were decided by the owner, so follow them on every page. When the owner decides something new about how pages are written or organised, it is added here (see "Keep the project docs up to date" in [AGENTS.md](AGENTS.md)). For anything this guide does not cover, follow the style of existing pages, and ask the owner before introducing structure that affects the whole site.

## Topics and pages

- Pages are Markdown files under `docs/`, grouped by topic. Each topic is a folder under `docs/` (for example `docs/mkdocs/`) and a section of the same name in `nav` in `mkdocs.yml`.
- `nav` is an explicit list, so a new page does not appear in the site navigation until it is added there.
- Each topic folder has an `index.md` topic page, listed first and without a label in its `nav` section. The `navigation.indexes` theme feature attaches it to the section, so clicking the section name in the sidebar opens it.
- The home page (`docs/index.md`) shows one card per topic, and each topic page shows one card per page in that topic. See [Cards](#cards).

### Add a page

1. Create the Markdown file in its topic folder, following the [page conventions](#page-conventions).
2. Add it to the topic's section in `nav`.
3. Add a card for it to the topic's `index.md`, in the same position as in `nav`.
4. Run the [checks before finishing](#checks-before-finishing).

### Add a topic

1. Create the folder under `docs/` with an `index.md` topic page: a `#` heading with the topic name, followed by a card grid.
2. Add a section to `nav` with the topic page listed first, without a label.
3. Add a card for the topic, with an icon, to the home page, in the same position as in `nav`.
4. Add the folder to the [repository layout](README.md#repository-layout) in README.md.
5. Run the [checks before finishing](#checks-before-finishing).

## Page conventions

- **File names:** lowercase words joined by hyphens, for example `move-project-to-another-region.md`. Topic folders follow the same rule, for example `coding-agents/`.
- **Title:** the page's `#` heading says what the page helps you do, for example "Move a Supabase project to another region". The `nav` label can be shorter, for example "Move a project to another region".
- **Last updated:** the line under the title is `*Last updated: 9 October 2026*`, with the day, the month name and the year. Change the date whenever you change what the page says. Topic pages and the home page do not have this line.
- **Section order:**
    1. A short introduction under the date: what the page does and, where it matters, which system or shell the commands are for.
    2. `## What you need`: the tools and accounts needed before you start.
    3. The steps, as `## Step 1: <what to do>`, `## Step 2: ...` and so on. A long guide can group its steps into parts (`## Part 1: <name>`), with the steps as `### Step N: ...` headings numbered on across the parts, as in the MkDocs full guide.
    4. `## References` last: links to the sources the page is based on, such as official documentation.

    Leave out the sections a page does not need. Other sections can be added where they help, for example "How it works" before the steps or "Troubleshooting" before the references.

### Quick steps and full guide

A topic can have a short `quick-steps.md` (only what to do) next to a long `full-guide.md` (what to do and why). The quick page repeats commands and config files from the full guide and links to its step headings, so when you change one page, update the other to match.

### Secrets and private information

Everything under `docs/` is published when it is pushed to `main`. Never put secrets, tokens, passwords or private information in pages, including in example commands, config files, connection strings and screenshots. Replace real values with placeholders and say what goes in their place.

## Links

Link between pages with relative paths to the `.md` file, for example `[Setup](../tools/setup.md)`. MkDocs only checks links written this way.

To link to a heading, add its anchor: the heading text in lowercase, without punctuation and with hyphens for spaces. For example, `## Step 2: Get the old project's connection string` is `#step-2-get-the-old-projects-connection-string`.

## Cards

The home page and the topic pages use Material's card grid. Material cannot fill in the cards automatically, so keep them up to date by hand, in the same order as `nav`.

A card's title is a bold link whose text is the `nav` label, followed by a rule and a one-line description:

```markdown
<div class="grid cards" markdown>

-   **[Codex Slack notifications](codex-slack-notifications.md)**

    ---

    Get a Slack message when Codex needs approval or finishes.

</div>
```

Home page cards also start with an icon, for example `:material-robot-outline:{ .lg .middle } **[Coding agents](coding-agents/index.md)**`. An icon name works if its SVG file exists in the installed theme, for example `:material-robot-outline:` is `.venv/Lib/site-packages/material/templates/.icons/material/robot-outline.svg`.

`docs/stylesheets/extra.css` stretches the card's link over the whole card, so clicking anywhere on the card opens the page. Keep exactly one link per card: with more, clicks would go to the wrong one.

## Markdown features

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

Other `pymdown-extensions` syntax is not enabled and will not render, for example task lists (`- [ ]`, `pymdownx.tasklist`) and keyboard keys (`++ctrl+c++`, `pymdownx.keys`). To use one, add it under `markdown_extensions` in `mkdocs.yml` and to the list above. The package is already installed, so this does not need an install.

## Checks before finishing

1. Run `.venv/Scripts/python.exe -m mkdocs build --strict` and fix every warning it reports.
2. Check that every new page has a card on its topic page, and every new topic has a card on the home page. The strict build does not check cards.
3. Check the `*Last updated*` date on every page whose content changed.
4. Check that no secrets or private information went into the pages.
