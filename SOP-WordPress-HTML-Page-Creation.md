# SOP: Creating Custom HTML Pages for WordPress (Classic Editor + Elementor)

**Project:** johannesmetzler.com  
**Stack:** WordPress · Classic Editor · Manage WPAutoP plugin · Elementor (header/footer only)  
**Last updated:** 2026-06-10  
**Covers:** WordPress integration · CSS isolation · SEO · Structured data · Performance

---

## Overview

This SOP governs the creation of every custom HTML page pasted into WordPress via the Classic Editor HTML view. The goal is pixel-perfect, fully responsive pages that render cleanly **between** the Elementor header and footer without touching or breaking any global theme styles.

---

## The Master Prompt (use this at the start of every new page)

Paste this verbatim as the system-level instruction before generating any HTML. It covers both WordPress integration AND SEO requirements:

```
Act as a senior WordPress front-end developer. I am using the Classic Editor with the
"Manage WPAutoP" plugin on a site where Elementor controls the header (menu) and footer.

Modify/generate my page code to strictly follow these integration rules to ensure it fits
perfectly ONLY between the existing menu and footer:

1. NO GLOBAL STYLES: Do not target generic tags like 'body', 'html', 'p', 'h1', 'img',
   or 'a' globally. Do not include any global resets.

2. TARGETED CONTAINMENT WRAPPER: Wrap the entire HTML content in a unique container:
   <div id="claude-exclusive-sandbox">...</div>. Every single HTML element must live
   inside this div.

3. FORCE CSS OVERRIDES WITH HIGHEST SPECIFICITY: Every single CSS rule must start with
   the parent ID selector to override the theme. Additionally, use the '!important' flag
   on structural property settings, colors, backgrounds, and fonts to ensure the global
   theme styles cannot overwrite them.
   Example format:
   #claude-exclusive-sandbox { display: block; background: #ffffff !important; }
   #claude-exclusive-sandbox h1 { color: #111827 !important; font-size: 2.5rem !important; }
   #claude-exclusive-sandbox p  { color: #4b5563 !important; line-height: 1.6 !important; }

4. CLEAN DOM EXIT: Ensure all <div>, <section>, and <main> tags inside the container are
   perfectly closed. Do not use absolute positioning that could float over the Elementor
   header or hide under the Elementor footer. Use standard block layouts (Flexbox or Grid).

5. SEO — STRUCTURED DATA: Add a <script type="application/ld+json"> block inside the
   sandbox div, just before the closing </div>. Include at minimum a WebPage schema.
   Add Book schema if the page promotes a book. Add Person schema if the page features
   an author or speaker. Use real URLs (not placeholders).

6. SEO — HEADING HIERARCHY: Use exactly one H1 (the page's primary keyword). Use H2 for
   major sections. Use H3 for sub-items within a section. Never skip levels. Every heading
   must contain a relevant keyword — no generic headings like "Section 1" or "More Info".

7. SEO — IMAGES: Every <img> tag must have a descriptive alt attribute containing a
   relevant keyword. Images above the fold: no lazy loading. Images below the fold:
   add loading="lazy". Hero images must be under 200KB (use WP Media Library URL, not
   base64) for acceptable LCP scores.

8. SEO — LINKS: All external links must have rel="noopener noreferrer". Anchor text must
   be descriptive — never use "click here", "read more", or "learn more" alone.
   Internal links should use full absolute URLs on johannesmetzler.com.
```

---

## Rule 1 — Sandbox ID Wrapper

Every page must be wrapped in exactly one unique container:

```html
<div id="claude-exclusive-sandbox">
  <!-- ALL content here -->
</div>
```

**Rules:**
- The ID must be unique sitewide. Use the format `claude-exclusive-sandbox` or a project-specific variant (e.g., `jm-10xceo-sandbox`).
- **No** `<!DOCTYPE html>`, `<html>`, `<head>`, or `<body>` tags — these are already provided by WordPress.
- `<link>` tags for fonts go **above** the sandbox div (not inside it). They are valid in the WordPress body output.
- `<style>` and `<script>` tags may go inside or directly above/below the div — both work in Classic Editor HTML view.

---

## Rule 2 — CSS Structure

### 2a. No global selectors — ever

