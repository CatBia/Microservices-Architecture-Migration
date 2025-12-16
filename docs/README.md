# GitHub Pages Documentation

This directory contains the GitHub Pages site for the Microservices Architecture Migration project.

## Viewing the Site

The GitHub Pages site is automatically deployed when changes are pushed to the `main` branch. You can view it at:

`https://[your-username].github.io/Microservices-Architecture-Migration/`

## Local Development

To preview the site locally:

1. **Using Python (Simple HTTP Server)**
   ```bash
   cd docs
   python3 -m http.server 8000
   ```
   Then open `http://localhost:8000` in your browser.

2. **Using Node.js (http-server)**
   ```bash
   npx http-server docs -p 8000
   ```

3. **Using VS Code Live Server**
   - Install the "Live Server" extension
   - Right-click on `index.html` and select "Open with Live Server"

## Structure

- `index.html` - Main page with project overview, achievements, tech stack, and services
- `_config.yml` - Jekyll configuration (for static HTML sites, minimal config needed)

## Customization

To customize the GitHub Pages site:

1. Edit `index.html` to modify content, styling, or structure
2. The site uses vanilla HTML, CSS, and JavaScript (no build step required)
3. Colors and styling can be modified in the `<style>` section of `index.html`

## Deployment

The site is automatically deployed via GitHub Actions (`.github/workflows/pages.yml`) when:
- Changes are pushed to the `main` branch in the `docs/` directory
- The workflow is manually triggered

To enable GitHub Pages:
1. Go to repository Settings → Pages
2. Select "GitHub Actions" as the source
3. The workflow will automatically deploy on the next push

