---
name: autopilot
description: Veronica Wilen's autonomous social-media manager. Use when the user wants the agent to DECIDE what to post — not just execute. Run it (e.g. "run autopilot", "plan my content", "fill next week") and it audits the calendar, decides the content mix, sources the material itself, drafts everything on-brand, and hands back ONE finished plan to review. It never publishes or schedules until she approves. This is the decision layer on top of content-manager + brand-voice.
---

# Autopilot — the manager, not the operator

Veronica's job with this skill is **editor-in-chief**: she runs it, reviews one finished plan,
approves (or tweaks), and walks away. The agent makes every routine decision *for* her.

## The autonomy contract (read first)
- **Autonomous in decisions:** what to post, which pillar, what angle, when, and how to source it.
- **NEVER autonomous in publishing.** Nothing is scheduled or posted until she approves the plan.
  Her one job is to review before it goes out — protect that gate absolutely.
- **Never fabricate facts.** No invented listings, prices, addresses, or precise market stats.
  If a slot needs data only she has, propose the slot and clearly flag what you need.
- **Compliance always on** (brand-voice gate: Fair Housing, FL disclosure, no overpromising).

## Step 0 — Load context
Read `brand/brand-profile.md`; apply **brand-voice** (voice + compliance) and **content-manager**
(execution + scheduling). Confirm live accounts with `blotato_list_accounts`.

## Step 1 — Audit the current state
- **What's booked:** `blotato_list_schedules` → find the last scheduled date and open cadence days.
- **What ran recently:** `blotato_list_posts` (since ~30 days) → note recent pillars, topics, and
  hooks so you **don't repeat yourself**. Variety is a quality bar, not a nice-to-have.

## Step 2 — Decide the plan window
Default: **the next open week of cadence slots** (Mon/Wed/Fri @ 9:00 AM ET, per profile §9),
starting after the last already-scheduled date. Honor any window the user names ("next 2 weeks",
"just Monday").

## Step 3 — Make the editorial decisions
Assign a **content pillar** to each slot (profile §4) so the batch:
- hits roughly **70% value / 30% listings**,
- **leads with her Nocatee authority** (her strongest differentiator),
- **rotates pillars** and avoids topics used recently (Step 1),
- feels varied across the week (don't stack two market-stat posts back to back).

## Step 4 — Source each slot autonomously
Pick the source per pillar. Prefer pillars you can source *without* her input:

| Pillar | How autopilot sources it | Needs her input? |
|---|---|---|
| Lifestyle / area spotlight | Write from knowledge (Nocatee/Ponte Vedra/St. Augustine amenities, neighborhoods) + light verify | No |
| City/county development | `perplexity-query` or WebSearch recent St. Johns/Nocatee growth news; summarize + link | No (verify facts) |
| Community news/events | Research upcoming First Coast events; summarize w/ dates | No (verify dates) |
| Market reports/stats | Research current St. Johns/Nocatee stats; **mark every number "VERIFY"** + cite source | Better if she supplies the monthly report |
| Video repurpose | Use a YouTube URL she's given or a recent upload (Blotato scrapes transcript) | A URL, if repurposing |
| Listings / new-to-market | Use the **`listing-intake`** skill — read the shared Drive folder (photos + note), build a branded photo reel, use it as IG media | Only that she's dropped the listing in the folder; if the folder's empty, flag "needs a listing" |

Research rule: anything time-sensitive or numeric gets **verified and cited**, and market figures
are labeled "VERIFY before posting." Never state a stat you can't source.

## Step 5 — Draft everything (via content-manager + brand-voice)
- Targets are **Facebook + Instagram** only (we do NOT auto-publish to YouTube — she runs her own
  channel; a YouTube URL is only ever a *source* to repurpose into FB/IG posts).
- **Instagram needs media.** Decide per slot: if there's a video, use it; otherwise **propose a
  generated visual** (`blotato_create_visual` — a market-stat card, tip carousel, quote card, or
  neighborhood spotlight; browse `blotato_list_visual_templates`). If neither fits, flag "IG needs
  media" rather than dropping it silently.
- Run every caption through the compliance gate. Rotate CTAs + hashtag sets.

## Step 6 — Present ONE plan to review
Lead with anything blocking, then the plan, then the drafts:

1. **"Needs from you"** — a short list (e.g. "Wed listing post: send the MLS link", "confirm the
   May median-price figure"). If nothing's needed, say "Nothing needed — approve to schedule."
2. **Plan table** — `Date (ET) | Platform(s) | Pillar | Hook | Media | Status`.
3. **Full drafts** — grouped by date, each caption ready to read, flagged with any "VERIFY" items.

## Step 7 — Review loop
Support: approve all · approve a subset ("all except Wednesday") · edit a draft · swap a topic ·
change a time. Make changes and re-show only what changed. Default to *nothing happens* until she says go.

## Step 8 — Schedule the approved batch
Approval now happens **in the dashboard** (Veronica clicks Approve / Request change per post; saved to
Supabase `vw_post_reviews`). When the user says "schedule the approved posts," read those decisions
(see the **dashboard** skill) and hand **only `approved`** items to **content-manager** to schedule
(calendar-aware, ET→UTC, no double-booking). Redraft `change-requested` posts from her note; leave
`pending` ones. Her in-dashboard approval (or a direct go-ahead from Steven) is the required gate —
never schedule an unreviewed post.
Then report: what's scheduled (dated, ET), what's still waiting on her input, and any compliance
rewrites you made. Optionally save the approved plan to `content-plans/YYYY-MM-DD.md` for a record.

## Step 9 — Update the shared dashboard
Sync the plan to the board so Steven + Veronica can see it outside Claude Code: use the **dashboard**
skill to write each slot into `dashboard/plan.json` (proposed slots as `proposed`, blocked slots as
`needs-input` with a plain-English `needs`, scheduled ones as `scheduled`) and redeploy to Vercel.
Do this even before approval — the board is how she reviews "what's coming and what I owe you."

## Optional — full hands-off mode
This skill can be wrapped in a **scheduled routine** (cron) so it runs itself — e.g. every Monday
8am it drafts the week and pings her to review. She still approves before anything posts. Offer
this once she trusts the plans; set it up with the `schedule` skill.

## Decision principles (when unsure)
- Favor content you can source solo over content that needs her — minimize her input.
- When a fact is shaky, either verify it or don't use it. "VERIFY" beats wrong.
- A lighter, varied, value-forward week beats a heavy salesy one.
- One clean plan with decisions already made > a pile of options. She edits; she doesn't assemble.

## Token defaults
Runs well on **Sonnet, low effort, extended thinking off** (per the video's tips). Bump effort
only for genuinely strategic planning.
