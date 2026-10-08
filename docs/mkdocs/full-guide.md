# How to make an MkDocs website and publish it on GitHub Pages

*Last updated: 7 October 2026*

!!! tip "In a hurry?"
    The [quick steps](quick-steps.md) list only what to do, without the explanations.

This guide builds a documentation website from plain Markdown files using [MkDocs](https://www.mkdocs.org/) and the [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme, then publishes it for free on GitHub Pages. Once it is set up, publishing a change is a single `git push`: a GitHub Actions workflow rebuilds and redeploys the site for you.

Where commands differ between operating systems, they are shown in tabs. Pick your system once and every other tab on the site switches with it.

The examples use `my-docs` as the project folder and repository name. Replace `YOUR_USERNAME` with your GitHub username wherever it appears.

## What you end up with

```text
my-docs/
├── .github/
│   └── workflows/
│       └── deploy.yml     # builds and publishes the site on every push
├── docs/
│   └── index.md           # home page; every other page goes in here too
├── .gitignore             # keeps .venv/ and site/ out of Git
├── mkdocs.yml             # site configuration
└── requirements.txt       # exact package versions, used by GitHub Actions
```

Two more folders appear on your computer but are never committed: `.venv/` (the Python virtual environment) and `site/` (the built website).

## Before you start

You need Python 3.10 or later, Git and a GitHub account. Check what you already have:

=== "Windows"

    ```powershell
    python --version
    git --version
    ```

    - **Python:** download it from [python.org/downloads](https://www.python.org/downloads/). If the installer offers **Add python.exe to PATH**, tick it.
    - **Git:** download it from [git-scm.com](https://git-scm.com/). It also installs Git Bash and Git Credential Manager, which handles signing in to GitHub.

=== "macOS"

    ```bash
    python3 --version
    git --version
    ```

    - **Python:** install it from [python.org/downloads](https://www.python.org/downloads/) or with Homebrew (`brew install python`).
    - **Git:** if `git --version` says Git is missing, macOS offers to install the Command Line Developer Tools, which include it.

=== "Linux"

    ```bash
    python3 --version
    git --version
    ```

    - **Python:** usually already installed. On Debian and Ubuntu, also install virtual environment support: `sudo apt install python3-venv`.
    - **Git:** `sudo apt install git` on Debian and Ubuntu, or `sudo dnf install git` on Fedora.

If you have never committed with Git on this computer, tell it who you are. The email address is stored in every commit and is visible to anyone who can see the repository. GitHub's settings offer a private `noreply` address you can use instead.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

GitHub Pages is free for **public** repositories. Publishing from a private repository needs a paid GitHub plan.

## Part 1: Build the site on your computer

### Step 1: Create a project folder and a virtual environment

A virtual environment is a private copy of Python for this project. It keeps the site's packages separate from everything else on your computer, so that `pip freeze` in step 3 lists only what the site needs.

=== "Windows"

    ```powershell
    mkdir my-docs
    cd my-docs
    python -m venv .venv
    ```

=== "macOS"

    ```bash
    mkdir my-docs
    cd my-docs
    python3 -m venv .venv
    ```

=== "Linux"

    ```bash
    mkdir my-docs
    cd my-docs
    python3 -m venv .venv
    ```

This creates a `.venv` folder. You never edit it and never commit it.

### Step 2: Activate the environment and install Material for MkDocs

=== "Windows"

    ```powershell
    .venv\Scripts\Activate.ps1
    python -m pip install mkdocs-material
    ```

    These commands are for PowerShell. In Command Prompt, activate with `.venv\Scripts\activate.bat`; in Git Bash, with `source .venv/Scripts/activate`. If PowerShell says running scripts is disabled, see [Troubleshooting](#powershell-says-running-scripts-is-disabled).

=== "macOS"

    ```bash
    source .venv/bin/activate
    python -m pip install mkdocs-material
    ```

=== "Linux"

    ```bash
    source .venv/bin/activate
    python -m pip install mkdocs-material
    ```

Once the environment is active, your prompt starts with `(.venv)`, and `python` means the environment's Python on every system. Installing `mkdocs-material` also installs MkDocs itself and the extensions this guide uses.

`python -m pip` is used instead of plain `pip` because it always runs the pip that belongs to the active Python. Plain `pip` sometimes is not found, or installs into a different Python.

!!! note
    Activation lasts only for the current terminal window. Every time you open a new terminal to work on the site, activate the environment again.

### Step 3: Record the exact package versions

Save the list of installed packages and their versions to `requirements.txt`. GitHub Actions installs from this file, so the published site is built with exactly the same versions as your preview.

=== "Windows"

    ```powershell
    python -m pip freeze | Out-File -Encoding utf8 requirements.txt
    ```

    In Windows PowerShell 5.1 (the version built into Windows), `python -m pip freeze > requirements.txt` saves the file as UTF-16, which Git treats as a binary file, so you cannot see its changes in diffs. The command above saves it as UTF-8. In Command Prompt or Git Bash, `python -m pip freeze > requirements.txt` is fine.

=== "macOS"

    ```bash
    python -m pip freeze > requirements.txt
    ```

=== "Linux"

    ```bash
    python -m pip freeze > requirements.txt
    ```

Run this again whenever you install or upgrade a package.

### Step 4: Create the MkDocs project

```bash
mkdocs new .
```

The `.` means "in the current folder". It creates:

```text
mkdocs.yml      # site configuration
docs/
    index.md    # home page
```

### Step 5: Add your pages

Every Markdown (`.md`) file in the `docs/` folder becomes a page. You can organise pages in subfolders, such as `docs/guides/setup.md`.

For the example configuration in the next step, create two pages, `docs/installation.md` and `docs/architecture.md`, each starting with a heading such as `# Installation`.

To link from one page to another, use the relative path to the `.md` file, for example `[Installation](installation.md)`. MkDocs only checks links written this way.

### Step 6: Configure the site in `mkdocs.yml`

Replace the contents of `mkdocs.yml` with:

```yaml
site_name: My Documentation
site_url: https://YOUR_USERNAME.github.io/my-docs/

theme:
  name: material
  features:
    - navigation.sections
    - navigation.top
    - search.suggest
    - content.code.copy
    - content.tabs.link

nav:
  - Home: index.md
  - Getting Started:
      - Installation: installation.md
  - Architecture: architecture.md

markdown_extensions:
  - admonition
  - tables
  - toc:
      permalink: true
  - pymdownx.details
  - pymdownx.highlight
  - pymdownx.superfences
  - pymdownx.tabbed:
      alternate_style: true

validation:
  nav:
    omitted_files: warn
  links:
    anchors: warn
```

What each setting does:

| Setting | What it does |
|---|---|
| `site_name` | The name shown in the site header and the browser tab. |
| `site_url` | The address the site is published at. For GitHub Pages it is `https://YOUR_USERNAME.github.io/REPOSITORY_NAME/`. `mkdocs serve` also uses its path for the preview address. |
| `theme: name: material` | Uses the Material for MkDocs theme. |
| `navigation.sections` | Shows top-level groups in `nav` as sections in the sidebar. |
| `navigation.top` | Shows a "Back to top" button when you scroll up. |
| `search.suggest` | Suggests how to finish the word you are typing in the search box. |
| `content.code.copy` | Adds a copy button to every code block. |
| `content.tabs.link` | Links tabs that have the same label, so choosing "macOS" in one place switches every "macOS" tab on the site. |
| `nav` | The order and titles of pages in the navigation. Paths are relative to `docs/`. A page that is not listed is still built and searchable, but does not appear in the navigation. |
| `admonition` | Callout boxes such as `!!! note` and `!!! warning`. |
| `tables` | Markdown tables, like this one. |
| `toc` with `permalink: true` | A table of contents for each page, and a link anchor (¶) next to every heading. |
| `pymdownx.details` | Collapsible blocks: `??? note "Title"` starts collapsed, `???+ note "Title"` starts open. |
| `pymdownx.highlight` | Syntax highlighting in code blocks. |
| `pymdownx.superfences` | Code blocks inside lists, callout boxes and tabs. |
| `pymdownx.tabbed` | Content tabs. `alternate_style: true` is the style Material for MkDocs requires. |
| `validation` | Turns two problems that MkDocs normally reports only as `INFO` into warnings: a page in `docs/` that is missing from `nav` (`omitted_files`), and a link to a heading that does not exist (`anchors`). |

Every file listed in `nav` must exist in `docs/`, and with the `validation` settings above, every page in `docs/` must be listed in `nav`. Breaking either rule causes a warning in the preview and fails the strict build in step 7.

!!! tip "Writing tabs in a page"
    Start each tab with `=== "Label"` and indent its content by four spaces:

    ````markdown
    === "Windows"

        ```powershell
        python -m venv .venv
        ```

    === "macOS"

        ```bash
        python3 -m venv .venv
        ```
    ````

    Use exactly the same labels everywhere, so that `content.tabs.link` can switch them together.

### Step 7: Preview and check the site

Start the preview server:

```bash
mkdocs serve
```

MkDocs prints the address it is serving on, for example `Serving on http://127.0.0.1:8000/my-docs/`. The path at the end comes from `site_url`. Opening `http://127.0.0.1:8000` redirects there. The page reloads by itself every time you save a file. Press **Ctrl+C** in the terminal to stop the server.

Before publishing, run a strict build:

```bash
mkdocs build --strict
```

It builds the site into the `site/` folder and stops with an error if a link to another page is broken or a `nav` entry points to a missing file. Thanks to the `validation` settings from step 6, it also stops if a page is missing from `nav` or a link points to a heading that does not exist. Without those settings, MkDocs reports the last two only as `INFO` lines, which are easy to miss.

## Part 2: Publish the site on GitHub Pages

### Step 8: Tell Git which files to ignore

Create a file named `.gitignore` in the project folder with:

```text
.venv/
site/
```

`.venv/` is large and only works on your computer. `site/` is the build output, and GitHub Actions builds its own copy. Neither belongs in the repository.

### Step 9: Create an empty repository on GitHub

1. On GitHub, click **+** in the top-right corner, then **New repository**.
2. Name it `my-docs`.
3. Make it **Public**, unless you have a paid plan (see [Before you start](#before-you-start)).
4. Do **not** add a README, `.gitignore` or licence. The repository must be empty, or the first push in the next step is rejected.
5. Click **Create repository**.

### Step 10: Push the project to GitHub

```bash
git init
git add .
git commit -m "Initial documentation"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/my-docs.git
git push -u origin main
```

| Command | What it does |
|---|---|
| `git init` | Turns the folder into a Git repository. |
| `git add .` | Stages every file, except those listed in `.gitignore`. |
| `git commit -m "..."` | Saves a snapshot of the staged files. |
| `git branch -M main` | Names the branch `main`. The workflow in step 12 deploys from `main`. |
| `git remote add origin ...` | Connects the folder to the repository you created on GitHub. |
| `git push -u origin main` | Uploads the commits. `-u` lets you use plain `git push` from now on. |

Before committing, you can run `git status` to check that `.venv/` and `site/` are not in the list.

The first push asks you to sign in to GitHub. On Windows, Git Credential Manager opens a browser for this. If Git asks for a password instead, your GitHub account password will not work: sign in with the [GitHub CLI](https://cli.github.com/) (`gh auth login`) or use a personal access token.

### Step 11: Set GitHub Pages to deploy from GitHub Actions

1. Open the repository on GitHub and go to **Settings** → **Pages**.
2. Under **Build and deployment** → **Source**, select **GitHub Actions**.

GitHub Pages can publish either straight from a branch or from a GitHub Actions workflow. GitHub recommends Actions for automated deployments. Your workflow builds the site and hands it to Pages, so no build output ever needs to be committed.

Do this before step 12. If the workflow runs while Pages is not set to GitHub Actions, the run fails.

### Step 12: Add the deployment workflow

Create the folders `.github/workflows/` in the project folder (note the dot at the start of `.github`), and inside them a file named `deploy.yml`:

```yaml
name: Deploy MkDocs

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.x"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Build MkDocs
        run: mkdocs build

      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v4
        with:
          path: site

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v4
```

How it works:

- **`on: push: branches: main`**: runs the workflow every time you push to `main`.
- **`permissions`**: lets the workflow read the repository and publish to GitHub Pages.
- **`concurrency`**: if you push again while a deployment is still running, the older run is cancelled.
- **`build` job**: downloads the repository, installs Python and the packages in `requirements.txt`, builds the site into `site/` and uploads that folder to GitHub Pages.
- **`deploy` job**: runs after `build` and publishes the uploaded site.

!!! warning "Broken links still get published"
    The workflow runs `mkdocs build` without `--strict`, so a page with a broken link still deploys. Run `mkdocs build --strict` on your computer before pushing, or change the line to `run: mkdocs build --strict` so that the workflow fails instead.

!!! note
    The action versions (`@v6`, `@v5`, `@v4`) were current when this guide was written. Newer major versions may be available by the time you read this.

### Step 13: Push the workflow

```bash
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Pages deployment"
git push
```

Open the **Actions** tab of the repository to watch the run. It usually takes a minute or two. A green tick means the site is published. A red cross means the run failed: click the run to see which step failed and why.

### Step 14: Open your live site

The site is published at:

```text
https://YOUR_USERNAME.github.io/my-docs/
```

The same link appears under **Settings** → **Pages** and in the summary of the workflow run. After the very first deployment, it can take a few minutes before the address starts working.

## Working on the site later

### Add or change a page

1. Open a terminal in the project folder and activate the environment ([step 2](#step-2-activate-the-environment-and-install-material-for-mkdocs)).
2. Create or edit a `.md` file in `docs/`.
3. If it is a new page, add it to `nav` in `mkdocs.yml`.
4. Preview with `mkdocs serve`.
5. Check with `mkdocs build --strict` and fix everything it reports.
6. Commit and push. The workflow republishes the site.

### Install or upgrade a package

Install or upgrade it with `python -m pip install ...`, then save the versions again ([step 3](#step-3-record-the-exact-package-versions)) and commit `requirements.txt`. If you skip that, GitHub Actions builds the site without the new package, and the build fails or the published site looks different from your preview.

### Set up the project on another computer

=== "Windows"

    ```powershell
    git clone https://github.com/YOUR_USERNAME/my-docs.git
    cd my-docs
    python -m venv .venv
    .venv\Scripts\Activate.ps1
    python -m pip install -r requirements.txt
    ```

=== "macOS"

    ```bash
    git clone https://github.com/YOUR_USERNAME/my-docs.git
    cd my-docs
    python3 -m venv .venv
    source .venv/bin/activate
    python -m pip install -r requirements.txt
    ```

=== "Linux"

    ```bash
    git clone https://github.com/YOUR_USERNAME/my-docs.git
    cd my-docs
    python3 -m venv .venv
    source .venv/bin/activate
    python -m pip install -r requirements.txt
    ```

Installing from `requirements.txt` gives you exactly the versions the site was last built with.

## Troubleshooting

### `pip` is not recognised, or packages install into the wrong Python

Use `python -m pip` instead of `pip`, and check that the environment is active (your prompt starts with `(.venv)`).

### PowerShell says running scripts is disabled

Windows blocks PowerShell scripts by default, including the activation script. Allow scripts for your own user account, then activate again:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Alternatively, use Command Prompt or Git Bash and their activation commands from [step 2](#step-2-activate-the-environment-and-install-material-for-mkdocs).

### `python` is not found on macOS or Linux

Use `python3` to create the environment. Once the environment is active, `python` works too.

### Creating the environment fails with "ensurepip is not available"

This happens on Debian and Ubuntu when virtual environment support is missing. Install it with `sudo apt install python3-venv`, delete the half-created `.venv` folder and repeat [step 1](#step-1-create-a-project-folder-and-a-virtual-environment).

### `mkdocs` is not recognised

The environment is not active. Activate it, or run MkDocs through Python, for example `python -m mkdocs serve`.

### The strict build fails

The message names the page and the link or `nav` entry that is wrong:

- **A link or `nav` entry points to a missing file:** fix the path or create the page. Links between pages must be relative paths to `.md` files.
- **A page is not included in `nav`:** add it to `nav` in `mkdocs.yml`.
- **An anchor is not found:** the heading was renamed, or the `#...` part of the link is misspelled. Copy the correct link from the ¶ symbol next to the heading.

### Tabs or code blocks show up as plain text

Check that the `markdown_extensions` from [step 6](#step-6-configure-the-site-in-mkdocsyml) are all in `mkdocs.yml`, and that everything inside a tab is indented by four spaces.

### The workflow fails at "Setup Pages" or "Deploy"

GitHub Pages is not set to deploy from GitHub Actions. Do [step 11](#step-11-set-github-pages-to-deploy-from-github-actions), then go to the failed run in the **Actions** tab and click **Re-run jobs**.

### The live site shows a 404 page

Wait a few minutes after the first deployment. Check that the address includes the repository name (`/my-docs/`) and that the latest run in the **Actions** tab has a green tick.

### Every build prints a boxed "Warning from the Material for MkDocs team"

Material for MkDocs 9.7 prints this announcement about MkDocs 2.0 on every build. It is not a build warning, it does not fail `--strict`, and it needs no action.

## References

- [MkDocs documentation](https://www.mkdocs.org/)
- [Material for MkDocs documentation](https://squidfunk.github.io/mkdocs-material/)
- [Material for MkDocs: Publishing your site](https://squidfunk.github.io/mkdocs-material/publishing-your-site/)
- [Material for MkDocs: Content tabs](https://squidfunk.github.io/mkdocs-material/reference/content-tabs/)
- [Material for MkDocs: Code blocks](https://squidfunk.github.io/mkdocs-material/reference/code-blocks/)
- [GitHub Pages documentation](https://docs.github.com/en/pages)
