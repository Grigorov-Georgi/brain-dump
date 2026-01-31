# Brain Dump

Knowledge base and project ideas repository, deployed to GitHub Pages using MkDocs.

## Local Development

1. Create and activate a virtual environment:
```bash
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

3. Create symlinks (if not already created):
```bash
cd docs
ln -sf ../knowledge knowledge
ln -sf ../project_ideas project_ideas
cd ..
```

4. Serve locally (auto-reloads on file changes):
```bash
mkdocs serve
```

The site will be available at `http://127.0.0.1:8000`

5. Build static site:
```bash
mkdocs build
```

## Deployment

The site is automatically deployed to GitHub Pages via GitHub Actions when changes are pushed to the `main` branch.

To enable GitHub Pages:
1. Go to repository Settings → Pages
2. Source should be set to "GitHub Actions"

## Structure

- `knowledge/` - Knowledge base articles
- `project_ideas/` - Project ideas and concepts
- `docs/` - Documentation index and assets