**Never write:**
```css
body { ... }
html { ... }
p { ... }
h1, h2, h3 { ... }
a { ... }
img { ... }
*, *::before, *::after { ... }
:root { ... }
```

All of these will bleed into Elementor's header, footer, and every other page on the site.

### 2b. Scope every single rule to the sandbox ID

**Always write:**
```css
#claude-exclusive-sandbox { ... }
#claude-exclusive-sandbox h1 { ... }
#claude-exclusive-sandbox p { ... }
#claude-exclusive-sandbox a { ... }
#claude-exclusive-sandbox img { ... }
#claude-exclusive-sandbox *,
#claude-exclusive-sandbox *::before,
#claude-exclusive-sandbox *::after { box-sizing: border-box !important; }
```

### 2c. CSS custom properties — define on sandbox, not :root

**Wrong:**
```css
:root { --brand-color: #c4990e; }
```

**Correct:**
```css
#claude-exclusive-sandbox {
  --brand-color: #c4990e;
  --font-body: "Inter", sans-serif;
  /* all other tokens */
}
```

Variables cascade to all children inside the sandbox automatically.

### 2d. !important placement rules

Apply `!important` to:
- All **color** properties: `color`, `background`, `background-color`, `background-image`, `border-color`
- All **font** properties: `font-family`, `font-size`, `font-weight`, `font-style`
- All **structural** properties: `display`, `position`, `width`, `max-width`, `padding`, `margin`, `flex`, `grid`, `gap`

Do **not** add `!important` to:
- `transition`, `animation`, `transform` (these are safe from theme interference)
- `cursor`, `outline`, `content` (low-conflict properties)

### 2e. Responsive breakpoints — always scoped

```css
@media (min-width: 1024px) {
  #claude-exclusive-sandbox .hero {
    grid-template-columns: 7fr 5fr !important;
  }
}
```

Never use unscoped media query blocks.

### 2f. Style tag placement

Place the `<style>` block **before** `<div id="claude-exclusive-sandbox">` in the HTML. This ensures styles load before the DOM renders.

```html
<style>
  #claude-exclusive-sandbox { ... }
</style>

<div id="claude-exclusive-sandbox">
  ...
</div>
```

---

## Rule 3 — Fonts

### Google Fonts

Use `<link>` tags placed **above** the `<style>` block:

```html
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin=""/>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap"/>

<style>
  #claude-exclusive-sandbox {
    --font-body: "Inter", system-ui, sans-serif;
    font-family: var(--font-body) !important;
  }
</style>
```

**Do not** use `@import` inside the `<style>` block — it must be at the very top of the stylesheet and can cause ordering issues inside WordPress output.

### If the site already loads fonts globally

Check the WordPress theme/Elementor settings. If the font is already loaded, skip the `<link>` tags and just reference it in the CSS. Duplicate font loading wastes bandwidth.

---

## Rule 4 — Positioning Rules

### Never use position: fixed

`position: fixed` places elements relative to the **browser viewport**, not the sandbox div. This causes decorative elements to overlap the Elementor header or footer.

**Always convert fixed → absolute:**

```css
/* WRONG — floats over Elementor header */
#claude-exclusive-sandbox .bg-layer {
  position: fixed !important;
  inset: 0 !important;
}

/* CORRECT — contained within sandbox */
#claude-exclusive-sandbox .bg-layer {
  position: absolute !important;
  inset: 0 !important;
  min-height: 100% !important;
}
```

The sandbox wrapper itself must be `position: relative` to contain absolutely-positioned children:

```css
#claude-exclusive-sandbox {
  position: relative !important;
  overflow-x: hidden !important;
}
```

### Avoid z-index conflicts with Elementor

Elementor's header uses `z-index` in the 9000–10000 range. Keep all internal z-index values under 1000. Use layering like:

```css
#claude-exclusive-sandbox .bg-layer    { z-index: 0 !important; }
#claude-exclusive-sandbox .page-wrap   { z-index: 10 !important; }
#claude-exclusive-sandbox .modal       { z-index: 100 !important; }
```

---

## Rule 5 — DOM Cleanliness

### Tag closure checklist

Before finalising any page, verify:
- Every `<div>` has a matching `</div>`
- Every `<section>` has a matching `</section>`
- Every `<main>` has a matching `</main>`
- Every `<form>` has a matching `</form>`
- The sandbox `<div id="claude-exclusive-sandbox">` is the last closing tag in the file

