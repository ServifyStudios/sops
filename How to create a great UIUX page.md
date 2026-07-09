# SOP — Sales Page UI/UX: Carousels, Videos, and Mobile

**Owner:** Servify Studios  
**Based on:** AJC Coaching Career School 2026 sales page build  
**Last updated:** June 2026

---

## Overview

This SOP captures UI/UX decisions made during the AJC sales page build. Apply these rules on future sales pages to avoid repeating the same debugging cycles.

---

## 1. Carousels — Preventing Page Jump (Height Instability)

**Problem encountered:** When a carousel auto-advances and the new slide has different content height, the page jumps up and down as the section resizes. This is disorienting for the reader.

**Fix:** Give the carousel container a fixed `min-height` that covers the tallest possible slide content. The container holds its space regardless of which slide is active.

```css
.testi-content { min-height: 440px; }
```

**Rule:** Any carousel with variable-height content (text testimonials, bios, descriptions) must have a fixed `min-height`. Measure the tallest slide during build and set that as the floor.

---

## 2. Carousels — Auto-Advance Behaviour

Three rules established on this project:

**2a. Stop when offscreen**  
Auto-advancing carousels should stop when the section is not in the viewport. Use `IntersectionObserver` to start/stop the timer. Reason: saves CPU, and avoids the carousel being on the "wrong" slide when the user scrolls back to it.

**2b. Reset to slide 1 on re-entry**  
When the user scrolls away and comes back, the carousel resets to the first slide. The first slide is the most important one — treat it like a first impression every time.

```javascript
new IntersectionObserver(function(entries) {
  entries.forEach(function(e) {
    if (e.isIntersecting) {
      goTo(0);           // always start from slide 1
      startAuto();
    } else {
      stopAuto();
    }
  });
}, { threshold: 0.15 }).observe(section);
```

**2c. Stop permanently on manual click**  
If the user manually clicks to change a slide, stop auto-advancing permanently for that session. Respect their choice — they have shown intent to browse at their own pace.

---

## 3. Carousels — Ordering Slides

The client decides the order of slides, not the developer. When adding new items:

- Ask the client: "What position in the list should this appear?"
- Default to appending at the end if client does not specify
- When inserting mid-list, renumber all `data-si` attributes and update the JS `total` constant

**On this project:** Michael McDonald was requested as 2nd in the Stories carousel. Required renumbering data-si="0–3" to data-si="0–4" and updating `const total = 4` to `const total = 5`.

---

## 4. Carousels — Touch Swipe on Mobile

All carousels must support touch swipe on mobile. Use a 40px threshold — below that, treat it as a tap, not a swipe.

```javascript
var tx0 = 0;
stage.addEventListener('touchstart', function(e) { tx0 = e.touches[0].clientX; }, { passive: true });
stage.addEventListener('touchend', function(e) {
  var dx = e.changedTouches[0].clientX - tx0;
  if (Math.abs(dx) > 40) go(dx < 0 ? 1 : -1);
});
```

Apply this to: testimonial carousels, video short carousels, faculty/masterclass carousels.

---

## 5. Videos — Format Rules

### 5a. YouTube Shorts / Portrait Videos (9:16)
- Use `shorts-facade` class on the video wrapper
- Modal opens in portrait mode (9:16 aspect ratio)
- Add `&playsinline=1` to all YouTube embed URLs (required for iOS Safari)

### 5b. Horizontal Videos (16:9) Inside a Portrait Carousel
- Use a **portrait thumbnail image** for the card (visually consistent with other cards)
- Add `data-landscape` attribute to the facade div — this signals the modal to open in 16:9 instead of 9:16
- The JS checks: `isPortrait = facade.classList.contains('shorts-facade') && !facade.hasAttribute('data-landscape')`

```html
<div class="video-facade shorts-facade" data-landscape data-src="https://www.youtube.com/embed/VIDEO_ID?autoplay=1&playsinline=1...">
  <img class="video-facade-thumb" src="portrait-thumbnail.webp" alt="Person name">
  ...
</div>
```

### 5c. Video Modals — Always Destroy iframe on Close
When the modal closes, destroy the iframe src. This stops the video playing in the background.

### 5d. Video Placement on Page
On this project:
- **"Start here" video** — placed early (after hero and vision) to introduce the school before the reader reads anything else
- **Culture/community video** — placed mid-page after the Who section, as social proof before testimonials
- **Short coach stories** — placed in a dedicated carousel after the main testimonials

---

## 5e. Accordions / Toggles — Keep Them Independent

**Problem encountered (Sarah Fox coaching page):** Three FAQ-style toggle sections ("If you are a mother", "If you are an entrepreneur", "If you are a man driven at work") were built as a mutually-exclusive accordion — opening one auto-closed whichever other was open, tracked via a single `openTab` state value.

