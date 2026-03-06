# Confluence ↔ GitHub Sync Setup Guide

## Introduction

This guide describes how to create a live sync between your Confluence pages and a GitHub repository. The idea is simple: your Confluence pages live as Markdown files in GitHub, and every time you edit and push a change, a GitHub Action automatically updates the corresponding Confluence page — no manual copy-pasting required.

This setup is useful for teams who want a single source of truth for their documentation, prefer editing in a code-friendly environment, or want version control and change history on their Confluence content.

The full setup involves a few moving parts — Git, GitHub, the Confluence API, and GitHub Actions — but you don't need to be an engineer to get it working. This guide walks through every step.

---

### 💡 Recommended: Let Claude guide you through this

The fastest and easiest way to complete this setup is to **upload this file to [Claude.ai](https://claude.ai) and ask Claude to walk you through it step by step.**

Claude can:
- Answer questions about any step in plain language
- Adapt instructions to your specific operating system and setup
- Create and push the GitHub Actions workflow file to your repo directly (no copy-pasting YAML)
- Help you troubleshoot errors as they come up in real time
- Load your Markdown files into the chat so you can edit and push them without leaving the browser

To get started, go to [claude.ai](https://claude.ai), start a new conversation, upload this file, and say something like:

> *"I want to set up a GitHub ↔ Confluence sync. Can you walk me through this guide step by step?"*

Claude will take it from there.

---

This guide walks you through the full setup to:
1. Export Confluence pages as Markdown files into a local Git repo
2. Edit them locally or via Claude
3. Push changes to GitHub and have Confluence updated automatically via GitHub Actions

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install Git](#2-install-git)
3. [Set Up Your Local Repository](#3-set-up-your-local-repository)
4. [Export Confluence Pages to Markdown](#4-export-confluence-pages-to-markdown)
5. [Create a GitHub Repository](#5-create-a-github-repository)
6. [Generate a Confluence API Token](#6-generate-a-confluence-api-token)
7. [Add GitHub Secrets](#7-add-github-secrets)
8. [Add the GitHub Actions Workflow](#8-add-the-github-actions-workflow)
9. [Day-to-Day Workflow](#9-day-to-day-workflow)
10. [Extending to Multiple Pages](#10-extending-to-multiple-pages)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Prerequisites

Before starting, make sure you have:

- A **GitHub account** → [github.com/signup](https://github.com/signup)
- An **Atlassian/Confluence account** with edit access to the pages you want to sync
- **Node.js 18+** installed (used by the GitHub Action) → [nodejs.org](https://nodejs.org)
- A terminal / command line app:
  - **Windows**: PowerShell or Git Bash
  - **Mac**: Terminal or iTerm2
  - **Linux**: Any terminal emulator

---

## 2. Install Git

### Windows
1. Download the installer from [git-scm.com/download/win](https://git-scm.com/download/win)
2. Run the installer, accepting defaults
3. Verify: open PowerShell and run:
   ```bash
   git --version
   ```

### macOS
Git comes pre-installed on most Macs. Verify with:
```bash
git --version
```
If not installed, you'll be prompted to install Xcode Command Line Tools. Alternatively, install via Homebrew:
```bash
brew install git
```

### Linux (Ubuntu/Debian)
```bash
sudo apt update && sudo apt install git -y
git --version
```

### Linux (Fedora/RHEL)
```bash
sudo dnf install git -y
```

### Configure Git (all platforms)
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Full Git config reference → [git-scm.com/docs/git-config](https://git-scm.com/docs/git-config)

---

## 3. Set Up Your Local Repository

### Option A: Clone an existing GitHub repo
```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO
```

### Option B: Create a new local repo and connect it to GitHub
```bash
mkdir my-confluence-docs
cd my-confluence-docs
git init
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
```

---

## 4. Export Confluence Pages to Markdown

Confluence doesn't natively export to Markdown, so you have two options:

### Option A: Manual Export via Browser (simple)

1. Open the Confluence page you want to sync
2. Click the **"..."** (more actions) menu → **Export** → **Export to Word**
3. Use a tool like [Pandoc](https://pandoc.org) to convert `.docx` to `.md`:

   **Install Pandoc:**

   - **Windows**: Download from [pandoc.org/installing.html](https://pandoc.org/installing.html) or use:
     ```powershell
     winget install JohnMacFarlane.Pandoc
     ```
   - **macOS**:
     ```bash
     brew install pandoc
     ```
   - **Linux**:
     ```bash
     sudo apt install pandoc -y
     ```

   **Convert:**
   ```bash
   pandoc your-file.docx -o your-page.md
   ```

### Option B: Use the Confluence REST API (automated)

```bash
CONFLUENCE_BASE_URL="https://yourcompany.atlassian.net"
CONFLUENCE_EMAIL="you@example.com"
CONFLUENCE_API_TOKEN="your_api_token"
PAGE_ID="1234567890"

curl -u "$CONFLUENCE_EMAIL:$CONFLUENCE_API_TOKEN" \
  "$CONFLUENCE_BASE_URL/wiki/rest/api/content/$PAGE_ID?expand=body.storage" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['body']['storage']['value'])" \
  > page.html

pandoc page.html -f html -t markdown -o page.md
```

**Windows (PowerShell):**
```powershell
$base = "https://yourcompany.atlassian.net"
$email = "you@example.com"
$token = "your_api_token"
$pageId = "1234567890"
$creds = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("${email}:${token}"))

$response = Invoke-RestMethod `
  -Uri "$base/wiki/rest/api/content/$pageId?expand=body.storage" `
  -Headers @{ Authorization = "Basic $creds" }

$response.body.storage.value | Out-File page.html
pandoc page.html -f html -t markdown -o page.md
```

### Naming Convention

Name your files using the Confluence page ID:

```
4646306093-product-one-pager-ps-plus-on-cvd-605.md
```

Add a metadata header at the top of each file:

```markdown
# Page Title

**Path:** Parent Page > This Page (id:4646306093)
**Created:** 2025-10-15
**Updated:** 2025-10-22
---

(your content here)
```

---

## 5. Create a GitHub Repository

1. Go to [github.com/new](https://github.com/new)
2. Name your repo (e.g., `pm_one_pagers`)
3. Set visibility to **Private** (recommended for internal docs)
4. Click **Create repository**
5. Push your local files:
   ```bash
   git add .
   git commit -m "Initial commit: add one-pager markdown files"
   git push -u origin main
   ```

GitHub repo settings reference → [docs.github.com/en/repositories](https://docs.github.com/en/repositories)

---

## 6. Generate a Confluence API Token

1. Go to [id.atlassian.net/manage-profile/security/api-tokens](https://id.atlassian.net/manage-profile/security/api-tokens)
2. Click **Create API token**
3. Give it a label (e.g., `GitHub Sync`)
4. Copy the token immediately — you won't be able to see it again

Atlassian API token docs → [support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/)

---

## 7. Add GitHub Secrets

1. Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** and add:

| Secret Name | Value |
|---|---|
| `CONFLUENCE_BASE_URL` | `https://yourcompany.atlassian.net` |
| `CONFLUENCE_EMAIL` | Your Atlassian login email |
| `CONFLUENCE_API_TOKEN` | The token you generated in Step 6 |

GitHub Secrets docs → [docs.github.com/en/actions/security-guides/encrypted-secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)

---

## 8. Add the GitHub Actions Workflow

> **Using Claude with the GitHub MCP integration?** You can skip this step. Claude can create and push the workflow file directly to your repo — just ask it to set up the Confluence sync workflow and provide your Confluence URL and page ID. Claude will commit `.github/workflows/sync-to-confluence.yml` automatically.

**If setting up manually**, create `.github/workflows/sync-to-confluence.yml`:

```yaml
name: Sync One-Pager to Confluence

on:
  push:
    branches:
      - main
    paths:
      - '4646306093-product-one-pager-ps-plus-on-cvd-605.md'

jobs:
  sync-to-confluence:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install marked
      - name: Convert Markdown and Push to Confluence
        env:
          CONFLUENCE_BASE_URL: ${{ secrets.CONFLUENCE_BASE_URL }}
          CONFLUENCE_EMAIL: ${{ secrets.CONFLUENCE_EMAIL }}
          CONFLUENCE_API_TOKEN: ${{ secrets.CONFLUENCE_API_TOKEN }}
          CONFLUENCE_PAGE_ID: '4646306093'
        run: |
          node << 'EOF'
          const fs = require('fs');
          const { marked } = require('marked');
          const raw = fs.readFileSync('4646306093-product-one-pager-ps-plus-on-cvd-605.md', 'utf8');
          const cleaned = raw.replace(/^\*\*(Path|Created|Updated):.*\n/gm, '').replace(/^---\n/m, '');
          const html = marked.parse(cleaned);
          const auth = Buffer.from(`${process.env.CONFLUENCE_EMAIL}:${process.env.CONFLUENCE_API_TOKEN}`).toString('base64');
          async function run() {
            const r = await fetch(`${process.env.CONFLUENCE_BASE_URL}/wiki/rest/api/content/${process.env.CONFLUENCE_PAGE_ID}?expand=version`, { headers: { Authorization: `Basic ${auth}` } });
            if (!r.ok) { console.error(await r.text()); process.exit(1); }
            const page = await r.json();
            const u = await fetch(`${process.env.CONFLUENCE_BASE_URL}/wiki/rest/api/content/${process.env.CONFLUENCE_PAGE_ID}`, { method: 'PUT', headers: { Authorization: `Basic ${auth}`, 'Content-Type': 'application/json' }, body: JSON.stringify({ version: { number: page.version.number + 1 }, title: page.title, type: 'page', body: { storage: { value: html, representation: 'storage' } } }) });
            if (!u.ok) { console.error(await u.text()); process.exit(1); }
            console.log('✅ Confluence page updated!');
          }
          run().catch(e => { console.error(e); process.exit(1); });
          EOF
```

GitHub Actions docs → [docs.github.com/en/actions](https://docs.github.com/en/actions)

---

## 9. Day-to-Day Workflow

```
Edit .md file  →  git add / commit / push  →  GitHub Action runs  →  Confluence updated ✅
```

```bash
git pull
# edit your file
git add your-file.md
git commit -m "Update one-pager"
git push
```

Monitor runs: GitHub repo → **Actions** tab.

---

## 10. Extending to Multiple Pages

```yaml
on:
  push:
    branches: [main]
    paths: ['**.md']
```

Parse the page ID dynamically from the filename prefix:

```javascript
const pageId = path.basename(file).split('-')[0];
```

Use `tj-actions/changed-files@v44` to get the list of changed files per push.

---

## 11. Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| Action not triggering | File path mismatch | Check exact filename in workflow `paths:` |
| `401 Unauthorized` | Wrong email or expired token | Regenerate at [id.atlassian.net](https://id.atlassian.net/manage-profile/security/api-tokens) |
| `404 Not Found` | Wrong page ID | Double-check the ID in the Confluence URL |
| `409 Conflict` | Version mismatch | Check for concurrent edits |
| Tables not rendering | Marked version | Use `marked` v9+ |
| Push rejected | Wrong remote | Run `git remote -v` |

---

## Useful Links

- Git → [git-scm.com/downloads](https://git-scm.com/downloads)
- Git config → [git-scm.com/docs/git-config](https://git-scm.com/docs/git-config)
- GitHub new repo → [github.com/new](https://github.com/new)
- GitHub Actions → [docs.github.com/en/actions](https://docs.github.com/en/actions)
- GitHub Secrets → [docs.github.com/en/actions/security-guides/encrypted-secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- Atlassian API Token → [id.atlassian.net/manage-profile/security/api-tokens](https://id.atlassian.net/manage-profile/security/api-tokens)
- Confluence REST API → [developer.atlassian.com/cloud/confluence/rest/v1/intro](https://developer.atlassian.com/cloud/confluence/rest/v1/intro/)
- Pandoc → [pandoc.org/installing.html](https://pandoc.org/installing.html)
- Node.js → [nodejs.org](https://nodejs.org)