Use an HTML validator or run this quick grep check:
```bash
grep -c "<div" file.html   # opening divs
grep -c "</div>" file.html  # closing divs — must match
```

### No orphaned HTML document tags

If converting a standalone HTML file, **remove** these before pasting into WordPress:
- `<!DOCTYPE html>`
- `<html lang="en">`
- `<head>...</head>`
- `<body>` and `</body>`
- `</html>`

WordPress provides all of these. Duplicate tags break the page.

### Semantic elements inside blockquotes

If using `<footer>` inside a `<blockquote>` for quote attribution, explicitly reset theme footer styles:

```css
#claude-exclusive-sandbox blockquote footer,
#claude-exclusive-sandbox .quote-attr {
  border-top: none !important;
  padding: 0 !important;
  background: none !important;
}
```

---

## Rule 6 — JavaScript Scoping

All JavaScript must scope queries to the sandbox container. Never query the global document for elements that exist inside the sandbox.

**Wrong:**
```javascript
const playBtn = document.getElementById('audioPlayBtn');
document.querySelectorAll('.reveal').forEach(...);
```

**Correct:**
```javascript
(function() {
  var root = document.getElementById('claude-exclusive-sandbox');
  if (!root) return;

  var playBtn  = root.querySelector('#audioPlayBtn');
  var reveals  = root.querySelectorAll('.reveal');
  // ...
})();
```

**Why:** If WordPress loads the same script twice (e.g., page builder + classic editor on different pages), unscoped queries can grab elements from the wrong context.

---

## Rule 7 — Form Element IDs

Generic form field IDs like `id="name"` or `id="email"` frequently conflict with Elementor forms, WooCommerce, or other plugins on the same page or globally.

Always prefix IDs with a page-specific slug:

```html
<!-- WRONG -->
<input id="name" ...>
<input id="email" ...>

<!-- CORRECT -->
<input id="jm-10xceo-name" ...>
<input id="jm-10xceo-email" ...>
```

Update matching `<label for="...">` attributes accordingly.

---

## Rule 8 — Images

### Base64 embedded images

For images that must be self-contained (no external hosting), embed as base64 `src` attributes. Keep individual images under these size guidelines:

| Image type     | Max width | Max file size (base64) |
|----------------|-----------|------------------------|
| Hero/product   | 700px     | ~900 KB                |
| Portrait/photo | 900px     | ~50 KB                 |
| Total HTML     | —         | ~1 MB                  |

For large base64 files, use PowerShell or Python to inject them after writing the HTML skeleton:

```powershell
$src  = [System.IO.File]::ReadAllLines($origPath, [System.Text.Encoding]::UTF8)
$img  = $src[781].Trim()  # the full <img src="data:image/..."> tag
$dest = [System.IO.File]::ReadAllText($newPath, [System.Text.Encoding]::UTF8)
$dest = $dest.Replace('PLACEHOLDER_BOOK_IMG', $img)
[System.IO.File]::WriteAllText($newPath, $dest, [System.Text.Encoding]::UTF8)
```

### External images

Always prefer WordPress Media Library URLs over external CDN URLs for production. Upload via Media Library, copy the URL, use it directly in the `src` attribute.

### Image CSS

```css
#claude-exclusive-sandbox img {
  display: block !important;
  max-width: 100% !important;
  height: auto !important;
}
```

Never set bare `img { ... }` globally.

---

## Rule 9 — WPAutoP (Manage WPAutoP Plugin)

The "Manage WPAutoP" plugin prevents WordPress from wrapping content in auto-generated `<p>` tags (wpautop filter). **This must be active on every custom page** or WordPress will wrap `<style>`, `<script>`, and `<div>` blocks in `<p>` tags, breaking the layout.

Verify it's enabled:
1. Edit the page in WordPress
2. Check the "Disable wpautop" checkbox in the Manage WPAutoP meta box (below the editor)
3. Save

---

## Rule 10 — Elementor Canvas Template

For pages that need **zero** theme chrome (no Elementor header/footer either), set the page template to **Elementor Canvas**:

```
Page Settings → Page Attributes → Template → Elementor Canvas
```

