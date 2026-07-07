---
name: content-manager
description: Veronica Wilen's end-to-end social-media workhorse. Use whenever the user wants to turn a source (a YouTube URL, article, TikTok, PDF, market report, listing info, a topic/idea, or a research question) into on-brand posts and then publish or schedule them across Facebook and Instagram via Blotato. YouTube is a video SOURCE only — we do not auto-publish to her YouTube channel. Handles transcript/article extraction, platform-tailored captions, calendar-aware batch scheduling, and media. Calls the brand-voice skill for every caption.
---

# Content Manager — Veronica Wilen

The pipeline: **Extract → Draft → Review → Publish/Schedule → Report.**
Give minimal input; produce a full set of on-brand, compliant, scheduled posts.

## Step 0 — Load context every run
1. Read **`brand/brand-profile.md`** (market, accounts, IDs, cadence, compliance).
2. Apply the **brand-voice** skill for ALL caption writing — its compliance gate is mandatory.
3. Confirm live accounts + required fields with **`blotato_list_accounts`** before any posting
   (IDs in the profile are reference only; never post to a stale ID).

## Inputs this skill accepts
- **YouTube URL** → **primary source for video content.** Veronica's short/long videos live on
  YouTube; Blotato scrapes the transcript directly (`sourceType: youtube`). No Drive/CLI needed.
- **TikTok / article / tweet / PDF / audio URL** → extract via Blotato.
- **Raw topic or idea** (e.g. "why Nocatee spring inventory is up") → write from scratch.
- **Research question** → `perplexity-query` source for fresh facts (verify before posting).
- **A listing** (new listing / price drop / open house / just sold) → **use the `listing-intake`
  skill**, which reads the shared Drive folder (photos + a note), builds a branded photo reel, and
  hands the facts here to caption + schedule. Never invent or alter listing prices/facts (§10).
- **Listing / market data the user pastes directly** → caption from what they gave; verify figures.
- **Local video files** → upload as media (see Media, below).
  > Video default path is **YouTube URLs** (Blotato scrapes the transcript). Google Drive is used for
  > **listing intake** (see `listing-intake`); don't pull other content from Drive unless asked.

## Step 1 — Extract source content
Use **`blotato_create_source`** with the right `sourceType`
(`youtube` | `article` | `tiktok` | `twitter` | `pdf` | `audio` | `text` | `perplexity-query`).
- Pass `customInstructions` to focus extraction (e.g. "pull the 5 key takeaways a local
  homebuyer would care about").
- It polls up to 20s. If it returns an in-progress status/id, poll **`blotato_get_source_status`**
  (wait ≥10s between polls) until `completed`. Use the returned `title` + `content`.
- For a plain topic with no URL, skip extraction and write directly from the idea.

## Step 2 — Decide platforms & format
Publishing targets are **Facebook + Instagram only.** We do **NOT** auto-publish to YouTube —
Veronica manages her own channel. (A YouTube URL is a *source* to repurpose into FB/IG posts, per
Step 1, never a destination.) Respect media rules (profile §5):
- **Facebook** — text-forward hub; text-only is fine; media optional.
- **Instagram** — **requires media** (reel/story). No media → either request it, or offer to
  generate a visual (`blotato_create_visual`). Don't silently drop IG.

Default output: **1 strong, platform-tailored caption per target platform per source.**
If the user wants options ("give me 5 for Facebook"), produce variations.

## Step 3 — Draft captions (via brand-voice)
For each platform, write in its shape (profile §6): FB = 3–6 short lines + ≤3 hashtags;
IG = hook-first + 8–15 hashtags + needs media; YT = title + description.
Run every draft through the brand-voice **compliance gate** (Fair Housing, disclosure,
no overpromising). Rotate CTAs + hashtag sets. Lean into her **Nocatee authority**.

## Step 4 — Review (always, before anything goes out)
Present drafts **grouped by platform, numbered**, clean enough to scan. Then ask what to do:
publish now, schedule, edit, or discard. **Never publish or schedule without explicit approval**
— posting is public and hard to undo. (See Safety.)

## Step 5 — Publish or schedule
Use **`blotato_create_post`** (accountId + platform + text + platform fields from Step 0).

**Publish now (live, irreversible):** omit `scheduledTime`. Requires explicit user go-ahead
naming the post. It polls ~20s; if still processing, poll `blotato_get_post_status`. Report the
live link.

**Schedule (preferred default for batches):** set `scheduledTime` (ISO 8601, **UTC**).
Calendar-aware placement (profile §9):
1. Call **`blotato_list_schedules`** to see what's already booked (UTC, ascending).
2. Start after the last scheduled date, or fill open days, on the cadence
   **Mon/Wed/Fri @ 9:00 AM ET** (or whatever the user specifies).
3. **Convert ET → UTC** (EDT = +4h, EST = +5h). E.g. 9:00 AM EDT = 13:00 UTC.
4. If a target slot is already taken, bump to the next open cadence day and **tell the user**.
   Never double-book.
- To move/fix a scheduled post: `blotato_update_schedule`. To remove: `blotato_delete_schedule`.
  Both need the schedule `id` from `blotato_list_schedules`.

## Step 6 — Report
Summarize what went where: live links for published posts, and a dated list (in **ET**, with the
UTC stored value) for scheduled posts. Flag anything skipped (e.g. IG with no media) and any
compliance rewrites you made.

## Media handling
- **Public URL** → pass directly in `mediaUrls`.
- **Local file** → `blotato_create_presigned_upload_url` (get presignedUrl + publicUrl) →
  HTTP `PUT --data-binary "@file"` the raw bytes → use the returned `publicUrl` in `mediaUrls`.
- **Generate a visual** (quote graphic, carousel, AI video) → `blotato_create_visual`
  (browse `blotato_list_visual_templates` first), poll `blotato_get_visual_status` (≥15s between
  polls) until `done`, then use its `mediaUrl`/`imageUrls`.

## Safety (do not skip)
- **Draft → explicit approval → post.** Confirm before anything publishes or schedules.
- Treat **publish-now** as irreversible; require a clear, specific go-ahead.
- Enforce the brand-voice compliance gate on every caption; if a draft can't be made compliant,
  don't post it — say why.
- Confirm account IDs live each session; don't trust stale IDs.

## Token defaults (per the video's tips)
Routine extract/draft/schedule runs well on **Sonnet, low effort, extended thinking off**.
Reserve heavier settings for strategy work.

## Related
- **brand-voice** — the voice + compliance rules this skill applies to every caption.
- **brand/brand-profile.md** — the brand facts, accounts, cadence, and guardrails.
