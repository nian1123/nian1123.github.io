# Blueberry Favicon Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a blueberry illustration favicon to the personal Hexo site and wire it into the generated page head.

**Architecture:** The site already supports custom head injection through `_config.stellar.yml`. The simplest approach is to place the favicon asset in `source/` so Hexo copies it to the site root, then inject standard `<link rel="icon">` and `<link rel="apple-touch-icon">` tags without modifying theme files.

**Tech Stack:** Hexo 8, Stellar theme, YAML config, SVG asset

---

### Task 1: Add favicon asset

**Files:**
- Create: `source/favicon.svg`

**Step 1: Create the SVG favicon**

Draw a compact blueberry icon with:
- one main berry body
- one berry crown detail
- two leaves
- one small highlight

**Step 2: Verify asset path**

Run: `ls source/favicon.svg`
Expected: file exists

### Task 2: Inject favicon tags

**Files:**
- Modify: `_config.stellar.yml`

**Step 1: Add head tags**

Append to `inject.head`:
- `<link rel="icon" type="image/svg+xml" href="/favicon.svg">`
- `<link rel="apple-touch-icon" href="/favicon.svg">`

**Step 2: Sanity check config**

Run: `sed -n '1,120p' _config.stellar.yml`
Expected: injected link tags appear under `inject.head`

### Task 3: Build and verify output

**Files:**
- Verify generated: `public/index.html`

**Step 1: Generate site**

Run: `npm run build`
Expected: Hexo generation completes successfully

**Step 2: Verify generated HTML**

Run: `rg -n "favicon.svg|apple-touch-icon" public/index.html`
Expected: generated head contains favicon tags