For pages that should appear **between** the existing header and footer (standard layout), use:
```
Page Attributes → Template → Default Template
```

The 10X CEO page uses **Default Template** so the Elementor nav and footer wrap it.

---

## Rule 11 — SEO: Responsibility Split

**Critical:** WordPress handles some SEO fields. The HTML fragment handles others. Never duplicate.

### WordPress/SEO Plugin handles (Yoast SEO or RankMath — set in page editor)

| Field | Where to set |
|---|---|
| Page `<title>` tag | Yoast/RankMath → SEO Title field |
| `<meta name="description">` | Yoast/RankMath → Meta Description field |
| Open Graph title + description | Yoast/RankMath → Social tab |
| Open Graph image | Yoast/RankMath → Social tab → upload image |
| `<link rel="canonical">` | Yoast/RankMath → Advanced tab |
| Twitter Card type | Yoast/RankMath → Social tab |

**Do NOT add these to the HTML fragment** — they belong in the `<head>`, which WordPress controls. Duplicating them creates conflicting tags that confuse search engines.

Recommended SEO title format: `[Page Topic] — Dr. Johannes Metzler | Executive Coach & Author`  
Recommended meta description: 150–160 characters, includes primary keyword, ends with a value statement.

### HTML fragment handles

- JSON-LD structured data (`<script type="application/ld+json">`)
- Heading hierarchy (H1–H3)
- Image alt text
- Anchor text quality
- Semantic HTML structure
- `loading="lazy"` on below-fold images

---

## Rule 12 — SEO: JSON-LD Structured Data

Add inside the sandbox div, immediately before `</div><!-- /#claude-exclusive-sandbox -->`.

### Minimum schema for every page

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebPage",
  "name": "PAGE TITLE",
  "description": "PAGE META DESCRIPTION",
  "url": "https://johannesmetzler.com/PAGE-SLUG/",
  "author": {
    "@type": "Person",
    "name": "Dr. Johannes Metzler",
    "url": "https://johannesmetzler.com"
  }
}
</script>
```

### Book page schema (add when page promotes a book)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Book",
      "name": "BOOK TITLE",
      "author": {
        "@type": "Person",
        "name": "Dr. Johannes Metzler",
        "url": "https://johannesmetzler.com"
      },
      "datePublished": "YEAR",
      "description": "BOOK DESCRIPTION",
      "image": "https://johannesmetzler.com/wp-content/uploads/YEAR/MM/book-image.jpg",
      "url": "https://johannesmetzler.com/PAGE-SLUG/",
      "inLanguage": "en",
      "genre": "Business & Leadership"
    },
    {
      "@type": "Person",
      "name": "Dr. Johannes Metzler",
      "jobTitle": "Executive Coach & Trusted Advisor",
      "url": "https://johannesmetzler.com",
      "image": "https://johannesmetzler.com/wp-content/uploads/YEAR/MM/portrait.jpg",
      "sameAs": [
        "https://www.linkedin.com/in/johannesmetzler"
      ],
      "description": "Transformative Executive Coach and Trusted Advisor to entrepreneurs and business leaders."
    }
  ]
}
</script>
```

### Other schema types to use per page type

| Page type | Schema types to include |
|---|---|
| Book landing page | `Book` + `Person` |
| Speaking / events page | `Event` + `Person` |
| Services / coaching page | `Service` + `Person` |
| About page | `Person` + `ProfilePage` |
| Blog article | `Article` + `Person` |
| Contact page | `ContactPage` + `Organization` |

Validate all schemas at: https://search.google.com/test/rich-results

---

## Rule 13 — SEO: Heading Hierarchy

Every page must follow a strict heading order. Google uses headings to understand page structure and keyword relevance.

| Level | Rule | Example |
|---|---|---|
| H1 | Exactly **one** per page. Primary keyword. | `The 10X CEO` |
| H2 | One per major section. Contains secondary keywords. | `The 10X CEO Audio Experience` |
| H3 | Sub-items within a section. | `Inside-Out` |

**Rules:**
- Never skip levels (H1 → H3 is wrong; H1 → H2 → H3 is correct)
- Author name as a heading: use H2 only if the author section is a primary page section; otherwise H3
- Success/error messages are not headings — use `<p>` or `<strong>` instead
- Every heading must contain a real keyword, not generic text

---

