# mrWheel.github.io — landing page

A single-page GitHub Pages site that lists all public repositories owned by [@mrWheel](https://github.com/mrWheel).

## Architecture

```
GitHub Action (update-repos.yml)
        │
        ▼ writes
   repos.json   ◄── committed to the repository
        │
        ▼ fetched on page load
   index.html   ── renders repository cards in the browser
```

- **No build step, bundler, or framework.** The page is plain HTML + CSS + JavaScript.
- **No GitHub API calls from the browser.** All API calls are made inside the GitHub Action.
- `repos.json` is the single data source for the landing page.
- GitHub Pages deployment runs through `.github/workflows/deploy-pages.yml` with **Node.js 24**.

## How repos.json is generated

The workflow `.github/workflows/update-repos.yml` runs automatically and:

1. Fetches all public repositories for `mrWheel` via the GitHub API (paginated).
2. For each repository, resolves GitHub Pages availability (`/repos/{owner}/{repo}/pages`).
3. For each repository, detects GitBook documentation by scanning:
   - `repo.homepage` for a `gitbook.io` or `app.gitbook.com` URL,
   - `repo.description` for the same patterns,
   - the repository README (fetched via `/repos/{owner}/{repo}/readme`) for the same patterns.
4. Writes the result to `repos.json` with the fields: `name`, `description`, `html_url`, `homepage`,
   `language`, `updated_at`, `stargazers_count`, `has_pages`, `pages_url`, `has_gitbook`, `gitbook_url`.
5. Commits and pushes `repos.json` if it changed.

Repositories without a README, homepage, or description are handled gracefully: missing fields result
in `has_gitbook: false` and `gitbook_url: ""`.

## How often is repos.json updated?

The workflow runs **daily at 03:00 UTC** (schedule: `0 3 * * *`).  
You can also trigger it manually at any time — see the next section.

## How to run the workflow manually

1. Open the repository on GitHub.
2. Go to **Actions** → **Update repositories data**.
3. Click **Run workflow** → **Run workflow**.

The workflow will regenerate `repos.json` and commit the result if anything changed.

## Page structure

`<main class="page">` contains two sections:

1. **Hero section** (`<section class="hero">`)
   - Intro text and CTA buttons.
   - Two live statistics: public repository count and distinct language count.

2. **Repositories section** (`<section class="repos" id="repositories">`)
   - A single flat tile grid (`#repo-list`) showing all public repositories.
   - A sort control with two buttons: **Last Updated** (default, newest first) and **Alphabetical** (A–Z by name).

## Repository tile grid

All public repositories are displayed together in one grid — there is no grouping by GitHub Pages or GitBook status.

Default sort order is **Last Updated** (newest first, using `updated_at` from `repos.json`).  
Click **Alphabetical** to sort A–Z by repository name. The active sort button is highlighted.

## Card layout

Each repository card shows:

- Repository name and description.
- Metadata pills: `Pages`, `GitBook`, language, `Fork`, star count, last-updated date (each shown only when applicable).
- Action buttons (always open in a new tab):
  - **repository** — links to the GitHub repository page (always present).
  - **pages** — links to the GitHub Pages URL (shown only when `has_pages` is true).
  - **gitbook** — links to the GitBook URL (shown only when `has_gitbook` is true).

## Local preview

Open `index.html` directly in a browser, or serve the repository root with any static file server:

```bash
python3 -m http.server 8080
```

## Required DOM IDs (JavaScript coupling)

The following IDs must not be renamed without also updating the JavaScript:

`repo-list`,  
`repo-status`, `repo-count`, `language-count`,  
`sort-updated`, `sort-alpha`
