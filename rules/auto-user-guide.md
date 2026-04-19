# Rule 14: Auto User Guide

**Owner: Docs agent** (`loop-docs.md`). Blog drafts are part of the Documented gate.

## Trigger

`feat:` PRs in your frontend repo with screenshots already captured.

## Where blogs publish

**Payload CMS on the landing page repo** (`YOUR_MARKETING_REPO`), NOT Webflow.

- **Endpoint**: `https://www.your-product.com/api/publish-post`
- **Auth**: `Authorization: Bearer ${PUBLISH_SECRET}` -- pull from AWS Secrets Manager at `{env}/YOUR_ORG/PayloadCMS` key `PAYLOAD_PUBLISH_SECRET` (same location the backend uses in `agentModeService.publishBlogArticle`)
- **Behavior**: If a post with the same slug exists it updates; otherwise it creates. Idempotent, so safe to retry.
- **Author**: ALWAYS `"harsha"` for docs-agent-generated posts.
- **Status**: ALWAYS `"draft"` -- you reviews in the Payload admin at `https://www.your-product.com/admin` before publishing.

## Request body

```json
{
  "title": "How to <feature action>",
  "slug": "how-to-<feature-action>",
  "summary": "<1-2 sentence hook from the PR description>",
  "bodyHtml": "<full HTML body including <style> blocks and annotated S3 image URLs>",
  "category": "User Guide",
  "author": "harsha",
  "featuredImageUrl": "<annotated after-screenshot S3 URL for the hero>",
  "publishedAt": "<ISO datetime, now>",
  "status": "draft"
}
```

Valid `category` values: `"AI in Marketing"`, `"Founder Playbooks"`, `"Growth Tips"`, `"Thought Leadership"`, `"Other"`, `"Alternatives"`, `"Listicles"`, `"User Guide"`, `"Case Studies"`. Feature-launch posts use `"User Guide"` by default.

Valid `author` values: `"harsha"`, `"rui"`, `"marketing-lead"`, `"team-lead"`, `"agent"`. Docs-agent-generated posts ALWAYS use `"harsha"`.

## Body content structure

The `bodyHtml` field should contain a standalone HTML fragment (no `<html>`, `<head>`, or `<body>` wrappers -- the endpoint strips them anyway). Structure:

```html
<style>/* any post-specific CSS */</style>
<h2>What's New</h2>
<p><!-- 1-2 sentence summary of the feature --></p>

<h2>Step-by-Step Guide</h2>
<ol>
  <li>
    <p><!-- Step 1 description --></p>
    <img src="https://YOUR_SCREENSHOT_BUCKET.s3.us-west-2.amazonaws.com/pr-{num}/{ts}/after-{step}.png" alt="<!-- step alt -->" />
  </li>
  <!-- repeat for each step -->
</ol>

<h2>Tips</h2>
<ul>
  <li><!-- edge case / power-user note --></li>
</ul>

<p>Questions? <a href="https://www.your-product.com/build">Try it yourself</a> or <a href="https://calendly.com/your-handle/30min">book a call with you</a>.</p>
```

Screenshots must be the **annotated** S3 URLs already uploaded during the Docs agent's screenshot phase (red arrow overlays pointing at the changed element -- see "Screenshot Annotations" in `loop-docs.md`).

## Flow