## Rule 14 — SEO: Images

### Alt text — required on every `<img>`

```html
<!-- WRONG -->
<img src="book.jpg"/>
<img src="photo.jpg" alt=""/>
<img src="photo.jpg" alt="image"/>

<!-- CORRECT -->
<img src="book.jpg" alt="The 10X CEO book by Dr. Johannes Metzler"/>
<img src="photo.jpg" alt="Dr. Johannes Metzler — Executive Coach and Author"/>
```

Alt text formula: `[what it shows] — [context/keyword if relevant]`

### Lazy loading

```html
<!-- Above the fold (hero image) — NO lazy loading -->
<img src="..." alt="..." width="700" height="1050"/>

<!-- Below the fold (portrait, secondary images) — ADD lazy loading -->
<img src="..." alt="..." loading="lazy" width="900" height="1200"/>
```

"Above the fold" = anything visible without scrolling on a 1080p desktop screen. For a landing page, typically only the hero book/product image.

### Always specify width and height

Prevents Cumulative Layout Shift (CLS), which Google scores as a Core Web Vital:

```html
<img src="..." alt="..." width="700" height="1050"/>
```

Use the actual pixel dimensions of the image.

### Prefer WP Media Library URLs over base64

| Method | LCP impact | Cacheability | Recommended |
|---|---|---|---|
| WP Media Library URL | Fast (cacheable CDN) | ✓ | ✓ Production |
| base64 embedded | Slow (blocks parse) | ✗ | Development only |

For production: upload to WP Media Library → copy URL → use in `src`.

---

## Rule 15 — SEO: Links

### External links

```html
<!-- CORRECT — always -->
<a href="https://www.linkedin.com/in/johannesmetzler"
   target="_blank"
   rel="noopener noreferrer">
  Connect with Dr. Metzler on LinkedIn
</a>
```

- `target="_blank"` — opens in new tab (keeps user on the page)
- `rel="noopener noreferrer"` — required for security + prevents referrer leakage

### Anchor text quality

| Rating | Example |
|---|---|
| ✗ Bad | "Click here", "Read more", "Learn more" |
| ✓ Good | "Get The 10X CEO on Amazon", "Connect on LinkedIn", "Book a coaching call" |

Anchor text tells Google what the linked page is about. Generic text wastes that signal.

### Internal links

```html
<!-- Use absolute URLs for internal links -->
<a href="https://johannesmetzler.com/contact/?source=10xceo">Book a session</a>
```

Add `?source=PAGE-SLUG` to internal CTA links for UTM tracking.

---

## Rule 16 — SEO: Semantic HTML

Use semantic elements — they signal content structure to search engines.

| Element | Use for |
|---|---|
| `<main>` | Wrap all page content (one per page) |
| `<section aria-label="...">` | Each distinct content section |
| `<article>` | Self-contained content (blog posts, reviews) |
| `<blockquote>` | Quotes with `<footer>` or `<cite>` attribution |
| `<address>` | Contact information |
| `<time datetime="...">` | Dates and times |
| `<strong>` | Keyword emphasis within body copy |

Every `<section>` must have an `aria-label` or `aria-labelledby` attribute — it serves both accessibility and search engine crawling.

---

## Complete File Structure Template

Use this as the starting skeleton for every new page:

