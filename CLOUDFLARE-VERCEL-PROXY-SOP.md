# SOP — Proxying a Vercel Page onto a Client's Custom Domain via Cloudflare

**Owner:** Servify Studios
**Based on:** AJC Coaching Career School 2026 sales page (proxied onto `ankushjain.co.uk/ajc` and `ankushjain.co.uk/coaching-career-school`)
**Last updated:** June 2026

---

## Overview

Sometimes a client already owns a domain and wants a new page (built and hosted separately on Vercel) to appear under a path on their existing site — without moving DNS, without a subdomain, and without the visitor ever seeing the Vercel URL.

Example: Ankush's main site lives elsewhere, but `ankushjain.co.uk/ajc` shows our Vercel-hosted sales page, seamlessly, as if it were part of his site.

The mechanism: **Cloudflare Worker**, sitting in front of the client's domain, intercepts requests to a specific path and fetches + returns the HTML from our Vercel deployment. The browser URL bar never changes — it stays on the client's domain the entire time.

**Prerequisite:** the client's domain DNS must already be managed through Cloudflare (orange-cloud proxy active). If it isn't, this whole approach doesn't work — ask the client to move DNS to Cloudflare first, or use a different approach (actual subdomain + CNAME, no proxy).

---

## How It Works

```
Visitor → ankushjain.co.uk/ajc
              │
              ▼
      Cloudflare Worker (intercepts this path)
              │
              ▼
      fetches https://ankush-sales-page.vercel.app/
              │
              ▼
      returns that HTML to the visitor
      (URL bar still shows ankushjain.co.uk/ajc)
```

The Worker does not redirect — it fetches server-side and passes the response straight through. This is why it's invisible to the visitor and why search engines index it under the client's domain, not Vercel's.

---

## Step-by-Step Setup

### Step 1 — Get the Vercel deployment URL
Deploy the page to Vercel as normal. Note the `.vercel.app` URL (e.g. `https://ankush-sales-page.vercel.app`).

### Step 2 — Create a NEW, dedicated Cloudflare Worker
**Critical rule: one Worker per project.** Do not add a new route to an existing shared Worker that already serves a different project. Reasons:
- Keeps projects fully isolated — editing one project's proxy logic can never break another's
- Easier to debug, audit, and hand off to another developer
- Avoids one Worker growing into an unmaintainable if/else chain across unrelated clients

In the Cloudflare dashboard: **Workers & Pages → Create → Worker**. Name it after the project (e.g. `ajc-sales-page-proxy`), not a generic name like "proxy" or "worker1".

### Step 3 — Worker script
Minimal version (what this project actually uses):

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url)

    if (url.pathname.startsWith('/ajc') ||
        url.pathname.startsWith('/coaching-career-school')) {
      return fetch('https://ankush-sales-page.vercel.app/')
    }

    return fetch(request)
  }
}
```

Notes on this pattern:
- It always fetches the Vercel **root** (`/`), regardless of which sub-path under `/ajc` was requested. This is fine for a single-page site with no client-side routing. If the target Vercel project has multiple real pages/routes, the Worker needs to forward the sub-path instead (see "Multi-page variant" below).
- The `return fetch(request)` fallback passes through every other request on the domain untouched — critical, otherwise the Worker would break the rest of the client's site.

**Multi-page variant** (if the Vercel project has more than one route):
```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url)

    if (url.pathname.startsWith('/ajc')) {
      const target = 'https://ankush-sales-page.vercel.app' + url.pathname.replace('/ajc', '')
      return fetch(target + url.search)
    }

    return fetch(request)
  }
}
```

### Step 4 — Add a Worker Route
A Worker script alone does nothing until a **Route** tells Cloudflare when to invoke it.

Cloudflare dashboard → the domain → **Workers Routes** → Add route:
- Route pattern: `ankushjain.co.uk/ajc*` (the trailing `*` is required to catch sub-paths and query strings)
- Worker: select the Worker created in Step 2

Repeat for each path the project should own (this project has two: `/ajc*` and `/coaching-career-school*`).

**Isolation check:** confirm this Route only matches this project's path(s). Never use a route pattern like `ankushjain.co.uk/*` unless the Worker is genuinely meant to own the entire domain.

### Step 5 — Fix all asset paths on the source page
This is the step that gets missed and causes "page loads with no styling" bugs.

The Worker returns HTML that was authored assuming it lives at `vercel.app/`. But the browser now believes it's at `ankushjain.co.uk/ajc/`. Any **relative** path (`css/styles.css`, `assets/logo.png`) resolves against the wrong base and 404s.

**Fix:** convert every asset reference to an absolute URL pointing at the Vercel deployment:

```html
<link rel="stylesheet" href="https://ankush-sales-page.vercel.app/css/styles.css">
<img src="https://ankush-sales-page.vercel.app/assets/images/logo.webp">
<script src="https://ankush-sales-page.vercel.app/js/main.js"></script>
```

```css
background-image: url('https://ankush-sales-page.vercel.app/assets/images/hero.webp');
```

This applies to every `src`, `href` (stylesheets only, not nav links), and CSS `url()` in the project.

**Do NOT use `<base href="https://project.vercel.app/">` to solve this instead.** It looks like a one-line fix but breaks in-page anchor links — `<a href="#apply">` becomes an absolute link to `https://project.vercel.app/#apply`, taking the visitor off the client's domain entirely. We tried this on the AJC page and had to revert it. Convert paths individually instead.

### Step 6 — Test
- Visit the client-domain path directly (`ankushjain.co.uk/ajc`) — not the Vercel URL
- Check DevTools console/network tab for any 404s on images, CSS, JS
- Click every in-page anchor link (`#apply`, `#brochure`, etc.) — confirm the URL bar stays on the client's domain, does not jump to `vercel.app`
- Confirm the page is NOT accessible at a URL outside its assigned path (e.g. `ankushjain.co.uk/ajc-typo` should fall through to the client's real site, not error)

---

## Common Mistakes

| Mistake | Result | Fix |
|---|---|---|
| Adding a route to an existing shared Worker instead of creating a new one | Projects become entangled; a bug in one can break another | Always create a new, dedicated Worker per project |
| Forgetting `return fetch(request)` fallback | Entire client domain breaks, not just the proxied path | Always include the fallback as the last line |
| Relative asset paths left in place | Page loads with no styling/images (client sees broken page) | Convert all `src`/`href`/`url()` to absolute Vercel URLs |
| Using `<base href="...">` to fix asset paths | Anchor links redirect visitors off the client's domain | Convert paths individually instead of using `<base>` |
| Route pattern too broad (e.g. `domain.com/*`) | Worker intercepts the client's entire site, not just one path | Scope the route pattern to the exact path(s) needed |
| Client's domain DNS not on Cloudflare | Worker never fires — no error, just doesn't work | Confirm Cloudflare manages DNS before starting |

---

## Quick Reference

| Situation | Rule |
|---|---|
| New project needs a proxy | Create a new, separate Cloudflare Worker — never share one across projects |
| Worker route pattern | Scope tightly to the project's own path(s), always with trailing `*` |
| Vercel project has multiple routes | Forward the sub-path (`url.pathname.replace(...)`), don't always fetch `/` |
| Vercel project is a single static page | Fetching `/` unconditionally is fine |
| Asset paths after proxying | Absolute Vercel URLs only, never relative |
| Anchor link breakage | Never use `<base href>` — fix each asset path instead |
| Every Worker script | Must end with a `return fetch(request)` fallback for non-matching paths |
