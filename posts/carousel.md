---
title: Coarousel
date: 2024-01-19
---

# Carousel Tab + Remake Text-Overlay — Implementation Plan

Two IG-native features for Creator Studio (`components/Creator.tsx` and children):

1. A **Carousel** tab, alongside the existing Remake and Shorts tabs.
2. A **suggested text overlay** button on generated videos in the **Remake** tab (top-of-video hook text, IG Reels style).

Grounded in the current code: `mainTab` in `Creator.tsx` / `CreatorDesktop.tsx` / `CreatorMobile.tsx` is currently `"text2video" | "shorts"` ("text2video" is labeled **Remake** in the UI). Generated videos are Cloudinary-hosted (`app/api/videoxters/status/[jobId]/route.ts` uploads provider output to Cloudinary), and the repo already has a precedent for computing derived-video URLs on the fly via string transformation (`app/api/videoxters/_services/cloudinary-transform.ts`, used today for watermark cropping). Both features reuse that pattern instead of introducing new render infra.

---

## Feature A: Carousel tab

### Goal
Turn a source video's key points into an IG carousel — a set of 1080×1350 slide images, each with a headline burned on top of a still — generated the same way Shorts/Remake already turn a video into repurposed content.

### User flow
1. User opens a Creator video, clicks the new **Carousel** tab.
2. Clicks **Generate Carousel**. Backend breaks the transcript into N key points (reusing the existing scene-analysis approach from `/api/creator/analyze`), picks/creates a background image per slide, and writes headline text on each.
3. User sees a grid of slides, can regenerate an individual slide's text, reorder, and download the set (ZIP) or (once available) publish to Instagram via `ConnectorsTab`.

### Open decisions (need your call before implementation)
- **Background source**: real frame grabs from the video vs. AI-generated illustrative backgrounds vs. a plain branded background (fastest to ship, matches `reports/brand.yaml`'s voice/visual restraint).
- **Slide count**: fixed (e.g. always 6) or variable based on transcript length.
- **Publish path**: Instagram publishing isn't live yet (`ConnectorsTab.tsx` lists Instagram as "Coming soon") — v1 should ship as **download-only** (ZIP export), with IG carousel publish wired in once that connector exists.

### Rollout
- Phase 1: generate + preview + download ZIP.
- Phase 2: per-slide regenerate/reorder/edit text inline.
- Phase 3: publish via Instagram connector once it ships.

---

## Feature B: Suggested text overlay on Remake videos

### Goal
After a Remake ("text2video") scene finishes generating, let the user add IG-style bold hook text burned onto the top of the video (e.g. "POV: ...", "3 things nobody tells you about X") without leaving the tab.

### User flow
1. In `Text2VideoTab.tsx`'s job grid (the `jobs.map(...)` block, ~line 410), once a job is `status === "completed"`, show a new **Add Text** button on that card.
2. Clicking it calls a suggestion endpoint seeded from that job's `scriptText` (+ the parent video's title/transcript for context) and returns 2–3 short suggested overlay lines.
3. User picks one (or types their own), sees a live preview of the text burned onto the video via a Cloudinary transform (instant — no re-render), and clicks **Apply**.
4. The chosen text is saved on the job; the video card, downloads, and any future publish flow use the overlay URL instead of the raw one from then on.

### UI changes
- `Text2VideoTab.tsx`: add an "Add Text" affordance per completed job card, a small popover/inline panel showing the 2–3 suggestions + a free-text field + Apply button, and swap the `<video src={job.videoUrl}>` for the overlay URL when `overlayText` is set.

### Data model
- Add `overlayText String?` (and optionally `overlayStyle Json?` for position/color if we want more than one style later) to `VideoxtersJob` in `prisma/schema.prisma`. No new table — this mirrors how cropping is handled today (computed URL, not a re-uploaded asset).

### Backend
- `app/api/creator/video-text-suggestions/route.ts` (new): `{ jobId }` → OpenAI structured output (same `captions/route.ts` pattern) returning `{ suggestions: string[] }`, seeded with `job.scriptText` and the parent `creatorVideo.title`/transcript excerpt.
- `app/api/videoxters/apply-text/route.ts` (new, or extend the existing `status/[jobId]` route): `{ jobId, text }` → validates ownership, writes `overlayText` on the `VideoxtersJob` row.
- New helper alongside `cloudinary-transform.ts`, e.g. `getTextOverlayVideoUrl(videoUrl, text)`, building a Cloudinary `l_text:Arial_60_bold:<url-encoded text>` layer positioned top-center (`g_north`) with a readability treatment (white text + subtle dark backing/stroke) — computed on read, same as `getCroppedVideoUrl`, so no extra storage or render step.

### Downstream wiring
- `ConnectorsTab`/`YouTubeConnectorCard` currently receive `publishableJob = { jobId, videoUrl }` from `CreatorDesktop.tsx` (latest completed `VideoxtersJob`, not scoped to Remake specifically — worth confirming that's the intended target before wiring overlay text into publishing). Once overlay text is applied, that `videoUrl` should resolve through `getTextOverlayVideoUrl` too, so what gets published matches the preview.

### Rollout
- Phase 1: suggest + preview + apply (this plan's scope).
- Phase 2: font/position/color controls if the fixed IG-meme-text style isn't enough.

---

## Shared groundwork
Both features lean on the same trick — Cloudinary delivery-URL transforms for compositing text/frames without a render pipeline — so it's worth extracting a small shared `lib/cloudinaryOverlay.ts` (text-layer string builder) that both `getTextOverlayVideoUrl` (Feature B) and the carousel slide renderer (Feature A) call into, rather than duplicating the transform-string logic.

## Sequencing recommendation
Ship Feature B first — it's the smaller change (one new column, two small routes, one UI affordance on an existing tab) and validates the Cloudinary-text-overlay approach before Feature A builds a whole new tab and generation pipeline on top of the same technique.