```html
<!-- FONTS (above sandbox) -->
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin=""/>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=YOUR_FONT&display=swap"/>

<!-- SCOPED STYLES -->
<style>

/* ---- Scoped resets ---- */
#claude-exclusive-sandbox *,
#claude-exclusive-sandbox *::before,
#claude-exclusive-sandbox *::after {
  box-sizing: border-box !important;
  margin: 0;
  padding: 0;
}

#claude-exclusive-sandbox img,
#claude-exclusive-sandbox svg,
#claude-exclusive-sandbox video {
  display: block !important;
  max-width: 100% !important;
  height: auto !important;
}

#claude-exclusive-sandbox a {
  color: inherit !important;
  text-decoration: none !important;
}

#claude-exclusive-sandbox button {
  cursor: pointer !important;
  background: none !important;
  border: none !important;
  font: inherit !important;
}

#claude-exclusive-sandbox input,
#claude-exclusive-sandbox textarea {
  font: inherit !important;
}

/* ---- Design tokens ---- */
#claude-exclusive-sandbox {
  /* brand variables */
  --color-bg:    #f2ede8;
  --color-ink:   #211d17;
  --color-accent:#c4990e;
  --font-body:   "Inter", system-ui, sans-serif;
  --font-display:"Your Display Font", Georgia, serif;
  --radius:      0.625rem;
  --max-w:       80rem;
  --px:          1.5rem;

  /* container base */
  display: block !important;
  position: relative !important;
  background-color: var(--color-bg) !important;
  color: var(--color-ink) !important;
  font-family: var(--font-body) !important;
  overflow-x: hidden !important;
  width: 100% !important;
}

@media (min-width: 640px) {
  #claude-exclusive-sandbox { --px: 2.5rem; }
}

/* ---- All other rules follow the same pattern ---- */
#claude-exclusive-sandbox .section-hero { ... }
#claude-exclusive-sandbox .section-hero h1 { color: var(--color-ink) !important; }

</style>

<!-- SANDBOX WRAPPER -->
<div id="claude-exclusive-sandbox">

  <div class="page-wrap">

    <main>

      <!-- ===== SECTION 1 ===== -->
      <section aria-label="Hero">
        ...
      </section>

      <!-- ===== SECTION N ===== -->

    </main>

  </div><!-- /.page-wrap -->

  <script>
  (function() {
    var root = document.getElementById('claude-exclusive-sandbox');
    if (!root) return;

    // All JS scoped to root
    var reveals = root.querySelectorAll('.reveal');
    // ...
  })();
  </script>

  <!-- ===== STRUCTURED DATA (SEO) ===== -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "WebPage",
    "name": "PAGE TITLE",
    "description": "PAGE META DESCRIPTION",
    "url": "https://johannesmetzler.com/PAGE-SLUG/",
    "author": {
      "@type": "Person",
      "name": "Dr. Johannes Metzler",
      "url": "https://johannesmetzler.com"
    }
  }
  </script>

</div><!-- /#claude-exclusive-sandbox -->
```

---

## Pre-Paste Checklist

Run through every item before pasting into WordPress.

### WordPress Integration
- [ ] No `<!DOCTYPE>`, `<html>`, `<head>`, or `<body>` tags present
- [ ] All content wrapped in `<div id="claude-exclusive-sandbox">`
- [ ] No CSS rule targets a bare tag globally (`body`, `html`, `p`, `h1`, etc.)
- [ ] All CSS rules start with `#claude-exclusive-sandbox`
- [ ] CSS variables defined on `#claude-exclusive-sandbox`, not `:root`
- [ ] `!important` on all color, font, and structural properties
- [ ] No `position: fixed` anywhere — converted to `position: absolute`
- [ ] `#claude-exclusive-sandbox` has `position: relative !important`
- [ ] All form field IDs are prefixed with a page slug (not `id="name"` or `id="email"`)
- [ ] All JS queries scoped to `root = document.getElementById('claude-exclusive-sandbox')`
- [ ] Div/section/main open and close tags balanced
- [ ] **Manage WPAutoP** is disabled on this page (checked in page editor)
- [ ] Page template set correctly (Default Template for header+footer, Elementor Canvas for none)
- [ ] Google Fonts `<link>` tags are above the `<style>` block
- [ ] Base64 images under size limits (~1 MB total HTML)

### SEO — HTML Fragment
- [ ] Exactly **one** H1 on the page, containing the primary keyword
- [ ] H2s used for major sections, H3s for sub-items — no skipped levels
- [ ] Every `<img>` has a descriptive `alt` attribute with a relevant keyword
- [ ] `loading="lazy"` on all images below the fold; none on hero/above-fold images
- [ ] `width` and `height` attributes on every `<img>` (prevents CLS)
- [ ] All external links have `rel="noopener noreferrer"`
- [ ] No generic anchor text ("click here", "read more") — all links are descriptive
- [ ] JSON-LD structured data block present before `</div><!-- /#claude-exclusive-sandbox -->`
- [ ] JSON-LD schema type matches page content (WebPage / Book / Event / Service)
- [ ] JSON-LD URLs are real absolute URLs (not placeholders)
- [ ] All `<section>` tags have `aria-label` attributes

