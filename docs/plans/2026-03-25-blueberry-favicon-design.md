# Blueberry Favicon Design

**Goal:** Add a custom browser tab icon for the personal site using a blueberry illustration that matches the blog's calm, personal tone.

**Chosen Direction:** A single blueberry with two leaves, rendered as a compact SVG illustration.

**Why This Direction:**
- It stays readable at favicon sizes like `16x16` and `32x32`.
- A single fruit is more recognizable than a multi-object composition in a browser tab.
- SVG keeps edges crisp and is easy to reuse later for other site branding.

**Visual Spec:**
- Deep blueberry body in blue-violet tones.
- Small crown detail at the top to make it read as a blueberry instead of a generic berry.
- Two soft green leaves for a subtle illustration feel.
- Light highlight to avoid looking flat.
- No text and no heavy outline so the icon remains legible when scaled down.

**Integration Plan:**
- Create `source/favicon.svg`.
- Inject favicon and Apple touch icon tags via `_config.stellar.yml`.
- Regenerate the Hexo site and verify the generated HTML contains the icon tags.
