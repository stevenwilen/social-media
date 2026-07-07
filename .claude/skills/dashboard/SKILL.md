---
name: dashboard
description: Keep the deployed content-board dashboard in sync with the real plan. Use whenever the plan changes — after autopilot drafts a week, after content-manager schedules posts, or when the user says "update the dashboard / update the board." Writes dashboard/plan.json (the single source of truth the site reads) and redeploys to Vercel so Steven and Veronica can see what's going out, when, and what still needs input.
---

# Dashboard — the shared content board

A view-only web dashboard (deployed on Vercel) that shows Veronica's content calendar, upcoming
posts with full captions, and — most importantly — a **"Needs from you"** panel. It's how Steven
(builder, doesn't know real estate) and Veronica (knows real estate) share one picture of the plan
without living inside Claude Code.

**It is a mirror, not a control panel.** Approvals and scheduling still happen in Claude Code
(autopilot / content-manager). The dashboard just *displays* the current plan.

## The data contract — `dashboard/plan.json`
The site reads this one file. Keep it valid JSON matching this shape:

```json
{
  "brand": { "name": "", "brokerage": "", "tagline": "", "phone": "", "website": "" },
  "generatedAt": "YYYY-MM-DD",
  "isSample": false,
  "window": { "label": "", "start": "YYYY-MM-DD", "end": "YYYY-MM-DD" },
  "cadence": "Mon / Wed / Fri · 9:00 AM ET",
  "posts": [
    {
      "id": "YYYY-MM-DD-<slug>",
      "date": "YYYY-MM-DD",
      "time": "9:00 AM ET",
      "platforms": ["facebook", "instagram", "youtube"],
      "pillar": "listings | market-update | development | community | lifestyle | video",
      "hook": "short title shown on the calendar",
      "status": "scheduled | proposed | needs-input | posted | draft",
      "media": { "type": "reel|image|video|none", "label": "human note about the media" },
      "captions": { "facebook": "text", "instagram": "text", "youtube": { "title": "", "description": "" } },
      "flags": ["VERIFY ... before posting"],
      "needs": "what's blocking (only when status is needs-input) — else null"
    }
  ]
}
```

Rules:
- **`status` drives everything.** `needs-input` posts surface in the "Needs from you" panel — always
  fill `needs` with a plain-English ask ("Drop the listing in the Drive folder", "Send the YouTube URL").
- `proposed` = drafted, awaiting Veronica's approval. `scheduled` = on the Blotato calendar.
  `posted` = live. Move a post's status forward as it progresses.
- Captions can be `null` when not written yet (blocked posts) — the site shows a placeholder.
- Set **`isSample: false`** on the first real plan (removes the "sample" banner).
- A caption/price/fact must never be invented — same compliance rules as everywhere.

## When to update it
- **After `autopilot`** produces a week's plan → write each slot as a post (mostly `proposed`, with
  `needs-input` for anything blocked). This is the main use.
- **After `content-manager` schedules** posts → flip those posts to `scheduled`.
- **After something posts** → flip to `posted`.
- Whenever Steven says "update the dashboard."

## How to update + redeploy
1. Edit `dashboard/plan.json` to reflect the current plan (validate it's still valid JSON).
2. Redeploy so the live site updates:
   ```
   vercel deploy --prod --yes
   ```
   (Run from the project root. Requires Steven to have run `vercel login` once — the deploy is
   otherwise non-interactive. If not logged in, tell him to run `vercel login`.)
3. Report the live URL and a one-line summary of what changed (e.g. "3 proposed, 1 needs a listing").

## Review layer (Approve / Request change) — the approval gate
Veronica reviews the plan **in the dashboard**. Each proposed post has **✓ Approve** and
**✏️ Request change** buttons; her choice saves to Supabase and shows live on the board.

- **Backend:** **Steven's own Supabase project** — URL `https://blqlgjpgdgbqrsqthbym.supabase.co`,
  table **`public.vw_post_reviews`** (columns: `post_id`, `status` ∈ `pending|approved|change-requested`,
  `note`, `plan_generated_at`, `reviewer`, `updated_at`). The dashboard reads/writes it directly with
  the public **publishable key** `sb_publishable_lLo3bBOWffNh6U48zewYTQ_jG8MhuLW` + RLS (safe: the key
  only grants access to this one table; it's already embedded in `dashboard/index.html`).
- ⚠️ **This project is NOT on the connected Supabase MCP** (that MCP is a different/agency account —
  do NOT use it for reviews). Read her decisions over REST instead, e.g.:
  ```
  curl -s "https://blqlgjpgdgbqrsqthbym.supabase.co/rest/v1/vw_post_reviews?select=post_id,status,note" \
    -H "apikey: sb_publishable_lLo3bBOWffNh6U48zewYTQ_jG8MhuLW" \
    -H "Authorization: Bearer sb_publishable_lLo3bBOWffNh6U48zewYTQ_jG8MhuLW"
  ```
  For schema changes (rare), Steven runs SQL in his Supabase SQL Editor — we don't have admin/MCP access.
- **Hard rule — only APPROVED posts get scheduled.** When the user says "schedule the approved posts":
  1. Query `vw_post_reviews`.
  2. Schedule (via content-manager) **only** posts whose `status = 'approved'`.
  3. For `change-requested`, read the `note`, redraft that post, update `plan.json`, redeploy — do
     NOT schedule it until she approves the new version.
  4. Leave `pending` posts alone (she hasn't reviewed them yet) and tell the user which are still pending.
- Post `id`s in `plan.json` must be **stable + unique** (date + slug) so her review sticks to the
  right post. When a new plan reuses an id, treat it as the same slot.

## Keep in sync with reality
Before flipping statuses to `scheduled`/`posted`, you can cross-check the real Blotato calendar with
`blotato_list_schedules` so the board matches what's actually booked. The board should never claim a
post is scheduled if Blotato doesn't have it.

## Notes
- Deploy scope is locked to `dashboard/**` via `vercel.json`; `.vercelignore` keeps secrets
  (`.mcp.json`, `.env`) off Vercel. Don't move plan.json out of `dashboard/`.
- The site is static and view-only by design (v1). Interactivity (approve/edit from the browser)
  is a future phase — don't add a backend without deciding that with Steven first.

## Related
- **autopilot** — produces the plan this board displays.
- **content-manager** — schedules the posts; flip them to `scheduled` here afterward.
- **listing-intake** — feeds listing posts; a `needs-input` listing slot points Veronica to the Drive folder.
