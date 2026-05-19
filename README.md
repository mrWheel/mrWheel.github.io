# mrWheel.github.io landing page documentation

This repository is a single-page static site served from `index.html`.
Use this document as the source of truth for how the page works and what must stay aligned when editing.

## Runtime model

- No build step, bundler, or framework.
- `index.html` contains:
  - all HTML markup,
  - all CSS styles (`<style>` in `<head>`),
  - all JavaScript logic (single IIFE in `<script>` at the end of `<body>`).
- The page runs fully in the browser.

## Page structure

`<main class="page">` contains two sections:

1. Hero section (`<section class="hero">`)
   - Intro text.
   - CTA buttons:
     - GitHub profile link.
     - Anchor link to repositories section (`#repositories`).
   - Two live statistics:
     - `#repo-count` = number of rendered public repositories (excluding the landing-page repo).
     - `#language-count` = number of distinct non-empty `language` values.

2. Repositories section (`<section class="repos" id="repositories">`)
   - Status element: `#repo-status`.
   - Three groups, each with its own list:
     - `#pages-group` with `#pages-repo-list`
     - `#gitbook-group` with `#gitbook-repo-list`
     - `#other-group` with `#other-repo-list`
   - `<noscript>` fallback message for users without JavaScript.

## Data sources and loading flow

The script follows this order:

1. Fetch live repositories:
   - `GET https://api.github.com/users/{username}/repos?per_page=100&sort=updated`
2. For each repo, verify GitHub Pages availability through:
   - `GET https://api.github.com/repos/{username}/{repo}/pages`
3. **Looking up the Description**: for repositories with missing/placeholder descriptions, display status `"Looking up the Description…"` and fetch details through:
   - `GET https://api.github.com/repos/{username}/{repo}`
4. Render grouped cards.
5. If live fetch fails, use bundled `fallbackRepos` snapshot embedded in `index.html`.
6. If verification of fallback data fails, render the fallback snapshot directly (with local description normalization).

When fallback data is used, the status message becomes:
- `Showing a bundled repository snapshot.`

## Looking up the Description

Between the Pages-availability check and the final render, the script runs `enrichMissingDescriptions(repos)`.
During this phase the status bar shows **"Looking up the Description…"**.

For each repository whose `description` field is `null`, an empty string, or a known placeholder
(`"null"`, `"undefined"`, `"no description available"`, `"no description provided."`), the function
calls `GET https://api.github.com/repos/{username}/{repo}` to retrieve the full repository object
and extract its description.

- If the call succeeds and the response carries a non-empty, non-placeholder description, that value
  replaces the missing one before the card is rendered.
- If the call fails (network error, rate limit, non-OK status), the repository is rendered without a
  description and the card falls back to displaying `"No description provided."`.
- The `fallbackRepos` snapshot bundled inside `index.html` is kept up-to-date with known descriptions
  so that the fallback path shows correct information even when the per-repo API calls cannot be made.

## Repository classification rules

Before rendering:

- Exclude private repositories.
- Exclude the landing page repository itself:
  - repo name exactly equals `{username}.github.io` (case-insensitive).

Each remaining repository is normalized with a computed `gitbook_url` and then can appear in:

- **GitHub Pages group**: `repo.has_pages === true`
- **GitBook group**: `repo.gitbook_url` is non-empty
- **Other group**: neither Pages nor GitBook

Important: a repository can appear in both Pages and GitBook groups if it satisfies both conditions.

## Link destination rules

Cards are no longer clickable as a whole. Each card has action buttons:

- `repository` button:
  - always uses `repo.html_url` (GitHub repository page).
- `gitbook` button (GitBook cards):
  - uses `repo.gitbook_url`.
- `pages` button (Pages cards):
  - uses `repo.pages_url` when available, normalized to end in `/index.html`.
  - else falls back to `https://{username}.github.io/{repo}/index.html`.
- Repositories without GitBook or Pages only get a single `repository` button.

## GitBook detection rules

`getGitbookUrl(repo)` resolves in this order:

1. `knownGitbookUrls[repo.name]` hardcoded overrides.
2. Regex scan in `repo.homepage` and `repo.description` for `gitbook.io` URL.
3. `null` if no match.

Trailing punctuation is trimmed from detected URLs.

## Known overrides in code

Two override maps are intentionally hardcoded and must stay current:

- `knownGitbookUrls`: manual GitBook URLs for selected repositories.
- `knownPagesIndexUrls`: manual Pages URL fixes for repositories that need explicit `index.html`.

## Rendered card content

Each card includes:

- repository name,
- description (trimmed/normalized; placeholders such as `null`, `undefined`, and `"No description available"` are treated as missing; the script then attempts a live API lookup — see **Looking up the Description**; if the lookup also yields nothing, `"No description provided."` is used as fallback),
- metadata pills:
  - `GitBook` pill when rendering GitBook destination,
  - `Pages` pill for Pages destination,
  - language pill when `repo.language` exists,
  - `Fork` pill when `repo.fork === true`,
  - star count (`★ {stargazers_count}`),
  - last updated date (`Updated {localized date}`),
- action buttons:
  - always `repository`,
  - plus `gitbook` for GitBook cards or `pages` for Pages cards.

All action buttons open in a new tab with `rel="noreferrer"`.

## Required DOM IDs and coupling with JavaScript

The script depends on these IDs and they must not be renamed without updating JavaScript:

- `pages-repo-list`, `gitbook-repo-list`, `other-repo-list`
- `pages-group`, `gitbook-group`, `other-group`
- `repo-status`
- `repo-count`, `language-count`

## Maintenance checklist for functional parity

When updating this landing page, verify:

1. HTML IDs used by JavaScript still exist.
2. Repository grouping rules are unchanged (or README is updated in the same change).
3. Link selection logic still routes to the correct Pages/GitBook/GitHub URL.
4. Fallback behavior still works when GitHub API calls fail.
5. `knownGitbookUrls`, `knownPagesIndexUrls`, and `fallbackRepos` reflect intended data.

## Local preview

Because this is a static file, open `index.html` directly in a browser or serve the repository root with any static file server.