This broke the experience: when a tall panel (e.g. mother's) was open and the user clicked a different toggle, the open panel collapsed instantly above the clicked button. Collapsing that height moved everything below it upward in the document — but the browser's scroll position (`scrollY`) doesn't move to compensate. The user experiences this as the page jumping down to reveal much later content, with no scroll animation to explain why. It reads as broken, not intentional.

**First fix attempted (wrong):** Tried to actively scroll/re-center the clicked toggle's title after the state change — first with a single delayed `scrollIntoView`, then with a continuous `requestAnimationFrame` loop re-centering the button every frame while the panels animated. Both technically worked but felt clunky and added motion the user never asked for. The user's actual ask was simpler: **don't move the screen at all.**

**Real fix:** Make each toggle independent. Track open/closed state per toggle (an object of booleans, e.g. `openTabs.mother`, `openTabs.entrepreneur`, `openTabs.man`), not a single shared "which one is open" value. Opening one toggle never closes another. Content only grows *downward* from the clicked button's own position — nothing above or at the current scroll position ever collapses, so there is nothing to compensate for and no scroll script is needed at all.

```javascript
// Wrong — mutually exclusive, causes layout-shift jump:
toggleMother: () => this.setState((s) => ({ openTab: s.openTab === 'mother' ? null : 'mother' })),

// Right — independent, no shared state, no jump:
toggleMother: () => this.setState((s) => ({
  openTabs: { ...s.openTabs, mother: !s.openTabs.mother }
})),
```

**Rule:** Any set of sibling accordions/toggles/FAQ panels on a page must open and close independently of each other by default, unless the client explicitly asks for exclusive (radio-button-style) behavior. Independent toggles avoid layout-shift scroll jumps entirely — it's a simpler fix than trying to patch the jump after the fact with scroll-anchoring or JS re-centering.

---

## 5f. Footer Copyright Year — Never Hardcode

**Rule:** The footer copyright year must always be computed at render time, never hardcoded as a static string. A hardcoded year (`© 2026`) silently goes stale every January and nobody remembers to update it across every site.

```javascript
// Wrong — goes stale next year:
<span>© 2026</span>

// Right — always current:
currentYear: new Date().getFullYear(),
```
```html
<span>© {{ currentYear }}</span>
```

Apply this on every site footer, no exceptions.

---

## 6. Mobile Navigation

**What worked:**
- Simple `classList.toggle('is-open')` — no body scroll lock, no position:fixed on body
- Hamburger always shows 3 stripes — no X animation when open. The open state is obvious from the drawer being visible; the icon does not need to change
- Hamburger z-index above the drawer (101 vs 99)
- Backdrop tap (clicking outside the drawer links) closes the menu

**What broke and why:**
- `backdrop-filter` on `<nav>` creates a CSS containing block — `position: fixed` children (the mobile drawer) position relative to the nav element instead of the viewport. The drawer appeared 57px tall (the nav height) instead of full screen. Fix: remove `backdrop-filter` on mobile, use a solid background colour instead

---

## 7. Images — Always Compress Before Deploy

All images must be converted to WebP before committing. On this project, uncompressed images (64MB total) caused Vercel bandwidth to reach 22.75GB — 99.9% of the account's usage.

**Process:**
1. Drop raw images into `assets/images/`
2. Run `node compress-images.js` (sharp-based script in repo root)
3. Commit only the `.webp` output files
4. Update all `src` and CSS `url()` references to use `.webp`

**Results on AJC project:** 64.4MB → 2.3MB (−96%)

Worst offenders if uncompressed: hero/carousel images (can be 2–12MB each as JPG/PNG).

---

## 8. Asset Paths — Cloudflare Proxy Projects

If the page is served through a Cloudflare Worker proxy (different domain than Vercel), all asset paths must be **absolute URLs** pointing to the Vercel deployment:

```html
<img src="https://your-project.vercel.app/assets/images/photo.webp">
```

```css
background-image: url('https://your-project.vercel.app/assets/images/hero.webp');
```

**Do NOT use `<base href="...">` for this.** The base tag breaks anchor links (`#apply`, `#brochure`, etc.) — clicking them navigates to the Vercel URL directly, taking the user off the proxy domain.

---

## Quick Reference

| Situation | Rule |
|---|---|
| Carousel content varies in height | Set `min-height` on container = tallest slide |
| Carousel auto-advances | Stop when offscreen, reset to slide 1 on re-entry, stop permanently on manual click |
| Adding new carousel item | Ask client for position; renumber data-si; update JS total constant |
| Mobile carousel | Always add touch swipe, 40px threshold |
| Sibling accordions/toggles/FAQ panels | Make independent (per-item open state), not mutually exclusive — avoids scroll-jump when one panel's collapse shifts layout above the viewport |
| Footer copyright year | Compute with `new Date().getFullYear()`, never hardcode — updates automatically every year |
| Portrait (Shorts) video | `shorts-facade` class, portrait modal |
| Horizontal video in portrait carousel | Portrait thumbnail + `data-landscape` attr, 16:9 modal |
| All YouTube embeds | Add `&playsinline=1` (iOS Safari) |
| Images before deploy | Convert to WebP via compress-images.js |
| Cloudflare proxy | Absolute Vercel URLs on all assets, no `<base>` tag |
| Mobile nav | No backdrop-filter on nav, 3-stripe hamburger always visible |
