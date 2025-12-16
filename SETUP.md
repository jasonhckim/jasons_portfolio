# Setup Guide

## GitHub Pages Deployment

### Initial Setup

1. **Push to GitHub**
   ```bash
   git init
   git add .
   git commit -m "Initial portfolio setup"
   git branch -M main
   git remote add origin [your-repo-url]
   git push -u origin main
   ```

2. **Enable GitHub Pages**
   - Go to repository Settings → Pages
   - Source: Deploy from a branch
   - Branch: `main` (or `master`)
   - Folder: `/ (root)`
   - Click Save

3. **Wait for Deployment**
   - GitHub Pages will build and deploy automatically
   - Your site will be available at: `https://jasonhckim.github.io/jasons_portfolio/`
   - First deployment may take 2-5 minutes
   - Check deployment status in: Repository → Actions tab

4. **Share Your Portfolio**
   - **Direct URL:** `https://jasonhckim.github.io/jasons_portfolio/`
   - **Resume Link:** `https://jasonhckim.github.io/jasons_portfolio/assets/resume.pdf`
   - Add to LinkedIn profile, email signature, or business cards

### Adding a New AI System Case Study

1. Copy the template:
   ```bash
   cp systems/template.md systems/[your-system-name].md
   ```

2. Fill in all sections following the template

3. Add architecture diagrams to `diagrams/`:
   - Name: `[your-system-name]-architecture.png`
   - Reference in case study: `![Architecture](./../diagrams/[your-system-name]-architecture.png)`

4. Add sample outputs to `assets/` if available

5. Update `README.md` and `index.md` to link to your new case study:
   ```markdown
   ### [System Name](./systems/[your-system-name].md)
   ```

6. Commit and push:
   ```bash
   git add .
   git commit -m "Add [system name] case study"
   git push
   ```

### Local Preview (Optional)

If you want to preview locally before pushing:

1. Install Jekyll (requires Ruby):
   ```bash
   gem install bundler jekyll
   ```

2. Serve locally:
   ```bash
   bundle exec jekyll serve
   ```

3. View at `http://localhost:4000`

## File Structure

```
jasons_portfolio/
├── README.md              # GitHub repo homepage
├── index.md               # Portfolio website homepage
├── _config.yml            # Jekyll configuration
├── SETUP.md               # This file
├── .gitignore
├── .github/
│   └── workflows/
│       └── pages.yml      # GitHub Actions for Pages
├── systems/               # AI system case studies
│   ├── README.md
│   └── template.md
├── diagrams/              # Architecture diagrams
│   └── README.md
└── assets/                # Screenshots and outputs
    └── README.md
```

## Customization

### Updating Site Title/Description

Edit `_config.yml`:
```yaml
title: Jason's AI Systems Portfolio
description: Production-grade AI systems that replace manual operations
```

### Changing Theme

The portfolio uses Jekyll's `minima` theme by default. To customize:
1. Edit `_config.yml` to change theme
2. Or create custom CSS in `assets/css/` and reference in `_config.yml`

## Troubleshooting

### Pages Not Updating
- Check GitHub Actions tab for build errors
- Verify `_config.yml` syntax is correct
- Ensure all markdown files use proper front matter (for Jekyll)

### Images Not Showing
- Use relative paths: `./../diagrams/image.png`
- Ensure image files are committed to repository
- Check file extensions match (case-sensitive)

### Markdown Not Rendering
- Ensure files have proper markdown syntax
- Check for special characters that need escaping
- Verify file encoding is UTF-8