1. PR hits the Docs agent queue with annotated screenshots already in S3 (arrow overlays pointing at the changed element).
2. **Spawn a writeup subagent** (`general-purpose` Agent) to produce the blog content. The main Docs loop must NOT write the blog inline -- it's a chunky content task that bloats the loop's context and steals cycles from other PRs in the queue.

   Subagent prompt template (fill in the `{{...}}` placeholders):

   ```
   You are writing a draft user-guide blog post for a PR that just merged.
   Your ONLY job is to return a JSON object matching the Payload CMS schema below.
   Do NOT post it anywhere -- the parent agent handles publishing.

   PR title: {{pr_title}}
   PR URL: {{pr_url}}
   PR diff (unified): {{pr_diff}}
   PR description: {{pr_body}}
   Notion task URL: {{notion_url}}

   Annotated screenshots (S3 URLs, red arrow overlays already baked in):
   - Hero / featured: {{hero_url}}
   - Step screenshots in order:
     1. {{step_1_url}} -- {{step_1_caption_hint}}
     2. {{step_2_url}} -- {{step_2_caption_hint}}
     ... (one per step)

   Write for a non-technical YOUR_PRODUCT customer. No jargon, no PR numbers,
   no "we" / "our team". Second person: "you".

   Return EXACTLY this JSON (no markdown fence, no extra keys):

   {
     "title": "How to <feature action> -- concise, under 60 chars",
     "slug": "kebab-case-of-title",
     "summary": "1-2 sentence hook that sells why this matters",
     "category": "User Guide",
     "author": "harsha",
     "featuredImageUrl": "<hero_url from above>",
     "bodyHtml": "<full HTML fragment, no <html>/<head>/<body> wrappers, with every step screenshot embedded as an <img> tag>",
     "publishedAt": "<ISO datetime, now>",
     "status": "draft"
   }

   bodyHtml structure (MANDATORY):
   <h2>What's New</h2>
   <p>...1-2 sentences...</p>
   <h2>Step-by-Step Guide</h2>
   <ol>
     <li>
       <p>...step description in plain language...</p>
       <img src="{{step_N_url}}" alt="...what the screenshot shows..." style="max-width:100%;border-radius:8px;margin:16px 0;" />
     </li>
     ... one <li> per step ...
   </ol>
   <h2>Tips</h2>
   <ul><li>...edge case / power-user note...</li></ul>
   <p>Questions? <a href="https://www.your-product.com/build">Try it yourself</a> or <a href="https://calendly.com/your-handle/30min">book a call with you</a>.</p>

   Every annotated screenshot MUST appear in the bodyHtml as an <img> tag.
   If a screenshot isn't referenced, the subagent has failed -- retry.
   ```

3. Validate the subagent's JSON: required fields present, every `{{step_N_url}}` appears in `bodyHtml` as an `<img src=`, category/author enum values are valid.
4. Fetch the Payload secret from AWS Secrets Manager (`{env}/YOUR_ORG/PayloadCMS` key `PAYLOAD_PUBLISH_SECRET`) or from `~/.config/YOUR_BACKEND_REPO/payload.env` if cached locally.
5. POST the validated JSON to `https://www.your-product.com/api/publish-post`. Retry once on 5xx.
6. On success, the response has `{success, action, id, slug, url}`. The public URL is `https://www.your-product.com/blog/{slug}`.
7. Comment on the PR: `Docs: Published draft to Payload -> https://www.your-product.com/admin/collections/posts/{id}. Review and hit publish when ready. Screenshots: {{N}} annotated shots embedded.`
8. Comment on the Notion task with the same admin URL.
9. Set the Docs gate status to **"Documented"**.

## Screenshot requirements (non-negotiable)

- **Every PR blog draft MUST embed the annotated after-screenshots.** A blog post with no screenshots is a failure -- re-run the subagent with clearer instructions.
- **Hero / featured image** = the main "after" screenshot showing the biggest visible change (red arrow on the new element).
- **Step images** = one annotated screenshot per interaction step. If a step has no visual change (e.g., "click save"), skip the image for that step but keep the text.
- Use the S3 URLs from `s3://YOUR_SCREENSHOT_BUCKET/pr-{num}/{timestamp}/` -- these already exist from the Docs agent's screenshot phase.
- If the Docs agent's screenshot phase captured fewer than 2 annotated shots, do NOT attempt the blog draft -- the post won't be useful. Flag `Documentation blocked -- not enough screenshots`.

## Failure modes

- **401 Unauthorized**: secret is missing or wrong. Check the secrets manager path and reload. Don't fall back to Webflow -- we don't use it anymore.
- **400 Missing required fields**: title, slug, or bodyHtml is empty. Re-read the PR and regenerate.
- **500 from Payload**: likely a category/author enum mismatch. Verify the values against the list above. Retry once, then flag as "Documentation blocked -- Payload error: {msg}".

## What NOT to do

- **Don't publish with `status: "published"`** -- always draft. You reviews in the admin.
- **Don't use Webflow.** The Webflow fallback in the backend still exists for safety but the docs agent should not trigger it. If the Payload endpoint is down, flag "Documentation blocked" and stop.
- **Don't use the agent's `publish_blog_article` tool** -- that routes through the backend and may still have the Webflow path as fallback. Go direct to `/api/publish-post`.
- **Don't create an Agent chat session for this.** The old "content automation tool session" flow is deprecated. Direct POST only.
