# GitHub Pages Setup

This directory is configured for GitHub Pages with Jekyll.

## Theme

Using **Cayman** theme - a clean, light documentation-style theme perfect for technical content.

## Local Development

To run the site locally:

```bash
cd docs
bundle install
bundle exec jekyll serve
```

Then visit `http://localhost:4000/notes-senior/`

## GitHub Pages Configuration

### Repository Settings

1. Go to your repository **Settings** → **Pages**
2. Under **Source**, select:
   - **Source**: Deploy from a branch
   - **Branch**: `main` (or `master`)
   - **Folder**: `/docs`
3. Click **Save**

### GitHub Actions (Recommended)

Alternatively, use GitHub Actions for more control:

1. Go to repository **Settings** → **Pages**
2. Under **Source**, select: **GitHub Actions**
3. The workflow in `.github/workflows/jekyll-gh-pages.yml` will automatically deploy on push

## URL Structure

- Homepage: `https://yourusername.github.io/notes-senior/`
- Topics: `https://yourusername.github.io/notes-senior/java/01-java-fundamentals.html`

## Customization

Edit `_config.yml` to customize:
- `title`: Site title
- `description`: Site description
- `baseurl`: Repository name (keep as `/notes-senior`)
- `url`: Your GitHub Pages URL

## Content Organization

```
docs/
├── _config.yml          # Jekyll configuration
├── index.html           # Homepage
├── Gemfile             # Ruby dependencies
├── java/               # Java notes
├── microservices/      # Microservices notes
├── system-design/      # System design notes
├── rest-api/           # REST API notes
├── lld/                # Low-level design notes
├── dbms/               # Database notes
└── nosql/              # NoSQL notes
```

All markdown files are automatically converted to HTML by Jekyll.
