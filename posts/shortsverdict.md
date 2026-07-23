# Council Verdict: Real Shorts Clip Generation Pipeline

Run against `plans/shortsplan.md` via the 5-advisor LLM Council (Contrarian, First Principles
Thinker, Expansionist, Outsider, Executor — independently analyzing, then peer-reviewing each
other anonymously, then synthesized by a chairman). Question posed: is the plan sound as written,
and should it be built as specced?

## Where the Council Agrees

Every advisor converges on the same structural complaint: **the plan is verification-first in
name only.** Steps 1-3 are labeled as gates, but steps 4-6 (pickMoments, templateStyles, the full
Prisma/polling/finalization architecture) are already fully specced as if they'll pass. The plan
preaches "riskiest first" and then doesn't act like it.

Unanimous agreement on two blocking numbers, not cosmetic ones:
- **Upload-Post's free-tier quota (~30 min/month)** is named then dropped. Must be resolved
  *before* step 1 — could make the feature economically nonviable regardless of whether the
  technical assumptions hold.
- **Partial-failure / kill-condition handling is absent.** What happens when yt-dlp gets
  bot-detected, or 2 of 3 clips finish and one fails permanently — no fallback is written down.

Convergence on execution mechanics: verification needs to be three separate gated tests (yt-dlp
resolution, single-input Upload-Post, dual-input SRT burn), each with a pass/fail line, run
against real hosted infrastructure, with actual response payloads captured — not "looks like it
should work per docs."

## Where the Council Clashes

**Verify-then-build vs. verify-then-productize.** The Expansionist argues the plan undersells
itself — pickMoments, resolveDirectVideoUrl, and templateStyles are reusable infrastructure worth
scoping as first-class services once verification passes. Four of the five peer reviews
independently flagged this as the council's biggest blind spot: it prices upside on components
that have never survived a single real HTTP call. Not a close call — the disagreement is genuine,
but it's answering a question nobody should be asking yet.

**Is this an engineering plan or a business decision in disguise?** The Contrarian frames
copyright/ToS exposure as a landmine specific to this feature's core function (auto-clipping and
reposting YouTube content). The Expansionist inverts it: if repost rights are actually cleared,
that's not a risk, that's the product. Neither can be resolved by engineering judgment — this is
a legal/business question the plan is silently punting.

**Does the plan even need yt-dlp?** First Principles alone attacked the premise: the Analyze
pipeline that already produced transcripts and scene timestamps had to get *something* from
somewhere — if a video asset were already stored, resolveDirectVideoUrl's bot-detection risk
might not need to exist at all. **Resolved same-day, post-verdict**: confirmed by direct schema
and codebase check — `CreatorVideo` has no stored video-file reference anywhere; the only
`videoUrl` field in the schema belongs to `VideoxtersJob` (AI-generated *output*, not source), and
no code path in this app has ever downloaded the raw YouTube video. Playback is a client-side
YouTube iframe embed only. yt-dlp (or an equivalent resolution mechanism) is genuinely necessary
— this was a great question to ask, but it does not dissolve the plan.

## Blind Spots the Council Caught

- **N=1 verification is insufficient.** The plan itself calls bot-detection "common." One green
  yt-dlp call and one green dual-SRT call don't rule out intermittent failure across video types
  (age-restricted, region-locked, live streams).
- **Burned-in captions are irreversible.** A transcript timing/accuracy error ships permanently
  baked into the video pixels, with no post-hoc fix path.
- **Vendor risk beyond cost**: polling latency/UX (do users wait minutes for a clip?), vendor
  lock-in/exit cost, data-privacy exposure of piping video+transcripts through a third-party SaaS.
  Also: free verification nobody's used yet — check Upload-Post's own support forum/changelog for
  whether others have hit the dual-SRT case before spending engineering time reproducing it.
- **The spike itself is underspecified as an experiment.** Who runs it, on what infra (same
  cloud/region as prod — that's what determines bot-detection risk), what "pass" means
  numerically (is 2 of 3 dual-SRT attempts a pass or a production retry-loop risk), and a hard
  test-budget number so the spike doesn't itself burn the monthly quota it's meant to protect.

## The Recommendation

Do not build this as specced. The plan's own structure — six build-order steps, two explicitly
unverified load-bearing assumptions, both flagged as commonly-failing in its own risk section —
describes an experiment wearing an implementation plan's clothes. The council is not divided on
this; the only advisor arguing for more ambition (Expansionist) is arguing for *more* speculative
build, not for building the current plan as-is, and four separate reviews flagged that position as
the least grounded in the room.

The right shape: treat steps 1-3 as a scoped, time-boxed spike with hard kill conditions, and treat
steps 4-6 as not-yet-real until that spike produces evidence, not documentation-reading.
Concretely — three gated tickets (yt-dlp resolution, single-input Upload-Post, dual-input SRT
burn), each executed against real hosted infrastructure with pasted response payloads, each with a
numeric pass bar (not "it worked once"), and a hard rule: no PR, schema change, or component gets
written until all three are green. Resolve the quota number and the copyright/ToS question as
inputs to whether this ships at all, not as risks filed for later. Nothing about reusable
infrastructure, premium templates, or productizing `pickMoments` gets scoped until this evidence
exists — that's next quarter's conversation, not this one's.

## The One Thing to Do First

Run the three verification gates as real, hand-tested API calls — from infrastructure matching
where this would actually deploy — before writing any pipeline code:
1. `yt-dlp -g` against a real hosted YouTube URL → confirm a direct, fetchable media URL comes
   back (this is now the *first* open question, since the "maybe it's redundant" check above is
   closed — there is no existing stored video asset to fall back on).
2. One real Upload-Post submit → poll → download with a single video input.
3. Immediate retest with a second `files[]` SRT entry + `subtitles={input1}` in `full_command`.

Each gate needs a numeric pass bar (not "worked once") and a hard stop if it fails — no schema
change, no new file, no component wiring starts until all three are green with real response
payloads recorded.
