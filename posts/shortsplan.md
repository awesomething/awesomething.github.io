# Real Shorts Clip Generation Pipeline

## Verification update (all 3 gates passed with real API calls)

All three unverified assumptions below were hand-tested against the real services and passed —
with one real architecture correction found along the way:

- **yt-dlp resolves a real, fetchable direct URL** — confirmed (`200`, `video/mp4`, correct size).
- **Upload-Post's ffmpeg endpoint works** — confirmed, but **the yt-dlp direct URL cannot be
  passed to Upload-Post directly.** Googlevideo direct URLs are IP-locked to whoever resolved
  them; Upload-Post's server (a different IP) gets a `403` trying to fetch it. **Fix, now part of
  the architecture**: after `resolveDirectVideoUrl`, download the bytes ourselves and upload to
  Cloudinary (`resource_type: "video"`, same pattern used everywhere else in this app), then hand
  Upload-Post *that* stable Cloudinary URL — never the raw googlevideo one.
- **Dual video+SRT input burns real captions** — confirmed by downloading the output and pulling
  an actual frame: burned-in text is visibly present, not just a `finished` status.
- **Correction to the documented contract**: real status values are lowercase `queued` /
  `finished` / `failed`, not `PENDING`/`FINISHED`/`ERROR` as the docs implied. `full_command` must
  literally start with the word `ffmpeg` (a bare arg string like `-ss 0 -to 5 -i {input} {output}`
  is rejected with `400`).

This means `resolveDirectVideoUrl` (Section: New/changed files) is no longer just a yt-dlp
wrapper — it's a 3-step staging pipeline: **resolve → download → upload to Cloudinary → return the
Cloudinary URL**. The rest of the architecture below is unchanged.

## Context

The previous pass built a "Shorts Automation" settings wizard (`ShortsAutomationWizard.tsx`,
`ShortsTab.tsx`, `ShortsAutomation` Prisma model) for `/dashboard/creator/[id]`'s Shorts tab —
deliberately scoped to settings + UI only, no real generation. The user noticed the template
picker shows dummy CSS gradients instead of real video and asked for a plan to build the actual
pipeline: moment detection, real clip cutting, template-styled burned-in captions, real
thumbnails. Recurring auto-post scheduling is explicitly **out of scope** for this pass (user's
choice) — `autoPostEnabled`/`scheduleSlots` stay inert, saved but unused.

This plan reuses proven infrastructure already in the codebase rather than inventing new
patterns: `VideoxtersJob`'s submit→poll→finalize shape, `checkCredit`/`deductCredit`, Cloudinary
upload, and the OpenAI structured-output scene pipeline already built for Text2Video's Analyze
step. It also surfaces two real, previously-unverified assumptions (yt-dlp actually resolving a
direct video URL from a hosted environment, and Upload-Post's ffmpeg service actually accepting a
second SRT input) that must be hand-verified before any code is built on top of them — this
codebase's own established practice (see `plans/GENERATE.md`'s Council Verdict).

## Architecture

Mirror `app/api/videoxters/generate` + `app/api/videoxters/status/[jobId]` (submit → poll →
finalize), storing results on the existing `CreatorJob` model (`type: "shorts"`, `clips: Json?`
already shaped for this: `[{url, start, end, thumbnail}]`) — not a new table.

```
"Generate Shorts Now" (ShortsTab.tsx)
  → POST /api/creator/shorts/generate
      - loads ShortsAutomation settings server-side (not from client body)
      - checkCredit()
      - getOrComputeScenes(creatorVideoId)      [reuses/extracts analyze's pipeline]
      - resolveDirectVideoUrl(videoId)          [wires up dead yt-dlp scaffolding]
      - pickMoments(scenes, settings)           [2nd OpenAI pass → clip windows]
      - per clip: buildSrtForRange + submitFfmpegJob(video, srt, template filter)
      - CreatorJob.pendingClips = [{start,end,uploadPostJobId,srtUrl,status}]
      - return { jobId } immediately
  → client polls GET /api/creator/shorts/status/[jobId] every 4s
      - checks each pending clip's Upload-Post status
      - finalizes ONE newly-FINISHED clip per poll (download → Cloudinary → move to `clips`)
      - once all done: atomic-claim the job, deductCredit() once, status: completed/failed
```

## Build order (riskiest/unverified first)

1. **Verify yt-dlp actually resolves a direct URL** — hand-run `ytDlpWrap.execPromise([url, "-g",
   "-f", "best[ext=mp4]/best"])` (existing `ytDlpWrap`/`ensureYTDlp` from
   `app/api/youtube-chapters/lib/config.ts` — confirmed via repo-wide grep to be dead code today,
   never actually called). This is the single biggest risk: hosted-IP YouTube bot-detection is a
   common real-world failure for yt-dlp, and `SERVER_COOKIES_PATH` in that same file is an unused
   empty path. If this fails, the whole pipeline needs a different source-resolution strategy
   before anything else is built.
