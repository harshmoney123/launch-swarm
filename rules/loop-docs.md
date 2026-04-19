# Docs -- "docs" or "document prs" or "ship it"

`/loop 20m`. **Pipeline gate** -- no PR merges without Documented status.

## The Gate

Watchdog only auto-merges when BOTH exist:
1. Senior Reviewer: Approve
2. **Docs: Documented**

## Each tick

1. Find open PRs targeting `YOUR_DEV_BRANCH` with Senior Reviewer Approve but no Docs Documented. Also check mega PRs (`YOUR_DEV_BRANCH` -> `main`) -- document them but **never auto-merge**.
2. Skip PRs without Senior Reviewer Approve (code review comes first).
3. **Backend-only PRs** (backend repo): auto-post "Documented -- backend API, no visual changes."

For each undocumented customer portal PR:

### Screenshots (all PRs)
- Navigate affected pages on `YOUR_DEV_BRANCH` base via Playwright -> `before-{pr}-{page}.png`
- Checkout PR branch -> `after-{pr}-{page}.png`
- **Annotate "after" screenshots** -- see "Screenshot Annotations" below
- Run happy-path interaction (fill forms, click, submit)
- Upload to S3: `s3://YOUR_SCREENSHOT_BUCKET/pr-{num}/{timestamp}/`
- Comment on PR with before/after images

### Blog Draft (`feat:` PRs only)
- **The blog writeup happens in a subagent**, not inline in the Docs loop. Spawn a `general-purpose` Agent with the PR diff + annotated S3 screenshot URLs + the Rule 14 template, have it return validated JSON, then POST. Keeps the Docs loop lean so it can handle multiple PRs per tick.
- **Publish directly to Payload CMS** on the landing page repo, NOT Webflow, NOT via Agent session.
- Endpoint: `POST https://www.your-product.com/api/publish-post` with `Authorization: Bearer ${PUBLISH_SECRET}` (pull from AWS Secrets Manager at `{env}/YOUR_ORG/PayloadCMS` key `PAYLOAD_PUBLISH_SECRET`).
- Always set `"author": "harsha"` and `"status": "draft"`. You reviews at `https://www.your-product.com/admin` before publishing.
- Build `bodyHtml` with annotated S3 screenshot URLs inline (red arrow overlays per the "Screenshot Annotations" section below).
- Full request body format, category/author enums, failure modes, and flow: see `auto-user-guide.md`.
- After a successful POST, comment on PR + Notion task with `https://www.your-product.com/admin/collections/posts/{id}` so you can open the draft directly.

### Approve or Block
- All evidence captured -> **"Documented"**
- UI is broken -> **"Documentation blocked -- [what's wrong]"** (blocks merge)

## Screenshot Annotations

All "after" screenshots, blog post images, and Feature Launch images MUST be annotated with red arrows pointing at the changed/new elements. This makes it obvious what changed without reading text.

### How to annotate (Playwright)

1. Read the PR diff to identify which components/elements changed.
2. Navigate to the page via Playwright.
3. Use `page.locator()` to find the changed element, get its bounding box.
4. Inject a red arrow SVG overlay pointing at the element:

```js
const box = await page.locator('.changed-element').boundingBox();
await page.evaluate(({x, y, w, h}) => {
  const svg = document.createElement('div');
  svg.style.cssText = 'position:fixed;top:0;left:0;width:100%;height:100%;z-index:99999;pointer-events:none';
  // Arrow starts 80px above-left of the element, points to its center
  const tipX = x + w/2;
  const tipY = y + h/2;
  const startX = tipX - 80;
  const startY = tipY - 100;
  svg.innerHTML = `<svg width="100%" height="100%" style="position:absolute;top:0;left:0">
    <defs><marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="red"/>
    </marker></defs>
    <line x1="${startX}" y1="${startY}" x2="${tipX}" y2="${tipY}"
      stroke="red" stroke-width="5" marker-end="url(#arrowhead)"/>
  </svg>`;
  document.body.appendChild(svg);
}, {x: box.x, y: box.y, w: box.width, h: box.height});
```

5. Take the screenshot -- arrow is baked into the image.

### Rules
- One arrow per changed element. Multiple changes = multiple arrows.
- Arrow should be large and obvious (red, 5px+ stroke).
- Don't obscure the element itself -- point at it from above-left or above-right.
- "Before" screenshots are never annotated -- only "after" and blog/feature-launch images.

## Feature Launch (after mega PR merge)

When the Watchdog detects a mega PR merged to `main`:

1. **Filter**: Only include `feat:` PRs that affect customer-facing UX. Skip refactors, bug fixes, infra, backend-only, test changes.
2. **Compile**: For each feature, write a non-technical explanation with step-by-step "how to use" instructions. Audience is sales/support/leadership -- no PR numbers, no technical jargon.
3. **Screenshots**: Use the annotated screenshots already captured during the PR documentation phase. If missing, take new annotated screenshots.
4. **Post to Discord**: Send to your feature announcement channel via webhook (see `reference-ids.md`).

### Discord message format

```
## New in YOUR_PRODUCT -- {date}

---

**Feature name in plain language**

{1-2 sentence explanation of what changed and why it matters}

**How to use:**
1. {Step}
2. {Step}
3. {Step}

{S3 screenshot URL -- annotated, Discord will inline it}

---

Questions or feature requests? Drop them here in #internal-feature-request
```

### Discord webhook call

```bash
curl -H "Content-Type: application/json" -d '{
  "username": "YOUR_PRODUCT Releases",
  "content": "{formatted message with image URLs}"
}' "YOUR_DISCORD_WEBHOOK_URL"
```

## Mega PR Walkthrough

During "prep mega pr": log in as admin + basic user, screenshot every changed page, post on mega PR.

**Test env**: `./test-env/test-env.sh status` -> customer portal at `http://{IP}:YOUR_PORT`.
