# CI1. GitHub Actions Workflow

This chapter documents the GitHub Actions workflow used to build and deploy the Salesforce AI Development Field Guide to GitHub Pages. It covers the workflow file, the build process, and the deployment configuration.

## Workflow file

The workflow is defined in `.github/workflows/docs.yml`. It uses `actions/deploy-pages@v4` for GitHub Pages deployment with OIDC authentication.

```yaml
name: docs

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          pip install mkdocs-material pymdown-extensions

      - name: Build site
        run: |
          cp -r diagrams docs/diagrams
          cp -r cookbook docs/cookbook
          cp -r templates docs/templates
          cp -r case-studies docs/case-studies
          mkdocs build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

## Build process

The build process copies assets into the `docs/` directory before running `mkdocs build`:

```
1. Copy diagrams/ to docs/diagrams/
2. Copy cookbook/ to docs/cookbook/
3. Copy templates/ to docs/templates/
4. Copy case-studies/ to docs/case-studies/
5. Run mkdocs build (reads mkdocs.yml for configuration)
6. Output goes to site/ directory
7. Upload site/ as GitHub Actions artifact
8. Deploy artifact to GitHub Pages
```

The `docs_dir: docs` in mkdocs.yml means mkdocs looks for content in the `docs/` directory. The workflow stage files into `docs/` before building.

## GitHub Pages configuration

The Pages source is set to "GitHub Actions" rather than the traditional gh-pages branch. This means the workflow controls deployment rather than a git push to the gh-pages branch.

To enable this:
1. Go to repository Settings -> Pages
2. Set Source to "GitHub Actions"
3. The workflow handles the rest on push to main

## Verification workflow

Run the build locally:

```bash
cd ~/Documents/Documentation/salesforce-ai-development-field-guide
cp -r diagrams docs/diagrams
cp -r cookbook docs/cookbook
cp -r templates docs/templates
cp -r case-studies docs/case-studies
mkdocs build
```

If the build succeeds, open `site/index.html` in a browser to verify.

## Troubleshooting

### Build fails with import errors

Ensure `requirements.txt` is installed:

```bash
pip install -r requirements.txt
```

### Pages shows 404 after deploy

Wait 2-3 minutes for the Pages build to complete. GitHub Pages takes a few minutes to update after the workflow completes.

Check the Actions tab for the deployment run. The "deploy" job must complete successfully before the site is live.

### Diagrams not appearing

Verify the diagrams are in `docs/diagrams/` after the copy step:

```bash
ls docs/diagrams/
```

If missing, check the path in mkdocs.yml. SVG files are referenced as `/diagrams/filename.svg` in the markdown.

## What this chapter covered

- The GitHub Actions workflow file and its structure
- The build process (copying assets, running mkdocs build, deploying)
- GitHub Pages configuration (Actions-based deployment)
- Local verification steps
- Troubleshooting common issues

## References

- [MkDocs documentation](https://www.mkdocs.org/)
- [MkDocs Material documentation](https://squidfunk.github.io/mkdocs-material/)
- [GitHub Actions deployment](https://docs.github.com/en/actions/deployment/deploying-github-pages)
- [actions/deploy-pages](https://github.com/actions/deploy-pages)