2. **Verify Upload-Post's ffmpeg endpoint is real and reachable** with the actual
   `UPLOAD_POST_KEY` — one real submit → poll → download round trip using the direct URL from
   step 1, via a small standalone `lib/shorts/uploadPostClient.ts`.
3. **Verify the two-input (video + SRT) command actually works** — same real test, now with a
   second `files[]` entry and `subtitles={input1}` in `full_command`. This is unproven in the
   documented contract and is the biggest single assumption in the plan; if it fails, captions
   need a different approach (e.g., pre-baked into the video before submission).
4. Only after 1–3 pass: build the pure/testable pieces — `getOrComputeScenes` (extracted from
   `analyze/route.ts`), `pickMoments.ts`, `buildSrtForRange.ts`, `templateStyles.ts`.
5. Then the two routes, then the `pendingClips` schema field, then `ShortsTab.tsx` wiring, then
   `vercel.json` maxDuration entries.
6. Ship with one template (`hormozi1`) fully working end-to-end before tuning the other two
   caption styles.

## New/changed files

- **`prisma/schema.prisma`** — add `pendingClips Json?` to `CreatorJob` (in-flight per-clip
  tracking: `[{start, end, uploadPostJobId, srtUrl, status}]`). `clips` stays exactly as already
  designed for the final array. Push via `prisma db push` (this repo's migration history is
  already broken for `migrate dev` — same workaround used earlier this session).
- **`lib/creator/computeScenes.ts`** *(new — extracted, not duplicated)* — pulls `buildWindows` +
  the OpenAI scene-parse call out of `app/api/creator/analyze/route.ts` into
  `getOrComputeScenes(creatorVideoId): Promise<Scene[]>` (cache-or-compute, same as today).
  `analyze/route.ts` becomes a thin auth+ownership wrapper calling it. Needed because a user can
  click "Generate Shorts Now" without ever visiting the Text2Video Series tab.
- **`lib/shorts/pickMoments.ts`** *(new)* — second OpenAI pass (same `gpt-4o` +
  `zodResponseFormat` pattern as `analyze/route.ts`) over the already-computed `scenes`: picks up
  to 3 non-overlapping contiguous scene ranges matching the `clipLength` bucket, scores each 0-100
  on standalone virality, server-side filters below `viralityThreshold` (don't trust the model to
  self-filter).
- **`lib/shorts/buildSrt.ts`** *(new)* — builds an SRT from `CreatorVideo.timestampedTranscript`
  sliced to one clip's range. Two correctness details that matter: (1) timestamps must be
  **clip-relative** (`original - clipStart`, clamped), since `-ss/-to` before `-i` resets ffmpeg's
  output PTS near zero; (2) `wordsPerCaption` grouping needs per-word timestamps interpolated
  linearly across each transcript segment's `duration` (the transcript only has phrase-level
  timing, not real word-level ASR) — note this as an approximation in a code comment, not implied
  precision.
- **`lib/shorts/templateStyles.ts`** *(new)* — maps `template`/`layout` to an ffmpeg crop/scale
  filter + a libass `force_style` string (font/size/color/position) per template. **Scope
  correction to be explicit about**: true "Hormozi-style" word-by-word progressive highlight
  needs `.ass` + `\k` karaoke tags, a materially bigger and separately-unverified lift. v1 ships
  visually-distinct-but-uniform-style captions per template; per-word animation is a known,
  explicitly-flagged gap, not a silent shortcut.
