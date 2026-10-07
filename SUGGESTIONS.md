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
