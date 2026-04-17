# Quick Setup Guide for MkDocs on GitHub

## What you got:

✅ `mkdocs.yml` — Full config with Material theme, ILA green/gold branding  
✅ `docs/` — All 10 markdown files in correct structure  
✅ `docs/css/ila.css` — Custom ILA branding (green: #2d5016, gold: #d4af37)  
✅ `.github/workflows/deploy.yml` — Auto-deploy to GitHub Pages on every push  

## How to deploy:

### 1. Copy files to your repo root (locally)

```bash
# Clone your repo
git clone https://github.com/thefactorystack/ILA
cd ILA

# Copy the files I created
cp /path/to/mkdocs.yml .
cp -r /path/to/docs .
cp -r /path/to/.github .
```

### 2. Push to GitHub

```bash
git add mkdocs.yml docs/ .github/workflows/deploy.yml
git commit -m "Add MkDocs documentation with Material theme and GitHub Pages deployment"
git push origin main
```

### 3. Enable GitHub Pages

Go to your repo → Settings → Pages
- Source: Deploy from a branch
- Branch: `gh-pages` (auto-created by the workflow)
- Folder: `/ (root)`
- Save

### 4. View your site

After ~1 minute, your site will be live at:  
**https://thefactorystack.github.io/ILA/**

---

## What happens on every push:

1. GitHub Actions triggers (`.github/workflows/deploy.yml`)
2. Installs MkDocs + Material theme + extensions
3. Builds `site/` directory from `docs/`
4. Deploys to GitHub Pages (`gh-pages` branch)
5. Your site updates automatically

No manual steps needed after this — just push and the site rebuilds.

---

## Test locally before pushing:

```bash
# Install MkDocs
pip install mkdocs mkdocs-material pymdown-extensions

# Run development server
mkdocs serve

# Visit http://localhost:8000
```

---

## Next Steps (Optional):

- Customize `mkdocs.yml` (add nav items, change colors in `docs/css/ila.css`)
- Add more documentation pages in `docs/pages/`
- Create a `docs/includes/abbreviations.md` file if you use abbreviations
- Add images: store in `docs/images/`, reference as `![alt](../../images/filename.png)`

---

Questions? Check:
- Material theme docs: https://squidfunk.github.io/mkdocs-material/
- MkDocs docs: https://www.mkdocs.org/