### SEO — WordPress Settings (do after paste)
- [ ] Yoast/RankMath SEO Title filled in (format: `Topic — Dr. Johannes Metzler | Executive Coach & Author`)
- [ ] Yoast/RankMath Meta Description filled in (150–160 chars, includes primary keyword)
- [ ] Yoast/RankMath Social OG Image set (use WP Media Library image)
- [ ] Canonical URL confirmed (should auto-populate; verify it matches the page URL)

---

## Common Mistakes & Fixes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `position: fixed` background layers | Decorative bg overlaps Elementor header on scroll | Change to `position: absolute` + add `min-height: 100%` |
| `:root { --var }` instead of `#sandbox { --var }` | Variables undefined or overridden by theme | Move all custom props to `#claude-exclusive-sandbox { }` |
| `margin-top` on `.btn-primary` set with `!important` | Button won't align in flex row even with inline `style="margin-top:0"` | Add specific override: `#sandbox .flex-row .btn-primary { margin-top: 0 !important; }` |
| Generic `id="name"`, `id="email"` | Conflicts with Elementor/WooCommerce forms sitewide | Prefix all IDs with page slug |
| JS using `document.querySelector` | Works on one page, breaks on others if same snippet is reused | Scope all queries to `root = document.getElementById('claude-exclusive-sandbox')` |
| Pasting without disabling WPAutoP | `<style>` and `<div>` wrapped in `<p>` tags, layout breaks | Enable Manage WPAutoP on the page |
| `*.html` standalone file pasted directly | Duplicate `<html>` tag, browser parse errors | Strip doctype/html/head/body before pasting |
| Theme `footer { padding: X }` bleeding into `<footer>` inside blockquote | Extra padding appears on quote attribution | Add `#sandbox blockquote footer { padding: 0 !important; border-top: none !important; }` |
| Meta description added inside HTML fragment | Two meta descriptions in page source; Google picks randomly | Remove from HTML — set only in Yoast/RankMath |
| JSON-LD contains placeholder text (`PAGE TITLE`, `PAGE-SLUG`) | Schema fails Google Rich Results validation | Replace all placeholders with real values before paste |
| Missing `width`/`height` on images | CLS (layout shift) hurts Core Web Vitals score | Add actual pixel dimensions to every `<img>` |
| Hero image embedded as base64 (~900KB) | Slow LCP, poor PageSpeed score | Upload to WP Media Library, use URL instead |
| Generic anchor text on CTAs | Missed keyword signal on links | Use descriptive text: "Get The 10X CEO on Amazon" not "Buy now" |
| Multiple H1 tags | Google confused about primary topic | Keep exactly one H1; all other titles use H2+ |

---

## Testing After Paste

### Layout & Functionality
1. **View page frontend** — renders between Elementor header and footer cleanly
2. **Scroll test** — no decorative elements bleed above the header or below the footer
3. **Mobile test** — resize browser to 375px, 768px, 1024px, 1440px
4. **Link test** — click all CTAs, verify `href` values
5. **Interactive elements** — test audio players, forms, accordions if present
6. **Theme interference** — DevTools: no bare `p`, `h1`, `a` theme styles winning over sandbox rules

### SEO Validation
7. **Rich Results Test** — https://search.google.com/test/rich-results → paste live page URL → confirm all schemas valid, no errors
8. **PageSpeed Insights** — https://pagespeed.web.dev → score 70+ on mobile; check LCP, CLS, FID
9. **Page source check** — Ctrl+U in browser → confirm `<script type="application/ld+json">` block present in rendered HTML
10. **Heading audit** — DevTools console: `document.querySelectorAll('h1,h2,h3')` → verify one H1, logical H2/H3 order
11. **Alt text audit** — DevTools console: `document.querySelectorAll('img:not([alt])')` → must return empty list
12. **Social preview** — https://developers.facebook.com/tools/debug/ → paste URL → confirm OG title + image appear (requires Yoast OG fields filled)

---

## File Naming Convention

| File | Purpose |
|------|---------|
| `[page-name]-wordpress.html` | Standalone preview (full HTML document with doctype) |
| `[page-name]-wp-sandbox.html` | WordPress-ready fragment (sandbox wrapper only, no doctype) |

Always deliver the `-wp-sandbox.html` version to paste into WordPress. Keep the `-wordpress.html` version for local preview testing.
