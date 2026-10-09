# AGENTS.md

Instructions for coding agents (Claude Code, OpenAI Codex) working in this repository.

Fieldbook is the owner's personal knowledge base, built with MkDocs and the Material theme and published on GitHub Pages.

## Read first

- [README.md](README.md): the project, setup, commands, repository layout, dependencies and deployment. Read it before your first change in a session.
- [WRITING-GUIDE.md](WRITING-GUIDE.md): how to add and change pages, and the page conventions. Read it before you change anything under `docs/` or in `mkdocs.yml`.

## Rules

1. **No git write operations.** Do not run any git command that changes the repository, index, branches, tags, remotes, config or history: `add`, `commit`, `push`, `pull`, `fetch`, `merge`, `rebase`, `reset`, `checkout`, `switch`, `restore`, `stash`, `branch`, `tag`, `rm`, `mv`, `clean`, and so on. This includes `mkdocs gh-deploy` and `ghp-import`, which commit and push to a `gh-pages` branch. Read-only commands such as `git status`, `git diff`, `git log` and `git show` are fine. When something needs to be committed or pushed, tell the owner.
2. **No installing anything.** Do not install, upgrade or uninstall packages (pip, uv, npm, winget, etc.), do not create or recreate the virtual environment, and do not edit `requirements.txt`. If something needs to be installed, stop and ask the owner to do it, giving the exact command. If `.venv/` is missing or broken, ask the owner to set it up.
3. **Run the strict build before finishing.** After any change under `docs/` or to `mkdocs.yml`, run `.venv/Scripts/python.exe -m mkdocs build --strict` and fix every warning it reports. Call the virtual environment's Python directly like this, because your shell may not have the environment activated. The boxed "Warning from the Material for MkDocs team" that every build prints is an announcement, not a build warning, so leave it.
4. **No secrets in pages.** Never put secrets, tokens, passwords or private information under `docs/`. Every push to `main` publishes it.
5. **Ask before changing the whole site.** You may edit `mkdocs.yml`, including `nav`, theme features and Markdown extensions. Ask the owner before introducing structure that affects the whole site and is not covered by WRITING-GUIDE.md.

## Checklist for page changes

- **New page:** add it to `nav` and add a card for it to its topic page.
- **New topic:** create the folder with an `index.md` topic page, add a `nav` section with the topic page listed first, add a card for it to the home page, and add the folder to the repository layout in README.md.

The strict build catches a page missing from `nav`, but not a missing card. WRITING-GUIDE.md has the details.

## Keep the project docs up to date

When the owner makes a decision about how the project or its pages work, for example by answering your questions or approving a proposal, record it as part of the same task, without being asked:

- page structure, naming, writing style, cards, navigation or Markdown features: WRITING-GUIDE.md
- setup, commands, repository layout, dependencies or deployment: README.md
- how agents must work: AGENTS.md
- an idea in SUGGESTIONS.md adopted or rejected: update its status there

Also update these files when your change makes them out of date, for example after adding a topic folder or a Markdown extension. Do not record choices that only apply to one page. In your final message, tell the owner what you recorded and where.

## Instruction files

This file is the single source of agent instructions. Codex reads `AGENTS.md` directly. Claude Code reads `CLAUDE.md`, which imports this file. Add new agent instructions here, not in `CLAUDE.md`.

Claude Code also has `.claude/settings.json`, which blocks git write and install commands. The `.claude` folder is git-ignored, so the file only exists on the computer where it was created.
