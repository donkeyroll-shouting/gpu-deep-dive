# GPU Deep Dive

An MkDocs-powered documentation site.

## Local development

```bash
python -m pip install -r requirements.txt
mkdocs serve
```

## GitHub Pages deployment

The workflow at `.github/workflows/deploy-pages.yml` builds and deploys the site on every push. In the repository's **Settings → Pages**, select **GitHub Actions** as the build and deployment source. Update `site_url` in `mkdocs.yml` to your final GitHub Pages URL.