- **`lib/shorts/uploadPostClient.ts`** *(new)* — thin wrapper: `submitFfmpegJob`,
  `getFfmpegJobStatus`, `downloadFfmpegResult` against Upload-Post's REST API (`Authorization:
  Apikey ${UPLOAD_POST_KEY}`). Verified real status values: `"queued"` → `"finished"` / `"failed"`
  (lowercase — not the `PENDING`/`FINISHED`/`ERROR` the docs implied). `full_command` must start
  with the literal word `ffmpeg` (confirmed via a real `400` when it didn't). Dual-input syntax
  confirmed working exactly as documented: `files: [videoUrl, srtUrl]` + `-i {input0} -vf
  "subtitles={input1}" {output}`.
- **`lib/creator/resolveDirectVideoUrl.ts`** *(new)* — wires up the currently-dead
  `ytDlpWrap`/`ensureYTDlp` scaffolding, but per the verified correction above this is now a
  3-step staging function, not a plain URL resolver: (1) `ytDlpWrap.execPromise([url, "-g", "-f",
  "best[ext=mp4]/best"])` to get the IP-locked direct URL, (2) fetch those bytes server-side, (3)
  upload to Cloudinary (`resource_type: "video"`, folder e.g. `"shorts_source"`) and return the
  `secure_url`. Only the Cloudinary URL is ever handed to Upload-Post.
- **`app/api/creator/shorts/generate/route.ts`** *(new)* — orchestrates the steps in the
  Architecture diagram above. Loads `ShortsAutomation` **server-side by `creatorVideoId`**, never
  trusting client-supplied settings — this is what makes "use currently-saved settings" a real
  guarantee. Fails the whole batch immediately if `resolveDirectVideoUrl` throws (don't create an
  unfinishable job).
- **`app/api/creator/shorts/status/[jobId]/route.ts`** *(new)* — polls pending clips, finalizes
  **at most one newly-completed clip per request** (mitigates Vercel `maxDuration` risk under a
  multi-clip batch), moves finished entries from `pendingClips` into `clips`. `CreatorJob` has no
  direct `userId` — scope the lookup through the relation:
  `prisma.creatorJob.findFirst({ where: { id: jobId, creatorVideo: { userId: user.id } } })`,
  same join-through-relation pattern already used in `shorts-automation/route.ts`. Uses an atomic
  `updateMany` claim (`where: { status: { notIn: ["completed","failed"] } }`) before calling
  `deductCredit()` so two concurrent polls can't double-deduct — a small, deliberate hardening
  over the `videoxters/status` pattern it otherwise copies, since a multi-clip batch's completion
  is reached via repeated polls rather than a single check.
- **Thumbnails**: no new mechanism — same technique already used in this codebase, just without
  the watermark-crop step (that's provider-specific to heygen/d-id, not relevant here): swap the
  Cloudinary video URL's extension to `.jpg` and Cloudinary renders a frame on first request.
- **`components/ShortsTab.tsx`** *(changed)* — add a **"Generate Shorts Now"** button in the
  existing automation-summary view (separate from the wizard's Confirm, which keeps only saving
  settings as it does today). Mirrors `Text2VideoTab.tsx`'s exact resume-polling-on-mount +
  job-grid pattern almost line for line: `useState` seeded from `creatorVideo.jobs.filter(j =>
  j.type === "shorts")`, a `pollRefs` ref, `useEffect` resuming polls for any non-terminal job on
  mount, optimistic job append on submit. Renders a 3-up clip grid per job once `clips` populate
  (`<video>` + start/end badge), spinner while `pendingClips` remain.
- **`vercel.json`** — add both new routes at `maxDuration: 120`, matching the existing
  `videoxters/generate`/`videoxters/status/[jobId]` entries (same category of work).
- **`ShortsAutomationWizard.tsx`** *(small copy fix)* — the Confirm-step "Credit cost" row
  currently says "1 credit per 20 secs of video length," which was never wired to real logic.
  Recommend flat **1 credit per Shorts generation batch** (matches the `videoxters` precedent,
  simplest to reason about, matches the atomic-claim guard above) — update the copy to match once
  this ships. Flagging this as the one product decision in the plan that isn't purely technical;
  easy to change later since it's just a string + which line in `status/[jobId]` calls
  `deductCredit()`.

## Known risks (surfaced now, not silently assumed away)

- ~~yt-dlp on a shared hosted IP~~ — **verified working** (see Verification update above). Residual
  risk: only tested from this session's shell, not Vercel's actual `iad1` deploy region, and only
  against one video (a real N=1 caveat the council raised — bot-detection is "common," not
  universal, so a production run against age-restricted/region-locked/live videos hasn't been
  tested).
- ~~Upload-Post's two-file-input contract~~ — **verified working**, with the IP-lock correction
  above baked into the architecture.
- Vercel `maxDuration` under multi-clip finalization — mitigated by finalizing one clip per poll,
  but worth confirming with a real multi-clip run, not just assumed safe by design.
- Upload-Post quota (free tier ~30 min/month per prior research) may not cover more than a
  handful of generations — worth a real look at the current plan before assuming headroom.
- Copyright/ToS exposure for cutting derivative clips of YouTube videos — same open question this
  repo already carries for Text2Video/Lyrics Video (`GENERATE.md`), not new to this feature, not
  an engineering problem to resolve here.

## Verification plan

1. ~~Manual, real-call tests for yt-dlp resolution and the Upload-Post single/dual-input ffmpeg
   contract~~ — **done**, see Verification update above (real API calls, real downloaded output,
   a real burned-in-caption frame visually confirmed).
2. `npx tsc --noEmit` after implementation — confirm no new errors touch changed/added files
   (established practice this session; pre-existing repo errors are expected and unrelated).
3. End-to-end real run against the same test `CreatorVideo` used earlier this session
   (`cmrb2kajs001t7kww92g162c5`, "Michael Jackson - Scream (Dance Breakdown 4k)"): trigger
   generate, poll to completion, confirm real Cloudinary URLs + thumbnails, confirm
   `CreatorJob.clips` populated correctly, confirm credit deducted exactly once (not twice under
   concurrent polls).
4. Browser click-through remains blocked by Clerk's dev-browser session requirement (same known
   limitation as the rest of this session) — verify via direct API/DB calls, not a live UI
   click-through, and say so explicitly rather than overclaiming.
