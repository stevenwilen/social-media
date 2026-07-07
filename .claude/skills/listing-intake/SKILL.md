---
name: listing-intake
description: Turn a listing that Veronica (or Steven) drops into the shared Google Drive folder — photos + a short note — into an accurate, on-brand listing post with a branded photo reel. Use whenever a post needs real listing data (address, price, beds/baths, status, photos) for a property she represents, or when the user says "post my new listing / price drop / open house / just sold." Replaces the old FlexMLS integration: no API, no re-typing — she just drops files in a folder.
---

# Listing Intake — Google Drive folder → listing post

## What this unlocks
Veronica's ONLY job for a listing post is to drop material into one shared Drive folder. This skill
reads it, builds a **branded photo reel from her real photos**, and hands it to brand-voice +
content-manager to caption and schedule. No FlexMLS, no API token, no hand-entering data into Claude.

## The intake folder (source of truth)
- **Folder name:** `Steven – Social Media Listings` (note the en-dash)
- **Drive folder ID:** `1Y3iXRf26PWAWQyGIQLJbU2LW5CH7cj58`
- **Owned by** `steven.wilen@gmail.com`, **shared (Editor) to the connected account** `ilanfridman23@gmail.com`.
- The claude.ai Google Drive connector here is authorized as **ilanfridman23** (agency account), so
  we read the folder through the share — not through Steven's account directly. This is expected.

Each listing lives as its own **subfolder** (or a loose batch of files if there's only one). Inside:
- **Photos** — the actual property photos (`image/*`). These become the reel.
- **A note** — a Google Doc or `.txt` with the listing details (template below).
- **(Optional) a ready-made flyer** — if she made one in Canva she loves, it's just an image; post it
  as-is instead of generating a reel.

## The note template (what she writes)
Keep it dead simple. Any of these fields she leaves blank, we work around or flag:

```
ADDRESS: 123 Marsh Landing Pkwy, Ponte Vedra Beach, FL 32082
PRICE: $875,000
BEDS / BATHS: 4 bed / 3 bath
SQFT: 2,850
STATUS: Just Listed        (Just Listed / Price Drop / Open House / Pending / Just Sold / Coming Soon)
HIGHLIGHTS: Renovated chef's kitchen; backs to preserve; gated Nocatee community; steps to Splash Park
OPEN HOUSE: Sat 6/12, 1–4pm   (only if STATUS is Open House)
NOTES: anything else she wants mentioned or avoided
```

> Want to hand her a physical copy of this? Offer to drop a `LISTING TEMPLATE.txt` into the folder so
> she can duplicate it per listing.

## Step-by-step

### Step 1 — Find the listing to work
- List folder contents: `search_files` with `parentId = '1Y3iXRf26PWAWQyGIQLJbU2LW5CH7cj58'`.
- If subfolders exist, each is one listing — pick the one the user named, or the newest
  (`createdTime` desc) if they just said "post my new listing."
- List that subfolder's files the same way (`parentId = <subfolder id>`).

### Step 2 — Read the note
- **Google Doc** → `read_file_content` (or `download_file_content`) on the doc ID.
- **`.txt`** → `download_file_content` on the file ID.
- Parse the fields above. If the note is freeform (she didn't use the template), read it and extract
  address / price / beds / baths / status / highlights yourself — be forgiving.
- **Never invent or alter a number.** If PRICE or an address is missing, flag it and stop before
  posting — don't guess. A wrong price on a listing is a serious error.

### Step 3 — Get the photos into Blotato (the media pipeline)
Drive files are **not public URLs**, and Blotato needs public URLs. For each photo (cap at ~6–10, best
first — exterior/hero shot first):
1. `download_file_content` on the Drive image ID → raw bytes.
2. `blotato_create_presigned_upload_url` → returns `presignedUrl` + `publicUrl`.
3. HTTP `PUT` the raw bytes to `presignedUrl` (`--data-binary`, correct content-type).
4. Keep the returned `publicUrl`.

### Step 4 — Build the branded reel
Use `blotato_create_visual` with the **"Image Slideshow with Text Overlays"** template
(`/base/v2/image-slideshow/5903b592-1255-43b4-b9ac-f8ed7cbf6a5f/v1`):
- One `slide` per photo, `imageSource` = the uploaded `publicUrl`.
- `textOverlay` per slide, driven by STATUS:
  - Slide 1 (hero): the status banner + price — e.g. **"JUST LISTED · $875,000"**.
  - Slide 2: **"4 BR · 3 BA · 2,850 SQFT"**.
  - Slide 3: address — **"123 Marsh Landing Pkwy · Ponte Vedra Beach"**.
  - Remaining slides: one highlight each ("Renovated chef's kitchen", "Backs to preserve"…).
  - Open House → make slide 1 **"OPEN HOUSE · Sat 1–4PM"**.
- `aspectRatio` **4:5** (feed) or **9:16** (reel/story) — 4:5 is the safe default for FB+IG.
- Poll `blotato_get_visual_status` (≥15s between polls) until done → use its `mediaUrl`.
- **Escape hatch:** if the folder already has a finished flyer/graphic she made, skip generation and
  use that image directly as `mediaUrls`.

### Step 5 — Hand off to captioning
Pass the parsed listing facts to **content-manager** / **brand-voice**:
- Draft platform-tailored captions (FB + IG always; the reel satisfies IG's media requirement).
- Lead with the status hook; weave in 1–2 highlights and her Nocatee authority.
- Run the **compliance gate** (below). Then Review → approve → schedule, exactly like any other post.

## Compliance (real-estate-specific — non-negotiable)
- **Her own listings only.** Don't repost other brokers' listings publicly.
- **Fair Housing:** describe the PROPERTY, never the ideal buyer (profile §10).
- **FL disclosure:** include brokerage attribution where required.
- **"Coming Soon" rules:** respect MLS status/attribution/photo-use rules.
- **Never alter price/facts** — use them exactly as she wrote them. If a figure looks off, ask her to
  reconfirm status/price before posting rather than guessing.

## Status
**LIVE.** Drive folder connected and read access verified (2026-07-07). No token needed. Drop a
listing in the folder and run this skill (or autopilot's listings pillar) to produce a post.

## Related
- **content-manager** — captions, scheduling, publishing (this skill feeds it the listing + reel).
- **brand-voice** — voice + Fair Housing compliance gate applied to every caption.
- **autopilot** — its LISTINGS pillar calls this skill when a slot is a listing post.
- **brand/brand-profile.md** — brand facts, accounts, cadence, guardrails.
