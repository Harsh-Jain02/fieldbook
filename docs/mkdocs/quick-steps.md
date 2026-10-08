# MkDocs website: quick steps

*Last updated: 7 October 2026*

Just the steps, in order. For what a step does and why, follow its **Details** link to the [full guide](full-guide.md). Stuck? See [Troubleshooting](full-guide.md#troubleshooting).

Replace `my-docs` with your project name and `YOUR_USERNAME` with your GitHub username.

## Set up a new site

### 1. Create the folder and environment

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

[Details](full-guide.md#step-1-create-a-project-folder-and-a-virtual-environment)

### 2. Activate and install

=== "Windows"

    ```powershell
    .venv\Scripts\Activate.ps1
    python -m pip install mkdocs-material
    ```

    Command Prompt: `.venv\Scripts\activate.bat`. Git Bash: `source .venv/Scripts/activate`.

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

[Details](full-guide.md#step-2-activate-the-environment-and-install-material-for-mkdocs)

### 3. Save package versions

=== "Windows"

    ```powershell
    python -m pip freeze | Out-File -Encoding utf8 requirements.txt
    ```

=== "macOS"

    ```bash
    python -m pip freeze > requirements.txt
    ```

=== "Linux"

    ```bash
    python -m pip freeze > requirements.txt
    ```

[Details](full-guide.md#step-3-record-the-exact-package-versions)

### 4. Create the project

```bash
mkdocs new .
```

[Details](full-guide.md#step-4-create-the-mkdocs-project)

### 5. Add pages

Put your `.md` files in `docs/`.

[Details](full-guide.md#step-5-add-your-pages)

### 6. Configure `mkdocs.yml`

Replace its contents with the configuration below, then change `site_url` and `nav` to match your site.

??? example "mkdocs.yml"

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

[Details](full-guide.md#step-6-configure-the-site-in-mkdocsyml)

### 7. Preview and check

```bash
mkdocs serve
mkdocs build --strict
```

`mkdocs serve` prints the preview address. Press **Ctrl+C** to stop it. Fix everything `mkdocs build --strict` reports.

[Details](full-guide.md#step-7-preview-and-check-the-site)

### 8. Create `.gitignore`

```text
.venv/
site/
```

[Details](full-guide.md#step-8-tell-git-which-files-to-ignore)

### 9. Create an empty GitHub repository

GitHub → **+** → **New repository** → name it `my-docs` → **Public** → no README, `.gitignore` or licence → **Create repository**.

[Details](full-guide.md#step-9-create-an-empty-repository-on-github)

### 10. Push to GitHub

```bash
git init
git add .
git commit -m "Initial documentation"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/my-docs.git
git push -u origin main
```

[Details](full-guide.md#step-10-push-the-project-to-github)

### 11. Turn on GitHub Pages

Repository → **Settings** → **Pages** → **Build and deployment** → **Source**: **GitHub Actions**.

[Details](full-guide.md#step-11-set-github-pages-to-deploy-from-github-actions)

### 12. Add the deployment workflow

Create `.github/workflows/deploy.yml` with:

??? example ".github/workflows/deploy.yml"

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

[Details](full-guide.md#step-12-add-the-deployment-workflow)

### 13. Push the workflow

```bash
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Pages deployment"
git push
```

Wait for a green tick in the repository's **Actions** tab.

[Details](full-guide.md#step-13-push-the-workflow)

### 14. Open the site

```text
https://YOUR_USERNAME.github.io/my-docs/
```

[Details](full-guide.md#step-14-open-your-live-site)

## Update the site

Activate the environment:

=== "Windows"

    ```powershell
    .venv\Scripts\Activate.ps1
    ```

=== "macOS"

    ```bash
    source .venv/bin/activate
    ```

=== "Linux"

    ```bash
    source .venv/bin/activate
    ```

Then:

1. Edit or add `.md` files in `docs/`. For a new page, also add it to `nav` in `mkdocs.yml`.
2. Preview: `mkdocs serve`
3. Check: `mkdocs build --strict`, and fix everything it reports.
4. Publish:

    ```bash
    git add .
    git commit -m "Describe the change"
    git push
    ```

Installed or upgraded a package? Save package versions again ([step 3](#3-save-package-versions)) and commit `requirements.txt`.

[Details](full-guide.md#working-on-the-site-later)

## Set up on another computer

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

[Details](full-guide.md#set-up-the-project-on-another-computer)
