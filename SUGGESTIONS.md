# Suggestions

Ideas for improving the site that have not been adopted yet. This file is outside `docs/`, so it is not part of the published site.

## Automatic "last updated" dates

**Status:** suggested on 7 October 2026, not adopted.

**Today:** pages show a hand-typed `*Last updated: ...*` line under the title. It has to be changed by hand on every edit, and it goes stale when someone forgets.

**Idea:** use [mkdocs-git-revision-date-localized-plugin](https://github.com/timvink/mkdocs-git-revision-date-localized-plugin). It takes each page's date from the last Git commit that changed the page, and Material for MkDocs shows it at the bottom of every page automatically. See [Document dates](https://squidfunk.github.io/mkdocs-material/setup/adding-a-git-repository/#document-dates) in the Material docs.

**Trade-offs:**

- One more package to install and keep pinned in `requirements.txt`.
- The date changes when a page is committed, not when the file is saved.
- It applies to every page, so the hand-typed lines should all be removed when it is adopted.
- The deploy workflow has to download the full Git history instead of only the latest commit. This is slightly slower, which is negligible for a repository this size.

**How to adopt it:**

1. Install the plugin:

   ```bash
   .venv/Scripts/python.exe -m pip install mkdocs-git-revision-date-localized-plugin
   ```

2. Save the new package list. In Git Bash or Command Prompt:

   ```bash
   .venv/Scripts/python.exe -m pip freeze > requirements.txt
   ```

   In Windows PowerShell 5.1, use `| Out-File -Encoding utf8 requirements.txt` instead of `> requirements.txt`, so the file is not saved as UTF-16.

3. Add the plugin to `mkdocs.yml`. `search` must be listed too, because a `plugins` list replaces the MkDocs default, which only contains `search`:

   ```yaml
   plugins:
     - search
     - git-revision-date-localized:
         type: date
   ```

4. In `.github/workflows/deploy.yml`, make the checkout step download the full history. Without this, every page shows the date of the latest commit:

   ```yaml
   - name: Checkout
     uses: actions/checkout@v6
     with:
       fetch-depth: 0
   ```

5. Remove the hand-typed `*Last updated: ...*` lines from the pages in `docs/`.

6. Run `.venv/Scripts/python.exe -m mkdocs build --strict`. A page that has not been committed yet has no Git history, so check how the plugin reports it. If that fails the strict build, see the plugin's `fallback_to_build_date` and `strict` options.

## Material-style visual refresh

**Status:** suggested on 8 October 2026. Phase 2 adopted on 9 October 2026; phases 1 and 3 not adopted.

**Today:** the site already uses Material for MkDocs and has useful basics such as navigation sections, search suggestions, copy buttons and linked content tabs. Its visual identity is still mostly the theme default, however, and the home page is the starter MkDocs page.

**Goal:** give Fieldbook the same kind of polished, easy-to-navigate feel as the [Material for MkDocs website](https://squidfunk.github.io/mkdocs-material/) without copying its branding or artwork. The reference site combines ordinary [theme configuration](https://github.com/squidfunk/mkdocs-material/blob/master/mkdocs.yml) with a [custom home-page template](https://github.com/squidfunk/mkdocs-material/blob/master/material/overrides/home.html), so configuration alone will not reproduce its landing page.

**Recommended direction:** use a restrained "personal field notebook" identity: indigo or deep-purple accents, a book-style icon, light and dark modes, strong search, and a home page that leads directly to topic areas. Keep documentation pages conventional and readable; reserve the more visual layout for the home page.

### Phase 1: polish the standard theme

This phase has the best benefit-to-effort ratio and needs no new package. Extend the existing `theme` configuration rather than replacing its current features:

```yaml
site_description: A personal fieldbook of useful things learned and solved
repo_name: harsh-jain02/fieldbook
repo_url: https://github.com/harsh-jain02/fieldbook

theme:
  name: material
  icon:
    logo: material/book-open-page-variant
  palette:
    - media: "(prefers-color-scheme: light)"
      scheme: default
      primary: indigo
      accent: deep purple
      toggle:
        icon: material/weather-night
        name: Switch to dark mode
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      primary: indigo
      accent: deep purple
      toggle:
        icon: material/weather-sunny
        name: Switch to light mode
  features:
    - navigation.tabs
    - navigation.indexes
    - navigation.sections
    - navigation.tracking
    - navigation.footer
    - navigation.top
    - search.suggest
    - search.highlight
    - search.share
    - toc.follow
    - content.code.copy
    - content.tabs.link
```

The [palette toggle](https://squidfunk.github.io/mkdocs-material/setup/changing-the-colors/#color-palette-toggle) gives readers explicit light and dark modes. [Navigation tabs, URL tracking and table-of-contents following](https://squidfunk.github.io/mkdocs-material/setup/setting-up-navigation/) make a larger knowledge base easier to explore. [Search highlighting and sharing](https://squidfunk.github.io/mkdocs-material/setup/setting-up-site-search/) improve the site's most important retrieval tool.

Also add a favicon when a final mark has been chosen:

```yaml
theme:
  favicon: assets/images/favicon.png
```

The repository link should be omitted if the source repository is not meant to be advertised in the header.

### Phase 2: replace the starter home page

Rewrite `docs/index.md` as a useful entry point rather than a list of MkDocs commands. A practical first version can remain ordinary Markdown and contain:

- a short Fieldbook title and one-sentence purpose;
- a prominent link to browse the first topic and a prompt to use search;
- a small "Topics" section with one card or short description per top-level navigation section;
- a "Recently useful" or "Start here" section with a few hand-picked pages.

This is likely enough while the site has only one topic. It avoids maintaining custom HTML before there is enough content to justify it.

**Adopted on 9 October 2026** in this form:

- The home page has a title, a one-sentence purpose and one card per topic, with an icon on each card. It hides the navigation and table-of-contents sidebars.
- Each topic has an `index.md` topic page with one card per page. The `navigation.indexes` feature opens it when the section name is clicked in the sidebar.
- `docs/stylesheets/extra.css` makes the whole card clickable.
- New Markdown extensions: `attr_list`, `md_in_html` and `pymdownx.emoji`.

The "Start here" section and the prompt to use search were not added. WRITING-GUIDE.md explains how to keep the cards up to date.

### Phase 3: add an optional custom landing page

When there are several topics, create a distinctive hero layout similar in spirit to the reference site:

```text
overrides/
└── home.html
docs/
├── assets/
│   └── images/
│       └── fieldbook-hero.svg
└── stylesheets/
    └── extra.css        (already exists; add the hero styles here)
```

`docs/stylesheets/extra.css` and its `extra_css` entry in `mkdocs.yml` were added with phase 2, so only the template folder needs registering:

```yaml
theme:
  name: material
  custom_dir: overrides
```

Then select the template in `docs/index.md`, which already has the `hide` list:

```yaml
---
template: home.html
hide:
  - navigation
  - toc
---
```

The template should extend Material's `main.html`, use its `tabs` or `content` block for a responsive hero, and retain Material's own buttons, grid width, typography and colour variables. The [theme customization guide](https://squidfunk.github.io/mkdocs-material/customization/#overriding-blocks) recommends extending blocks so upstream theme updates remain easier to adopt.

A Fieldbook hero could contain:

- the heading "Useful things, kept findable";
- one sentence explaining that this is a personal reference for learned solutions;
- primary and secondary buttons for "Browse topics" and "MkDocs guide";
- an original notebook, map or index-card illustration;
- a compact grid of topic cards below the hero.

Do not copy the Material site's illustration or its complete home-page CSS. Original artwork and a small Fieldbook-specific stylesheet will avoid branding/licensing ambiguity and be much easier to maintain.

### Decisions needed before adoption

- Confirm the visual tone: notebook/library, minimal technical docs, or a more colourful landing page.
- Choose the primary/accent colours and whether to create a custom logo or use a bundled Material icon.
- Decide whether the GitHub repository link should appear in the header.
- Choose whether to stop after the Markdown home page or invest in the custom hero and illustration.

### Checks when adopting

1. Preview at desktop and mobile widths in both light and dark modes.
2. Check keyboard focus, colour contrast, descriptive image alternative text and reduced-motion behavior.
3. Confirm that navigation tabs remain useful as the number of topics grows; with only one topic, they may feel sparse.
4. Run `.venv/Scripts/python.exe -m mkdocs build --strict` and fix every warning.
