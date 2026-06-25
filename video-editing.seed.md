# SEED: video-editing

> seed-format: 1

> **You're an AI agent reading this seed.** Your job: run `## Steps` Step 0
> (Interview), then execute every Step in order, then perform `## Verify`.
> Two of these Steps are **your own reasoning, not a script** — Step 4
> (take-selection) is done by YOU reading the transcript; do not delegate it
> to a matcher program. When all `## Verify` evidence is in hand print
> `SEED_RESULT=DONE`; on a blocker print `BLOCKED_REASON=<short reason>` and stop.
>
> **FIRST decide which flow you're in** — see `## Three flows — pick ONE (DECISION RULE)` directly
> below. Everything from `## Goal` through the `## Phase-2 …` sections (Steps 0–11, Phase 2) is
> **FLOW 2 (PROCESS → vertical 9:16 reels)**. **FLOW 1 (COMBINE)** is the self-contained section
> above `## Goal`. **FLOW 3 (MULTICAM LONG-FORM → landscape 16:9)** is the self-contained section at
> the very bottom of this file.

## Generativity — Step-0 gate (read before any harden spend)

This is a **TRUE GENERATIVE seed**, per the seedlab standard (`seedlab/docs/PRINCIPLES.md` §2 "What is a SEED"; `seedlab/docs/test-a-seed.md` Step 0). The gold-standard reference is the teleprompter/almanac seed: **zero pre-baked source** — a blind agent GENERATES the software from intent + contracts + acceptance journeys and self-verifies against it.

**What that means for this seed, concretely:**
- **You GENERATE every pipeline script** (transcription call, offset/sign-validate, EDL builder, per-piece extractor, SAM3 framing, render/concat) from the **contracts** in `## Steps`, `## Verify`, and the rule blocks. The seed names the **intent, the inputs/outputs, the locked parameters, and the pass/fail gates** — not a program to paste.
- **There is NO pre-baked editing code in this file**: zero heredocs, zero base64 blobs, zero vendored/tarballed source, nothing you `cat >` into a file and run. The Step-0 generativity gate (PRINCIPLES §2) is therefore **PASS** — hardening this seed hardens the spec, and fixes fold back as **contracts**, never as pasted code.
- **The fenced blocks are illustrations, not the product**: (i) ASCII pipeline **diagrams**, and (ii) short **intent-level command examples** (one-liners showing the shape of an ffmpeg/curl invocation). They are guidance for the agent's own generation. **Pasted program-source lines = 0** → the "≤10% code / ≥90% instructions" bar is met in substance (the only fenced content is diagrams + command illustrations; if a strict line-counter still reads >10%, the single largest contributor is the Phase-2 ASCII diagram, which is a diagram, not code).
- **Two Steps are explicitly YOUR reasoning, not a script** (the generativity heart): Step 4 take-selection (you read the transcript) and Step 9 agent-reasoned camera switching. Do not replace them with a matcher program.

**Acceptance is measurable, not prose:** each flow's `## Verify` is a hard gate over **observable running state** (re-transcribe the delivered mp4, SAM3 head-safety on real rendered frames, A/V-lock numbers, the timing-artifact gate) — never "process launched" or a parameter cited in prose. A blind agent runs `## Verify` from zero; if it passes with no in-container fixes, the seed is proven.

## Three flows — pick ONE (DECISION RULE)

This seed covers **three distinct video-editing flows**. They share the final discipline
(A/V locked + `video start_time == 0.000`) and the rule that **the transcription is the
source-of-truth for every cut**; otherwise they do different things. **Decide first:**

| You were handed… | → FLOW | What it does |
|---|---|---|
| **Pieces a human editor already cut** — individual hook / body / CTA clips, already vertical **1080×1920**, with B-roll / jump-cuts / on-screen text baked in | **FLOW 1 — COMBINE** | concatenate them **AS-IS** into reels. NO transcription, NO `final_cut.py`, NO SAM3. The editor's cut is preserved untouched. |
| **Raw camera footage → a short VERTICAL reel** — one or two un-cut camera files (Cam A / Cam B), deliverable is **9:16 1080×1920** (hook/body/CTA pieces) | **FLOW 2 — PROCESS (reels)** | the per-piece vertical pipeline: Parakeet STT → `final_cut.py` silence-removal → SAM3 **vertical** reframe + centering-3 → assemble combos. |
| **Raw camera footage → a long LANDSCAPE video** — one or two long Cam A / Cam B takes, deliverable is **16:9 3840×2160** (a full talk/episode, NOT a reel) | **FLOW 3 — MULTICAM LONG-FORM (landscape)** | the content-aware landscape pipeline: sync A/B → `/transcribe` (source-of-truth) → segment+match → rough+fine cut → framing-discovery → `multicam-edit-v2` (camera-runs + zoom arcs + **per-shot landscape crops**, audio always Cam A) → render. |

**THE RULE (stated once):** *pieces already individually edited by a human → FLOW 1 (combine);
raw camera footage you turn into a **short vertical reel** → FLOW 2 (process); raw camera footage you
turn into a **long landscape video** → FLOW 3 (multicam long-form).* If the pieces are already
vertical 1080×1920 and visibly cut/B-rolled, they are editor-delivered → **FLOW 1** — do **NOT** push
them back through SAM3/`final_cut`/Parakeet (that re-processes a finished edit and destroys the
editor's work). **FLOW 2 vs FLOW 3 is decided by the deliverable's orientation/length:** 9:16 short
reel → FLOW 2; 16:9 long-form → FLOW 3.

> `final_cut.py` parameters are CEO-decided **per deliverable** — never substitute your own; if a value
> seems wrong, RAISE it to the CEO (that is how FLOW 3 went 0.30 → 0.35). **FLOW 2 (short vertical):
> `--pad-start 0.0 --pad-end 0.3`. FLOW 3 (long landscape): `--pad-start 0.0 --pad-end 0.35`, break
> threshold `0.35`** (CEO-raised on the accepted long-form run — 0.30 broke natural sentence pauses; see
> FLOW 3). **`no VAD`** = no editing-client energy/RMS VAD, in EITHER flow. (In FLOW 1 `final_cut` does
> not run.)
>
> **"no VAD" bans the EDITING CLIENT's energy/RMS VAD — NOT the STT server's internal VAD.** Speech and
> cut points are defined ONLY by the transcription (the source-of-truth), never by client-side audio
> energy. This does **not** ban the **server-side Silero VAD windowing inside the fixed `/transcribe`**
> — that is purely how a long file is chunked so the decoder stays reliable (it produces the very word
> timestamps that ARE the source-of-truth). Two different things; only the client-side energy heuristic
> is banned. See FLOW 3 → "Transcription is the source-of-truth (the fixed `/transcribe`)".

---

## Dev-validation harness — the LAST 3 MINUTES (fast seed check, NOT the production output)

**PRODUCTION edits the WHOLE video.** Every flow's real deliverable is the FULL source, full length —
never a slice. The last-3-minutes below is **only a development/test harness:** while building or
hardening this seed, run it on the final 3:00 of the source to validate quickly that the pipeline works
end-to-end without waiting for a full-length render. It is NOT the production deliverable and NOT a
quality-review unit.

- **DEV/VALIDATION ONLY — fast seed check:** restrict the edit to the final **3:00** of each input (e.g. Cam A `C4779`,
  Cam B `C4831`), A/B-offset aligned so both cameras cover the SAME window, and run the chosen flow on
  that window. A clean 3-min run proves the seed works; iterate on the seed using this fast loop.
  - **"Cut the final 3:00" means RESTRICT THE EDIT WINDOW, not create a slice FILE (CEO 2026-06-25 fold, harden #2 gap #9; reconciles with the no-intermediate rule).** Define the window `[dur − 180, dur]` in Cam-A time (Cam B = that window − tau) and **seek the FULL ORIGINALS** at that offset (`-ss`) for transcription / extraction / render — exactly as production does. Do **NOT** write a re-encoded/downscaled 3-min slice file (that would violate the SOURCE-RESOLUTION DISCIPLINE below). The only thing the harness changes vs production is the time bounds.
- **PRODUCTION — edit the FULL video:** once the seed is validated, run it on the ENTIRE source (full
  length). The whole-video edit is the deliverable. **Never ship the 3-min slice as the output.**
- Applies to every flow (FLOW 1 / 2 / 3): the 3-min slice is the dev harness; production is full-length.
  The same gold-standard bar applies either way.

---

## SOURCE-RESOLUTION DISCIPLINE — edit from the FULL ORIGINAL, set resolution only at export (CEO 2026-06-20)
**ALWAYS edit from the full original source resolution end-to-end. NEVER introduce a downscaled intermediate.** All cutting, camera switching, crop, and zoom operate on the **full original footage** (these cameras are native **4K 3840×2160**) — the ONLY place resolution is set is the **final export step**.
- A 1080p (or any downscaled) working slice is **WRONG as a default** — it throws away quality and corrupts crop/zoom (upscaling soft pixels). Do not create one.
- **For crop/zoom this is mandatory:** crop from the full 4K and scale to the delivery size at the final render → sharp. Cropping a 1080p slice upscales/softens.
- **FINAL EXPORT IS ALWAYS 4K (3840×2160) — ENFORCED, NOT A DEFAULT (CEO-decided 2026-06-22).** The deliverable is **always** native-4K regardless of the crops used during editing: full-frame shots are native 4K, punch-ins scale UP to 4K at the final render — `scale=3840:2160` on every shot. **1080p / 1440p / any sub-4K export is WRONG** (it throws away the 4K the CEO recorded — a real rejection). Do **NOT** re-debate crop-floor vs upscaling as a reason to ship sub-4K: the punch-in upscale (≤1.43× on the tightest crop with the 1.4×FHD floor) is accepted; the export is 4K, period. (If a genuinely smaller deliverable is ever explicitly ordered, **downscale ONLY at that final export**, never upstream — but the default and the standing rule is 4K.)
- **Render directly from the originals** (e.g. `ffmpeg -ss <camA_start> -i CAMA -ss <camB_start> -i CAMB -filter_complex <trims/crops at slice-time> … scale=<export> …`) so there is **no re-encoded intermediate** — one encode generation, full quality.

---

## Cut discipline — FALSE-START removal + CONSISTENT pad-end (CEO refinement 2026-06-18)

Two gold-standard requirements that apply wherever this seed does transcription-driven silence removal
(FLOW 2 Step 7, FLOW 3 cut). The CEO caught both on a real run; fold them every time.

> **HARD RULE — TRANSCRIPTION ONLY for pause/silence detection. ZERO acoustic.** Pauses, silences, and
> cut points are derived **only** from the transcription's word timestamps. **`silencedetect` / energy /
> RMS / any acoustic measure is PROHIBITED** anywhere in the cut (it is the old, banned approach). If
> word timestamps seem too loose to place a cut, fix it **inside the transcription domain** — word-level
> timestamps, a tighter STT setting, or forced alignment — never acoustic.

### A) Remove FALSE STARTS (not just silence)
Silence removal alone is NOT enough — the speaker's **false starts** (abandoned / restarted / stumbled
sentences) MUST be cut too. A false start is content the speaker began and re-attempted; leaving it in
reads as a stumble even with clean silence handling.
- **This is the agent's READING step** (the same judgment as take-selection — a Claude worker reads the
  transcript; it is NOT a pure matcher). Identify and DROP the abandoned fragment, keep the clean
  re-attempt.
- **Heuristic signals to flag for the agent's review:** (1) a short fragment that ends on a *dangling
  function word* (preposition/article/conjunction — e.g. "...na questão **do.**") immediately followed by
  a restart; (2) a clause whose **leading words repeat** the next clause's opening ("é uma coisa que eu
  configure, **é uma coisa que eu** itero..."); (3) a cut-off word / self-correction.
- **SPLICE RULE — preserve the previous valid word's tail, but GUARD against the stumble onset.** When
  you drop a false-start fragment, do **NOT** cut at the previous word's `end` (that clips the good word
  and the splice mismatches), and do **NOT** pad all the way up to the stumble word's `start` (that bleeds
  the **removed stumble's first word** into the output — the bug). Keep the previous good word PLUS up to
  the pad-end, **always ending ≥ GUARD before the stumble onset**, never before the good word ends:
  ```
  PAD = 0.30                                   # FLOW 2 (FLOW 3 long-form = 0.35)
  GUARD = 0.10                                 # hard buffer before the stumble's first word (CEO: 0.05 still leaked; 0.10 clean)
  silence_available = stumble_first_word.start - prev_good_word.end
  pad      = max(0.0, min(PAD, silence_available - GUARD))
  span_end = prev_good_word.end + pad          # then resume at next_good_word.start
  ```
  Net: the tail is up to PAD but **always ends ≥ 0.05 s before the stumble's first word starts**, so the
  removed stumble can never leak (audio or video). **VERIFY at every false-start splice:**
  `stumble_first_word.start - span_end ≥ GUARD` and `span_end ≥ prev_good_word.end`.
  (Real audit on the accepted run, GUARD=0.10: "ajuda" → drop "na questão do." → tail 0.22 s, lands 0.10 s
  before "na"; "configurei," → drop "é uma coisa que eu configure," → tail 0.06 s, lands 0.10 s before "é".
  No leak, no clip. Earlier: padding to `stumble.start` (0-guard) leaked the stumble's first word, and
  GUARD=0.05 still left an audible sliver — 0.10 is clean. If a future run still leaks, raise GUARD further.)
- Do NOT over-cut: intentional rhetorical repetition / anaphora ("se você quer X… se você quer Y…") is
  NOT a false start — keep it. When unsure, surface the candidate to the boss.

### B) CONSISTENT pad-end — TRANSCRIPTION ONLY, cap every word-gap at the configured pad
**Root cause of the inconsistency seen on the first run:** the cut left false starts in AND did not
normalize the trailing pause uniformly, so phrase-ends varied (some clipped, some long). **The fix is
purely transcription-based — no acoustic.**
- **Method:** walk the retained words (after false-start removal). For every consecutive pair, compute
  `gap = next.start − prev.end`. Wherever `gap > pad-end`, trim the gap down to **exactly the configured
  pad-end** (FLOW 2 = 0.30 s; FLOW 3 long-form = 0.35 s) — i.e. keep `prev.end + pad-end`, then jump to
  `next.start`. `pad-start = 0.0`. Gaps `≤ pad-end` are natural intra-phrase rhythm — leave them.
- **Result by construction:** every phrase boundary in the output has *exactly* the pad-end, and no gap
  anywhere exceeds it — consistent breathing at every cut, derived 100 % from the transcription.
- **If a real pause is still missed** (word timestamps overran it), do NOT reach for `silencedetect` —
  tighten within transcription: re-transcribe that span / use word-level timestamps or forced alignment
  to get an accurate `prev.end`, then re-apply the cap.
- **Verify after render (transcription only):** re-transcribe the OUTPUT and confirm no inter-word gap
  exceeds pad-end by more than ~1 frame. (Do NOT verify with `silencedetect`.)

---

# FLOW 1 — COMBINE (editor delivered pre-edited pieces)

> Proven on the CEO-accepted runs: the **4-sample manual-body combos** and the **200-video A/B test**
> (10 hooks × 2 bodies × CTA 01–05 × {editor pieces, our-pipeline pieces}). Both used exactly the
> steps below. These learnings previously lived only in ad-hoc `_MANIFEST.txt` files; this is the
> canonical home.

## Flow-1 Goal
Editor-delivered pieces — each already a finished vertical **1080×1920** clip (a hook, a body, or a
CTA, with the editor's cuts / B-roll / text baked in) — are **concatenated AS-IS** into combination
reels (`hook + body + cta`). The editor's cut is **preserved untouched** at the video-stream level;
the only processing is an audio re-encode + a clean concat that leaves **no black first frame and no
A/V drift**.

## Flow-1 Done (observable)
- Every output is **1080×1920, h264, 24000/1001**, AAC 256k 48k stereo.
- **`video start_time == 0.000000` AND `== audio start_time`** on EVERY output — the 21 ms
  black-first-frame is gone. Verified per file: this is the hard gate.
- The editor's frames are unchanged (no reframe, no silence-trim, no re-cut): a combine reel is just
  its source pieces back-to-back.
- A `_MANIFEST.txt` maps each output to its source pieces (and, for an A/B build, each A to its
  matched B) with the per-file `start_time` proof.

## Flow-1 Inputs
| name | required | default | ask |
|---|---|---|---|
| `PIECES` | yes | none | "Paths to the editor's pre-edited pieces (hooks, bodies, CTAs), each already vertical 1080×1920." |
| `COMBINATIONS` | yes | none | "Which hook×body×cta combinations to build (or 'all')." |
| `OUT_DIR` | no | `~/Downloads` (review) or the NAS delivery dir | "Where to deliver the finished reels." |

## Flow-1 Steps

### F1.0 — Confirm you're in FLOW 1
Probe a piece: `ffprobe` shows **1080×1920** and it's visibly a finished edit (cuts / B-roll / text).
If so, do **not** transcribe, `final_cut`, or SAM3. If the pieces are raw camera files → STOP, you're
in FLOW 2.

### F1.1 — Per-segment prep (audio only; video untouched)
Editor pieces are usually **PCM s24le** audio + a stray **`tmcd`/data** timecode track. Make every
segment codec-uniform for a clean join **without touching the picture**:
- audio **PCM → AAC 256k, 48 kHz, stereo**
- **drop the data/tmcd stream** (`-map 0:v:0 -map 0:a:0`; the mov muxer's auto timecode track is
  suppressed with `-write_tmcd 0`)
- **video is NOT re-encoded here** — preserve the editor cut.

You may skip a standalone prep pass and feed the raw pieces straight into the F1.2 concat filter
(it normalizes audio inline via `aformat`) — equivalent result.

### F1.2 — Concat + the `start_time=0` re-encode (THE flow)
**One ffmpeg per output: concat FILTER (not the demuxer) → re-encode → start_time 0.** The concat
*filter* (a) tolerates pieces whose video params differ (e.g. editor pieces and our-pipeline pieces
in the **same** reel, as in the A/B build) and (b) lets us reset PTS so the output starts at 0.
```bash
AF="aformat=sample_fmts=fltp:sample_rates=48000:channel_layouts=stereo"
# NB: the -filter_complex string MUST stay on ONE line. A line-break inside the quotes
# leaves a stray whitespace token that ffmpeg parses as an empty filter ("No such filter: ''").
ffmpeg -y -i SEG1 -i SEG2 -i SEG3 -filter_complex \
 "[0:v]fps=24000/1001,setsar=1[v0];[1:v]fps=24000/1001,setsar=1[v1];[2:v]fps=24000/1001,setsar=1[v2];[0:a]$AF[a0];[1:a]$AF[a1];[2:a]$AF[a2];[v0][a0][v1][a1][v2][a2]concat=n=3:v=1:a=1[vc][ac];[vc]setpts=PTS-STARTPTS[v];[ac]asetpts=PTS-STARTPTS[a]" \
 -map "[v]" -map "[a]" \
 -c:v h264_videotoolbox -b:v 12M -bf 0 -pix_fmt yuv420p -tag:v avc1 \
 -c:a aac -b:a 256k -video_track_timescale 24000 -movflags +faststart OUT.mp4
```
> For N pieces other than 3, scale the pattern: one `[i:v]fps=…,setsar=1[vi]` + `[i:a]$AF[ai]` per
> input, list all `[v0][a0]…[vk][ak]` before `concat=n=N:v=1:a=1`. (n=1 is a valid degenerate combine
> — it still applies the `setpts` reset + `-bf 0` re-encode that lands `start_time=0`.)
Non-negotiables baked in:
- **`setpts=PTS-STARTPTS` / `asetpts=PTS-STARTPTS`** — reset the first frame to PTS 0. This is what
  makes `start_time == 0.000`. Editor pieces carry a ~0.021 s first-frame composition offset; without
  this reset the reel opens on a black frame (the bug the CEO caught).
- **`-bf 0`** — no B-frames → no composition-time offset that would re-introduce a non-zero start.
- **re-encode `h264_videotoolbox -b:v 12M`** — matches the accepted combo format; videotoolbox emits
  no B-frames and a clean PTS-0 start. (Yes, FLOW 1 re-encodes the video once — that is the only way
  to land `start_time==0` given the source pieces' offset. The editor's *frames/cut* are unchanged;
  only the container/codec is rewritten.)
- **Do NOT add the `-itsoffset … -avoid_negative_ts make_zero` realign here.** That realign belongs to
  the *stream-copy / concat-demuxer* path; on this concat-FILTER path it **re-introduces** the
  ~0.021 s half-frame offset on the copy-remux. `setpts` in the filtergraph is the correct fix.

### F1.3 — (variant) source pieces only exist inside a prior combine build
If the pieces you need aren't standalone files but live inside already-built combos, extract them
**losslessly** at the forced-IDR join keyframes (`ffmpeg -ss <join> -i combo -c copy piece.mp4`): the
join is the keyframe whose pts breaks the regular GOP grid. Anchor one boundary from a cleanly-detected
combo, then derive the rest algebraically from combo durations and snap to the nearest keyframe.
(This sourced Group B of the A/B test from the 200-combo build, byte-identical to the approved pieces.)

## Flow-1 Verify (hard gate — per output)
- **`video start_time == 0.000000` AND `== audio` start_time, on EVERY output.**
  `ffprobe -v error -select_streams v:0 -show_entries stream=start_time -of csv=p=0 OUT.mp4`.
  Any non-zero → FAIL (= a black first frame). Check all of them, not a sample.
- **No black frame anywhere:** `blackdetect=d=0.02:pic_th=0.98` returns nothing (sample across joins
  + the first frame; `signalstats` YAVG at t≈0 is normal, not <16).
- **A/V locked:** `|vdur − adur| ≤ 0.042 s` (≤ 1 frame).
- **Format:** 1080×1920, `r_frame_rate=24000/1001`, AAC 256k, `has_b_frames=0`.
- **Editor cut intact:** the picture is the source pieces unchanged (no reframe / trim). Spot-check.
- Write `_MANIFEST.txt` (output → source pieces + `start_time` proof; A↔B map for A/B builds).

## Flow-1 Failure modes
- **Ran SAM3 / `final_cut.py` / Parakeet on editor-delivered pieces.** Re-processes a finished edit
  (re-reframes, re-trims) → destroys the editor's work. **Detect:** you're transcribing / reframing
  1080×1920 already-cut clips. **Fix:** editor pieces → FLOW 1, no pipeline.
- **Black first frame (the 21 ms bug — caught on the 200 combos and the 4 samples).** Cause = the
  editor pieces' first-frame composition offset (`video start_time ≈ 0.021`, audio `0.000`), carried
  through `-c copy` concat. **Detect:** video `start_time ≠ 0`. **Fix:** the F1.2 concat-FILTER +
  `setpts` reset + `-bf 0` re-encode. **NOT** the itsoffset realign (it puts the offset back here).
- **`-c copy` / concat-demuxer of editor pieces** → carries the start offset (black frame) AND AAC
  packet misalignment at joins (+~21 ms tail, "Non-monotonic DTS"). **Fix:** the F1.2 re-encode path.
- **Concat-demuxer can't mix different video params** (editor pieces + our-pipeline pieces in one
  reel). **Fix:** the concat FILTER (it decodes/normalizes), which F1.2 uses.
- **Hydrating online-only Dropbox source pieces:** trigger a **file-coordination** read (a coordinated
  `NSFileCoordinator` read materializes the File Provider placeholder, headless, **no window**).
  `cat`/`dd`/`brctl`/`rclone` do **not** hydrate this Dropbox setup; `qlmanage`/`open`/Preview pop GUI
  windows — never use them in background.

---

# FLOW 2 — PROCESS (raw camera footage) — the full pipeline from here down

## Goal

A two-camera ("A/B") talking-head recording becomes a set of **clean, individually
usable good-take clips — one per piece of content — cut from BOTH cameras, with
audio always from Cam A, A/V locked (audio duration == video duration to the
millisecond) and A/B locked (both angles frame-aligned).** The output is the raw
gold the CEO assembles into final reels/combos. This is the established Mac-Mini
multicam process (`/Users/delattre/multicam/`), captured so it runs correctly for
**every** video. Two modes: **with** a reference script (match each script piece
to its best take) and **without** one (you find the gold from the transcript alone).

The hard-won, non-negotiable invariants this seed enforces:
- **Transcription is our Parakeet API ONLY** (`parakeet-tdt-0.6b-v3`). Never Whisper or any other model.
- **Finding the good takes is an AGENT judgment step** — a Claude worker literally reads the full transcript and picks the gold. It is NOT an automated transcript-analyzer (that is exactly what failed).
- **Extraction is frame-exact, audio-always-Cam-A, with the Step-10 audio-pad**, A/B offset from `sync_offset.py` *validated by independent transcription*, never a blindly-trusted sign.
- **Everything runs local (Mac / Mac Mini). Zero S3 egress** — never pull raw video out of S3 to a non-in-region machine.

## Done

All observable, proven by `## Verify`:

- **Per-piece clips exist for both cameras.** For each selected piece `<NN>_<label>`,
  `<out>/camA/<NN>_<label>.mp4` and `<out>/camB/<NN>_<label>.mp4` exist, full source
  resolution (e.g. 3840×2160), 24000/1001 fps.
- **A/V locked on every clip.** For every clip, `ffprobe` video-stream duration and
  audio-stream duration match within **< 1 frame (≤ 0.042 s)** — proof the Step-10
  audio-pad ran and there is no accumulating drift.
- **Audio is Cam A on every clip** (the good mic). The camA and camB clip of the same
  piece carry the **same** audio track → trivially A/B-synced for assembly.
- **A/B picture is aligned.** `sync_offset.py` produced an integer-frame offset whose
  sign was **validated** (independent Cam A vs Cam B transcription at early/mid/late
  windows yields matching content words). A side-by-side `sync_check` clip exists.
- **Content is correct, confirmed by Parakeet** (not the matcher): each clip's audio,
  re-transcribed via Parakeet, contains the intended piece's words. With a reference
  script, every script piece that was actually recorded has a clip (and any genuinely
  missing piece is reported with the verbatim transcript region as proof).
- **A `manifest.json` / EDL** records, per piece, the Cam A source start, n_frames,
  label, and (script mode) the matched script piece.
- **No egress incurred.** `aws ce get-cost-and-usage` for `USW2-DataTransfer-Out-Bytes`
  over the run window shows no increase attributable to this work.

## Inputs

| name | required | default | detect | ask |
|---|---|---|---|---|
| `CAM_FILES` | yes | none | two source MP4/MOV in the working dir (e.g. `C48xx.MP4`, `C47xx.MP4`) | "Paths to the two camera files. Which is the close-up/better-mic angle (Cam A) and which is the wide (Cam B)? If unsure I will pick Cam A by loudest mean audio." |
| `REFERENCE_SCRIPT` | conditional (script mode) | none | a `.md`/`.txt` script with the pieces the talent read (e.g. `teleprompter-launch-reels-ptBR.md`) | "Is there a reference script the talent read (gives the exact piece list to match)? Path, and which language it's in (match the transcript's language). If none, I'll find the gold from the transcript alone." |
| `LANGUAGE` | no | `pt` | infer from the reference script / a 30s Parakeet probe | "Spoken language (default pt-BR)." |
| `STT_URL` | no | `http://192.168.15.14:8101` | `curl -sf $STT_URL/health` == 200 | "Parakeet STT API base URL. Default is our server. If health != 200, I will START it, not switch models." |
| `OUT_DIR` | no | `<workdir>/out` | dir exists | "Where to write camA/ camB/ + manifest. Default <workdir>/out." |
| `RUN_HOST` | no | any host with ffmpeg + a numpy-capable `python3` + reachable STT/SAM3 | `command -v ffmpeg && python3 -c "import numpy" && curl -sf $STT_URL/health` | "Where to run. ANY host with ffmpeg, a numpy-capable python3, and network access to STT/SAM3 works — the seed generates its own scripts (Steps 3, 5). The Mac Mini (`/Users/delattre/multicam`) is the historically-proven box but NOT required. Footage must be local to the run host (copy it there; never pull from S3)." |

## Components

| component | source | notes |
|---|---|---|
| Established multicam pipeline (**OPTIONAL convenience — NOT a dependency**) | `mac-mini:/Users/delattre/multicam/pipeline/` | If present, these helpers exist: `transcribe_full.py`, `sync_offset.py`, `validate_sync.py`, `word_sync.py`/`refine_sync.py`/`drift_correct.py`, `verify_av_sync.py`, `render_sidebyside.py`, and `render_cuts.py` (the *switched-edit* reel — NOT this seed's per-piece extractor, see Step 5). **They are absent on a fresh host, and this seed never requires them:** the offset finder (Step 3 GCC-PHAT contract) and the per-piece extractor (Step 5 recipe) are fully specified here for you to GENERATE. This seed is **self-contained** — do not assume any `LEARNINGS.md`, `sync_offset.py`, or `multicam/` dir exists on the host. |
| Parakeet STT API | `http://192.168.15.14:8101` (docker container `stt-api-stt-api-1`, GPU) | model `nvidia/parakeet-tdt-0.6b-v3`; `POST /transcribe` multipart `audio=@file` → `{text, segments, words:[{word,start,end}]}`. Long-audio capable (local attention) — send the whole Cam A track. |
| ffmpeg/ffprobe (`/opt/homebrew/bin`) | homebrew | hardware videotoolbox decode+encode. |
| A real Claude agent | this session / a spawned worker | performs Step 4 (take-selection) by READING. |

## Steps

### Step 0 — Interview + preflight
Read `## Inputs`, run each `detect`, send ONE consolidated message: which camera is A
(better mic) vs B, whether there is a reference script (+ language), confirm the
Parakeet API is reachable, and the run host. Do not proceed until answered.

**Preflight (autonomous — establish the toolchain before Step 1, never assume it).** This
seed GENERATES its own offset/extractor scripts (Steps 3, 5); it does **not** depend on any
pre-installed pipeline. Confirm, in order, and FIX before proceeding:
- **ffmpeg/ffprobe** at `/opt/homebrew/bin` (or `command -v ffmpeg`). Hardware videotoolbox path.
- **STT** `curl -sf $STT_URL/health` == 200 (and **SAM3** `:8100/health` if Phase 2 / reframe is in scope). These are ACCESSIBLE external services — never install or generate them; if down, START the container per Step 2, do not switch models.
- **A numpy-capable `python3`** (Step 3's GCC-PHAT needs numpy). Probe `python3 -c "import numpy"`.
  If it fails (a common trap: a Homebrew/pyenv Python with a broken `pip`/`venv`, e.g. a
  `pyexpat`/libexpat symbol mismatch), do **not** fight pip — fall back in this order:
  (a) `uv run --with numpy python …`; (b) macOS system `/usr/bin/python3` (ships numpy);
  (c) **create a venv** (works on Linux too, where there is no macOS `/usr/bin/python3` and the system
  python may be PEP-668 externally-managed): `python3 -m venv .venv && .venv/bin/pip install numpy`, then
  use `.venv/bin/python` (CEO 2026-06-25 fold, harden #2 gap #12); (d) any interpreter where `import numpy`
  succeeds. Record which interpreter you bound to and use it for every Step-3 computation. This is
  preflight, not a mid-run "fix".

### Step 1 — Stage footage on the run host, in-region, zero egress
Put both source files on `RUN_HOST` (default `mac-mini:~/multicam/`). If the footage
lives only in S3, copy it **in-region on the EC2** (or use the already-local copy) —
**never** download raw video from S3 to the Mac/Mac-Mini (that is the egress mistake
the cost incident was about). Confirm Cam A = the louder mic:
`ffmpeg -i <file> -t 120 -af volumedetect -f null -` → Cam A has the higher mean_volume.

### Step 2 — Transcribe Cam A with Parakeet (ONLY)
`transcribe_full.py` (or a direct multipart POST) sends the full Cam A audio
(`-ac 1 -ar 16000 -af dynaudnorm`) to `STT_URL/transcribe` and saves word-level JSON
in **Cam A original time**. **If `/health` != 200, FIX the API** — `ssh server "docker
start stt-api-stt-api-1"`, wait for health 200 (model load ~30s), retry. Do **NOT**
fall back to Whisper or any other model. See Failure modes.

### Step 3 — A/B offset, then VALIDATE its sign
**Generate the offset finder from this contract — do NOT assume a host `sync_offset.py` exists.**
A pre-existing `mac-mini:~/multicam/pipeline/sync_offset.py` is an *optional convenience*; on any
fresh host it is absent, so the seed pins the algorithm here and you generate it (≈30 lines, numpy):

> **GCC-PHAT A/B offset (the contract).** Decode both tracks to mono PCM at one shared rate
> (48 kHz is fine: `ffmpeg -i CAM -ac 1 -ar 48000 -f wav`). Compute the generalized
> cross-correlation with phase transform: `A=rfft(a); B=rfft(b); R=A*conj(B); R/=|R|+eps;
> cc=irfft(R)`, search a window WIDE ENOUGH for the real offset, `argmax|cc|` → integer sample `shift`; **`tau = shift/fs`**
> is the delay where **`camB(t) ≈ camA(t − tau)`**. **SEARCH RANGE — auto-widen, do NOT cap at ±10 s (CEO 2026-06-25, harden #4 gap #1).** A fixed ±10 s can MISS the true peak — this rig has produced a **26.6 s** desync, and cameras started seconds-to-tens-of-seconds apart (their durations differ by ~14 s here). Use `search = ±max(30 s, 2·|dur_a − dur_b| + 10 s)`, and if `snr` is low or the peak sits at the search edge, **widen and re-search**. (The sign is ALWAYS validated next regardless — see below.) **SIGN CONVENTION (gap #2): the formula `camB(t) ≈ camA(t − tau)` means Cam B is seeked at `astart − tau` (Step 5 extractor); but the rfft sign is implementation-dependent, so NEVER trust it — empirically validate (independent A/B re-transcription Jaccard at +tau vs −tau) and use whichever sign aligns the speech.** Report `tau` in **seconds** (sub-frame
> precision — keep it) AND in frames (`tau/FRAME`, `FRAME=DEN/NUM=1001/24000`), plus
> `snr = peak/rms(cc)`. `offset_frames = round(tau/FRAME)` is for logging/`sync.json`; the
> **seek uses the exact `tau`**, never the rounded value (see Step 5). Persist
> `{tau_s, offset_frames, leader, snr}` to `sync.json`.

Any docstring/comment in a borrowed `sync_offset.py` that hard-codes ground truth from an OLD
recording ("Cam A C4819 started first") is **not your recording — ignore it.** Trust the computed
sign only after validating:
- **Validate the SIGN (generate the windows yourself — no `validate_sync.py`/`word_windows.py`
  required):** pick a handful of word-dense spans from the Step-2 transcript spread early/mid/late,
  independently re-transcribe Cam A and Cam B (B at `A_time − tau`) per window via Parakeet, and
  confirm matching content words. Matching content at all three points rules out a **gross sign
  inversion** (the ~2× desync failure) and drift. (If `validate_sync.py` happens to be present you
  may call it, but it is not a dependency.)
- **Precision at small offsets:** content-word matching is COARSE — at a ~1-frame offset
  it cannot tell +1 from −1 frame (a 42 ms shift doesn't change words). The sub-frame proof
  is the **quantitative residual** (`sync_offset.py` SNR / a GCC-PHAT residual). Use the
  residual for precision, the transcription match for the gross-sign guardrail.
- **Drift check (per-piece path):** measure the source GCC-PHAT residual (Cam A vs Cam B at
  `A_time − offset`) at early/mid/late. **Flat within ½ frame (≤ 21 ms) ⇒ no drift, the fixed
  `offset_frames` is correct.** Only if the residual GROWS toward the end apply a per-window
  `b_start = slope·camA_t + intercept` (`word_sync.py` slope/intercept + GCC). Do NOT invoke
  `drift_correct.py` in this flow — it needs a camera-tagged switched-edit EDL this per-piece
  path never builds; it belongs to the switched-edit reel only. Also ignore
  `refine_sync.py`'s `needs_correction` flag here: its 2 ms threshold is for sub-frame
  re-extraction in the switched edit; this path snaps to the frame grid and the tolerance is
  ½ frame (21 ms), so a single-digit-ms residual is correct, not a defect.

### Step 4 — FIND THE GOLD — *agent reads the transcript* (NOT a script)
**This is your judgment, done by reading.** Open the full Parakeet transcript (with
timestamps) and read it end to end. Do not write or run a program that "analyzes" the
transcript to pick takes — that automation is what produced wrong output.

- **Mode A — with a reference script:** for each script piece, read the transcript and
  find where the talent delivered it. The talent typically marks takes verbally — an
  announce ("Gancho 2" / "CTA 4"), a "gravando" to start, and a verdict **"ficou bom"**
  (keeper) / **"ficou ruim" / "não deu certo"** (redo). The good take is the clean
  delivery just before a "ficou bom" (use the LAST good take if several). Read past ASR
  noise: if a region shows an obviously hallucinated repeat (e.g. "Gravando" ×N) where a
  piece should be, **re-transcribe just that window via Parakeet** and read it — the take
  is usually there. Record each piece's [start,end] in Cam A time. If a script piece was
  genuinely never recorded, note it with the verbatim transcript region as proof.
- **Mode B — no reference script:** read the transcript and identify the gold yourself —
  the coherent, keeper segments worth cutting (drop dead air, false starts, "ficou ruim"
  takes, setup chatter). You define the piece list.
- **Mode C — CONTINUOUS LIVE talk (no retakes, no take list) (CEO 2026-06-25 fold, harden #2 gap #10).** A live Q&A / talk is one continuous take, NOT discrete retakes — there is no "ficou bom" verdict to anchor on. Keep the whole talk in order; your reading-judgment DROPS only: (a) **editor-instructions the speaker says to themselves** ("Editor, pode cortar essa tentativa", "corta isso", "de novo") — meta-direction, never content; (b) **post-talk / off-script chatter** after the talk has clearly ended ("ficou bom, Pueblo?", banter with crew); (c) the same **false starts / stutters / dead air** the cut-discipline section already covers. There is no per-piece selection — it's one keeper span minus these drops. Record dropped spans with their verbatim text as proof (same as Mode A's "never recorded" note).

Output a window list: `[(label, start_s, end_s), ...]` in Cam A time, trimmed to the
spoken line only (exclude the "gravando"/"ficou bom" markers). Spot-read 2-3 against the
script before extracting.

### Step 5 — Extract per piece, per camera (write a per-piece extractor)
**`render_cuts.py` is NOT this step's tool.** That script makes the *switched-edit reel*
(one camera per window, crop to a Cam-A face anchor, scale to 1920×1080, concat into a
single video). This deliverable is the opposite: **per-piece, BOTH cameras, FULL source
resolution, no crop, separate files.** Borrow only its A/V method (frame-exact + Cam-A
audio + Step-10 pad) and write your own extractor. Reference recipe (self-contained — do
not depend on any mac-mini script):

```python
NUM,DEN=24000,1001; offset_s = tau_s                   # EXACT sub-frame tau from Step 3 (NOT offset_frames*DEN/NUM — the rounded value adds up to ½-frame residual)
for nn,label,cs,ce in WINDOWS:                          # cs,ce = your Step-4 take, Cam A time
    s=max(0,cs-0.10); e=ce+0.35                         # small lead/tail pad
    f0=round(s*NUM/DEN); f1=round(e*NUM/DEN); n=f1-f0   # snap to integer frame grid
    astart=f0*DEN/NUM; dur=n*DEN/NUM
    for cam,vfile,vstart in (("camA",CAM_A,astart),("camB",CAM_B,astart-offset_s)):
        # B at A_time - offset_s (or per-window b_start from drift_correct for long takes)
        tmp=f"{nn}_{label}_{cam}.tmp.mov"; out=f"{cam}/{nn}_{label}.mp4"
        ffmpeg -y -hwaccel videotoolbox -ss {vstart} -t {dur+0.5} -i {vfile} \
               -ss {astart} -t {dur+0.5} -i {CAM_A} \
               -filter_complex "[0:v]fps={NUM}/{DEN},setpts=PTS-STARTPTS[v];[1:a]asetpts=PTS-STARTPTS[a]" \
               -map [v] -map [a] -frames:v {n} \
               -c:v hevc_videotoolbox -q:v 60 -tag:v hvc1 -c:a pcm_s16le {tmp}
        vdur = ffprobe -select_streams v:0 -show_entries stream=duration -of csv=p=0 {tmp}
        ffmpeg -y -i {tmp} -af apad -t {vdur} -c:v copy -c:a aac -b:a 256k -movflags +faststart {out}
```
Rules baked in: full frame (no crop/scale), **`-frames:v n`** exact count, **audio always
Cam A** (input 1), **Step-10 pad** (`apad -t vdur`, NEVER `-shortest`), hardware decode+encode,
never libx264/x265 for 4K. Standalone clips so concat isn't done here; the *assembly* step
must use ProRes intermediates or a concat filter (never `-c copy` of separate HEVC → green
frames). Write `manifest.json` (per piece: Cam A start, n_frames, label, matched script piece).

### Step 6 — Deliver
Place `camA/`, `camB/`, `manifest.json`, `README.txt` under `OUT_DIR`. Optionally also
copy to the canonical asset location `Sorted Media/<date>/<topic>/AB-processed/` — but only
if you can derive it: `<date>` = the recording date (footage mtime / a `YYYY-MM-DD` in the
filename); `<topic>` = a `CANONICAL_TOPIC` given in the Interview (e.g. `danedelattre-<slug>`).
If either can't be derived, deliver to `OUT_DIR` only and ask the user for the canonical
path — do NOT guess a path. Note Cam A = audio source on both angles. Report any missing
script piece with the verbatim transcript region as proof.

## Verify

Runs as a script; the seed is proven only when it passes.

1. **Counts:** `camA/` and `camB/` have the same number of clips == the selected piece count.
2. **A/V lock (every clip):** for each `f`, `|vdur - adur| <= 0.042` where
   `vdur=ffprobe -select_streams v:0 -show_entries stream=duration`,
   `adur=...a:0...`. Any clip failing → FAIL.
3. **Resolution/fps:** each clip is the source resolution and `r_frame_rate=24000/1001`.
4. **A/B picture sync — measured from the SOURCES, not the clips.** The camA and camB
   clips share ONE Cam-A audio track by design, so you CANNOT cross-correlate the clips'
   audio to get A/B offset — don't try. Proof = source-level GCC-PHAT residual between Cam A
   and Cam B (B at `A_time − offset`) at early/mid/late pieces, each **< ½ frame (≤ 0.021 s)**,
   plus a visual side-by-side `sync_check/` clip — build it yourself (don't use
   `render_sidebyside.py`, that's the switched-edit tool): for a sample piece,
   `ffmpeg -ss <Astart> -t <d> -i CAM_A -ss <Astart-offset_s> -t <d> -i CAM_B -filter_complex
   "[0:v]scale=-2:720[a];[1:v]scale=-2:720[b];[a][b]hstack" sync_check/<piece>.mp4`.
   `sync.json` records the validated `offset_frames` + `snr`. **Do NOT use `verify_av_sync.py`
   here** — it's built for the switched-edit final cut (cumulative EDL, camera-tagged windows),
   not per-piece clips.
5. **Content correctness via Parakeet (not the matcher):** re-transcribe 3 sampled clips
   (incl. one CTA/short piece) through `STT_URL/transcribe`; the returned text matches the
   intended piece (and, script mode, the script line). Print clip→text.
6. **No egress.** Primary proof is **structural** (always available): no command in Steps
   1-5 reads from S3 — footage is local on the run host and all transc/render is local —
   so no GetObject on the bucket is possible. IF AWS creds are present
   (`aws sts get-caller-identity` succeeds), ALSO confirm via Cost Explorer:
   `aws ce get-cost-and-usage --metrics UsageQuantity` filtered to `Amazon S3` +
   `USW2-DataTransfer-Out-Bytes` shows ~0 GB over the run. **If `aws` is unavailable for
   ANY reason — no creds, OR the binary isn't installed (`command -v aws` empty), OR any
   error — the structural proof stands. NEVER let this gate crash or fail Verify** (guard
   the call: skip on missing binary / non-zero exit, don't let it abort the script).

## Failure modes

- **Used Whisper / any non-Parakeet model → wrong takes (THE failure that created this seed).**
  Whisper hallucinated a `"Gravando"` loop over a real take and the matcher dropped CTA 4
  as "missing"; the CEO had recorded it. **Detect:** a region of repeated identical tokens
  where speech should be; takes that "disappear". **Fix:** transcription is Parakeet ONLY.
  If `:8101/health` != 200, **restart the container** (`ssh server "docker start
  stt-api-stt-api-1"`, wait for health 200) — do NOT substitute a worse model. Re-transcribe
  the suspect window with Parakeet and read it.
- **Take-selection automated instead of read by an agent.** A program that "finds good
  takes" repeats the original mistake (it can't read intent, gets fooled by ASR noise).
  **Detect:** Step 4 produced windows with no agent having read the transcript. **Fix:** a
  real Claude worker reads the full transcript and selects; the only scripts allowed in
  Step 4 are dumping the transcript and re-transcribing a suspect window.
- **A/B offset sign inverted → ~2× desync (LEARNINGS "THE BIG ONE").** `find_offset`/
  `sync_offset` sign has been wrong, doubling skew (~26.6s once). **Detect:** Step 3
  validation (independent A/B transcription) words don't match; side-by-side lips off.
  **Fix:** never trust the sign blind — validate; measure who started first directly.
- **A/V drift from time-based cutting.** Cutting video by `select='between(t,…)'` while
  cutting audio by `atrim` desyncs ±1 frame/window and accumulates (+1.33s over 18min).
  **Detect:** Verify #2 fails; audio late at the end. **Fix:** integer-frame windows
  (`-frames:v n`) + Step-10 audio-pad (probe vdur, `apad -t vdur`).
- **STT crashes the GPU (it happened).** Starting/overloading the STT container can throw a
  CUDA error that knocks GPU0 off the bus (`nvidia-smi` can't get the handle; container
  won't restart; `daily-yield-stt` stalls). **Detect:** `:8101` down + `nvidia-container-cli
  nvml error`. **Fix:** it often recovers; otherwise a controlled GPU reset / host reboot —
  but that box runs the **live stream**, so coordinate, don't reboot blind. Send Cam A as one
  request (Parakeet handles long audio) rather than hammering it.
- **Used `render_cuts.py` as the extractor → wrong deliverable.** It outputs a 1080p
  crop-and-switch reel (one camera per window, concatenated), not per-piece full-res both-
  camera clips. **Detect:** clips are 1920×1080 / cropped / only one camera / a single file
  (Verify #1/#3 fail). **Fix:** use Step 5's per-piece extractor (full frame, both cameras,
  separate files); `render_cuts.py` is only for the switched-edit reel.
- **HEVC concat green frames.** `-c copy` concat of independently-encoded HEVC clips
  corrupts at joins. **Fix:** ProRes intermediates → one HEVC encode, or a concat filter
  (`concat=n=N`). (Per-piece deliverables here are standalone — this bites the assembly step.)
- **Software decode → `speed<1x` on 4K.** Forgot `-hwaccel videotoolbox` before an `-i`.
  **Fix:** hardware decode AND encode; never libx264/x265 for 4K.
- **Egress (the cost incident).** Pulling raw video out of S3 to the Mac/Mac-Mini bills as
  S3 egress. **Detect:** `DataTransfer-Out-Bytes` rises during the run. **Fix:** keep
  footage local / extract audio in-region on the EC2; never download `raw/` to a non-region host.
- **Wrong camera = audio source.** Cam B mic is much quieter; using it ruins audio.
  **Fix:** Cam A (loudest mean_volume) is always the audio + reference clock.

## Cleanup

Remove `OUT_DIR` and any staged footage copy on the run host. The Parakeet container and
the established pipeline stay installed (they are infrastructure, not per-run artifacts).

---

# Phase 2 — The EDIT: vertical 9:16 multicam reel

> Captured from the real run the CEO **accepted** (teleprompter Hook1+Body1+CTA1). These are
> the steps/commands that ACTUALLY worked, in order, with the lessons he caught folded in.
> Phase 1 gives matched per-piece camA/camB clips; Phase 2 turns chosen pieces into a final
> vertical reel. Order matters: **silence → reframe → A/B multicam → measured zoom → concat.**
> Everything runs locally (Mac + Mac-Mini) + the two server APIs. ZERO S3 egress.

## Phase-2 Goal
A combination of pieces (e.g. Hook1 + Body1 + CTA1) becomes ONE **vertical 1080×1920** reel:
silence removed, **audio always cam-A**, cut between cam-A and cam-B (A/B), with
**measurement-driven in-camera zoom** — and no lip-sync drift, no head-crop, no frame creep.

## Phase-2 pipeline at a glance (ASCII)

How one piece is edited, and how edited pieces fan out into the launch-reel combos
(transcription-driven, NO VAD, NO relight; this batch is 100% cam-A + centering-3):

```text
  cam-A SOURCE segment (4K 3840x2160; one take = one hook/body/cta)
    -> [1] PARAKEET transcription -> word-level timestamps (NO VAD; speech = word spans, silence = gaps)
    -> [2] TRANSCRIPTION-DRIVEN silence removal (final_cut.py --pad-start 0.0 --pad-end 0.3; cut points = word
           timestamps ONLY; start AT first word, keep 0.3s after last word clamped to clip end; nothing on top
           — no energy/RMS onset, no leading-silence trim, no tail-normalize)
    -> [3] SAM3 VERTICAL reframe (face bbox -> 9:16 crop 1215x2160 -> scale 1080x1920)
    -> [4] CENTERING-3 fluid (SAM3 head @0.1s -> one-euro mc=1.8 b=0.25 -> 30fps sendcmd crop x; head dead-center)
    =  EDITED PIECE (vertical 1080x1920, audio cam-A; lead/tail = 0.0 / 0.3) [NO relight]
  ...repeat for all 22 pieces: 10 hooks | 2 bodies | 10 CTAs...
    -> [5] CONCAT hookX+bodyY+ctaZ (encode each piece -> H.264 once; concat-demuxer -c:v copy; PTS-realign;
           seam = prev piece's pad-end tail + next piece starting at its first word; no clipped words)
    -> [6] 200 COMBOS = 10 hooks x 2 bodies x 10 CTAs -> hookX-bodyY-ctaZ.mp4 (9:16, ~55-58s each)
```

(A/B multicam, Steps 8–10, stays available for future videos; this launch-reels batch is 100% cam-A.)

## Phase-2 RUNBOOK (reproducible — exactly what was run; a fresh engineer follows this verbatim on any new cam-A footage)

Inputs: cam-A source segment clips (4K 3840×2160, one take = one piece). APIs up: Parakeet STT `:8101`, SAM3 `:8100`. ffmpeg = `/opt/homebrew/bin/ffmpeg`. Per piece, in order:

```bash
# 1) PARAKEET transcription (NEVER Whisper, NEVER VAD/energy/RMS)
ffmpeg -nostdin -v error -y -i SRC.mp4 -ar 16000 -ac 1 -af dynaudnorm /tmp/a.wav
curl -s -X POST http://192.168.15.14:8101/transcribe -F "audio=@/tmp/a.wav" -o SRC_trans.json

# 2) SILENCE CUT — final_cut.py, LOCKED pad-start 0.0 / pad-end 0.3 (transcription-driven, nothing on top)
# NB (CEO 2026-06-25, harden #4 gap #3): --crf/--preset are libx264 params — they apply ONLY to this
# FLOW-2 9:16 1080p piece encode (libx264 is fine at 1080p; the no-libx264 ban is for 4K). In FLOW 3
# the silence "cut" is CONCEPTUAL — it is the set of keep-windows applied as trims in the 4K HW render
# (videotoolbox/nvenc); never run final_cut.py as a separate libx264 encode on 4K. The cut CONTRACT
# (break-threshold 0.35, pad 0.0/0.35 on raw word.start/end) is what carries to FLOW 3, NOT these flags.
python3 <video-editing>/src/video/final_cut.py SRC.mp4 \
  --transcription SRC_trans.json --output PIECE_desil.mp4 \
  --pad-start 0.0 --pad-end 0.3 --crf 12 --preset veryfast    # libx264 @1080p only — see NB above

# 3) SAM3 vertical + 4) CENTERING-3 (one-euro), z0 full height:
#    sample face center_x every 0.10s via SAM3 :8100 ; one-euro(mincut=1.8, beta=0.25) ;
#    interpolate to 30 fps ; ffmpeg sendcmd crop x  (cw=even(1215) ch=even(2160), x clamp 0..3840-cw) ; scale=1080:1920,setsar=1 ; output ProRes
#    -> PIECE_vert.mov   (NO leading trim, NO tail-normalize, NO energy)

# 5) Step-10 audio-pad (audio == video, zero drift)
VD=$(ffprobe -v error -select_streams v:0 -show_entries stream=duration -of csv=p=0 PIECE_vert.mov)
ffmpeg -nostdin -v error -y -i PIECE_vert.mov -af apad -t $VD -c:v copy -c:a pcm_s16le PIECE_proc.mov
```

Combos (concat = stream-copy + PTS-realign; the ONLY non-edit steps allowed):
```bash
# 6a) encode each PIECE_proc.mov -> H.264 ONCE (identical params so concat can stream-copy)
ffmpeg -nostdin -v error -y -i PIECE_proc.mov -c:v h264_videotoolbox -b:v 12M -pix_fmt yuv420p \
  -c:a aac -b:a 256k -r 24000/1001 -movflags +faststart PIECE.mp4
# 6b) per combo: concat-demuxer, video STREAM-COPY, audio AAC
printf "file '%s'\n" HOOK.mp4 BODY.mp4 CTA.mp4 > list.txt
ffmpeg -nostdin -v error -y -f concat -safe 0 -i list.txt -c:v copy -c:a aac -b:a 256k -movflags +faststart tmp.mp4
# 6c) PTS-realign (black-first-frame fix): video start->0, audio capped to video
VS=$(ffprobe -v quiet -select_streams v:0 -show_entries stream=start_time -of csv=p=0 tmp.mp4)
VD=$(ffprobe -v quiet -select_streams v:0 -show_entries stream=duration   -of csv=p=0 tmp.mp4)
ffmpeg -nostdin -v error -y -i tmp.mp4 -itsoffset $VS -i tmp.mp4 -map 0:v -map 1:a -c copy -avoid_negative_ts make_zero -t $VD hookX_bodyY_ctaZ.mp4
```
Verify per output: `start_time` v:0 == a:0 == 0 and `duration` v:0 ≥ a:0 (black-frame), last word present (no clip), centering-3 face cx within a few px of frame center.

## Phase-2 Steps

### Step 7 — Silence removal per piece (cam-A is clock + audio), TAIL-EXACT + LIP-SYNC-SAFE
**TRANSCRIPTION-DRIVEN — NO VAD.** Speech vs silence is defined ONLY by the Parakeet word-level
timestamps: speech = the word spans, silence = the gaps between words. Do **NOT** use VAD /
`vad/generate_timestamps.py` / TEN-VAD / a rough-cut tier — they are removed from this pipeline.
Flow: **Parakeet transcribe cam-A (Step 2) → `video/final_cut.py`** on the source with that
transcription. `final_cut.py` is fully transcription-driven (it reads the `segments`/word
timestamps, keeps each speech span + the pad, and drops the inter-segment gaps).
**LOCKED DEFAULTS (CEO-decided, do not change): `--pad-start 0.0 --pad-end 0.3`** (`--crf 12 --preset veryfast`).
pad-start 0.0 → each piece starts AT its first word (final_cut cuts at `segment.start`, which equals the first `word.start`). pad-end 0.3 → keep up to 0.3 s after the last word (clamped to the source clip end). Apply NOTHING on top of final_cut's output — no leading-silence trim, no tail-normalize, no energy/RMS, no custom pad values. Two non-negotiable rules learned the hard way:
- **TAIL:** run `final_cut.py` on the **SOURCE clip with the SOURCE transcription** (which contains
  the FULL last word). final_cut trims silence at sentence boundaries AFTER the last word
  (`--pad-end 0.3`). A re-transcription of an already-cut clip MISSES the tail → the last word gets
  clipped (we cut "eu começo"). Command:
  `final_cut.py <camA_source> --transcription <camA_SOURCE_trans.json> --output <camA_desil> --pad-start 0.0 --pad-end 0.3 --crf 12 --preset veryfast`
- **LIP-SYNC:** de-silence cam-B with the **EXACT SAME cam-A source transcription** (NOT a cam-B
  transcription, NOT a re-transcription): `final_cut.py <camB_source> --transcription <camA_SOURCE_trans.json> --output <camB_desil> …`.
  Identical segment times → identical frame cuts → cam-B frame-locked to cam-A.
  **VERIFY:** `sync/word_sync.py <camA_desil_trans> <camB_desil_trans>` must report **offset≈0, drift≈0, std≈0**.
  Nonzero ⇒ you used the wrong transcription for cam-B; redo.

### Step 8 — SAM3 vertical reframe (start the API if down)
- **SAM3 SERVICE CONTRACT (CEO 2026-06-25 fold — harden #2 gaps #2/#3/#4; do not make the next worker guess):**
  - **Host/port are configurable — `SAM3_URL` (default `http://192.168.15.14:8100`).** The default host is OUR server; a different host (e.g. `127.0.0.1` when running ON the server) is set via `SAM3_URL`, NOT hardcoded. **Port:** `:8100` is the primary SAM3 worker; a **second worker may run on `:8103`** (isolated on a separate GPU) — either is the same `facebook/sam3` API; use `:8103` when you want an isolated GPU for framing. (Earlier text shows both `:8100` and `:8103`; they are the same service on different workers — pick one via `SAM3_URL`.)
  - Health: `curl -sf $SAM3_URL/health` → `{"status":"ready", "model":"facebook/sam3"}`. If down: `ssh <sam3-host> "cd <video-editing>/apps/sam3-api && docker compose up -d"`, wait ~30 s.
  - **Request — `POST $SAM3_URL/segment`, MULTIPART form, fields `image` + `prompt`. The image part MUST carry content-type `image/jpeg` (or `image/png`).** `curl -F image=@frame.jpg` works because curl infers the mime; a programmatic client (Python `requests`) MUST set it explicitly — `files={'image': ('f.jpg', data, 'image/jpeg')}` — otherwise the server returns **HTTP 400 "Invalid image format. Supported: JPEG, PNG"** (the OpenAPI schema mislabels it `application/octet-stream`; ignore that). `prompt` is the open-vocab string (`head`, `eyes`, `face`).
  - **Response shape:** `{"objects": [ {"confidence": float, "bounding_box": {"x1","y1","x2","y2","width","height","center_x","center_y"}, "relative_position": {...}}, ... ]}`. **Anchor mapping** (seed-name → JSON): for the `head` prompt's chosen object, `crown = bounding_box.y1`, `chin = bounding_box.y2`, `head_h = bounding_box.height`, `head_cx = bounding_box.center_x`; for the `eyes` prompt, `eyes_y = bounding_box.center_y`. Multi-person: pick the head object whose box contains the eyes center, else the most central/prominent (deterministic; NOT identity recognition).
- Vertical crop at zoom Z: `cw=even(1215·(1−Z/100))`, `ch=even(2160·(1−Z/100))`, `x=even(center_cx−cw/2)` (clamp 0..3840−cw); for Z=0 `y=0` (full height); for Z>0 position face ~45% from top (`y=center_cy−0.45·ch`) clamped so `head_top=center_cy−0.65·h` stays inside (`y ≤ head_top−20`). Scale crop → `1080:1920,setsar=1`.

### Step 9 — A/B multicam: AGENT-REASONED beat-grid switching (CEO redesign 2026-06-20; replaces blind A/B)
**The camera switch is THOUGHT about per beat from the content — never a hardcoded alternate table, never the old "2-span" rule.** Audio is ALWAYS Cam A; only the video switches.
- **BEAT GRID (cadence) — applies to PHASE-2 / FLOW-2 (9:16 reels). FLOW 3 (landscape long-form) uses NO MAX_WINDOW instead — see note (CEO 2026-06-25, harden #3 gap #2).** lay change-points every **3–5 s** on the silence-removed (output) timeline, each **snapped to a clause/word boundary** (sentence end, comma, or a trimmed-pause cut) so the switch lands on a speech beat. *Change-on-cut* makes the cut read intentional, not like hiding a mistake. (Strict 3–5 s — do not drift to 6 s.) **In FLOW 3 this 3–5 s cap does NOT apply: per "The editing rules" (`There is NO MAX_WINDOW`), never force a cut mid-sentence — a 7 s unbroken phrase stays a 7 s window; cut only at natural `.!?`/comma boundaries with `MIN_CUT_INTERVAL = 1.0 s`. Use the 3–5 s grid for 9:16 reels, the no-max-window rule for 16:9 long-form.**
- **AGENT-REASONED camera per beat (the pipeline THINKS):** a Claude worker reads each beat's transcript text + recent history and CHOOSES the camera by meaning, emitting a **one-line reason per beat** (logged for audit). Principles:
  - **Cam A (close / better-mic angle):** personal "eu…" statements, emphasis, key claims, punchlines, the answer to a question.
  - **Cam B (wide):** setup/context, transitions ("Então", "Próxima pergunta"), enumerations/lists, opening a new question/topic, audience address (outro/CTA).
- **CAMERA DISTRIBUTION — balance toward ~50/50, BOUNDED by each camera's gate-valid CEILING (CEO 2026-06-22). SUPERSEDES the old "cam-B workhorse / cam-A on emphasis only" bias.** Each camera's *ceiling* = the fraction of beats where that camera passes ALL gates (eyeline ≤ 42, head ≤ 70, headroom ≥ 7, crop ≥ 2688×1512, X-centered) at beat-start (measure via SAM3 on BOTH cameras per beat). Target per camera = `min(ceiling, 50%)`: if the SCARCER camera's ceiling < 50%, use **ALL** its gate-valid beats and the other camera fills the remainder; if BOTH ceilings ≥ 50%, split **50/50**. Distribute the chosen camera **EVENLY across the timeline** (don't bunch one camera). Examples: cam-A ceiling 9% → cam-A 9% / cam-B 91%; ceiling 25% → 25% / 75%; both ≥ 80% → 50/50. A beat valid on only ONE camera is forced to it. The content principles above (Cam A = personal/emphasis, Cam B = setup/lists) now only **guide WHICH valid beats each camera takes** within this 50/50-capped distribution — they no longer bias the overall ratio. (Move/zoom LEVEL stays content-driven per Step 10, independent of which camera the beat lands on.)
  - **Holds are allowed and expected** — keep the same camera across beats while a single thought/emphasis continues (this is what makes it NOT blind alternation). Vary for freshness, but the choice is content-first.
  - Never emit a fixed A/B/A table; the camera depends on WHAT is being said.
- **AUDIO IS ALWAYS CAM-A** (better mic). cam-B = video only, frame-locked (Step 7).
- **Framing (Phase 1 = camera-only):** each beat is a **full-frame** cut from the chosen camera (static within the beat). **Zoom / crop / reframe is a SEPARATE later phase** (Phase 2) — do NOT add per-beat zoom/crop or change source resolution until that phase is greenlit. **Fluid head-follow (Step 10b) is OPT-IN — never a default, never a hard-gate.**
- **RENDER DISCIPLINE — spans MUST be strictly contiguous, non-overlapping, monotonic** (a real bug class: correct transcript → duplicated word on every cut). A camera-change split *within continuous speech* is a CLEAN cut: `prev_piece.end = next_word.start` (no pad) so `start[i+1] == end[i]`. **Apply the 0.30 s pad-end ONLY at real silence cuts** (word-gap >0.30) and the final end — NEVER at a camera-change split (that pad reaches past the next word's start and replays it). **Assert before render:** `end[i] <= start[i+1]` for all pieces, and every kept word is covered by **exactly one** piece. **Assert after render:** re-transcribe and confirm **0 NEW consecutive-duplicate words** vs the source (real-speech repetitions like anaphora/"tchau, tchau" are allowed; edit-induced ones are not).
- **DECLICK FADE — kill the splice click at silence/false-start cuts.** Apply a short **~4 ms fade-OUT on the outgoing segment edge + ~4 ms fade-IN on the incoming segment edge** (per-segment `afade=t=out:st=<dur-0.004>:d=0.004` and `afade=t=in:st=0:d=0.004`), **ONLY at silence / false-start cuts — NEVER at camera-change splits** (those rejoin the *same Cam-A sample* and are already seamless; fading them would notch real speech). The fades **ramp amplitude in place → segment durations are UNCHANGED**, so the silence trim is preserved exactly and there is **no A/V shift**.
  - **WHY:** audio is segmented per piece and concatenated; at a silence cut the splice joins two *non-adjacent* source samples. Parakeet's `word.end` is **conservative** (the vowel sustains past the labeled end), so the 0.30 s pad-end frequently lands on **still-voiced, high-amplitude** audio, concatenated to the next word's near-zero onset → a **non-zero-crossing step = a few-ms click** (heard at some cuts, not all — depends on the amplitude at the pad point). The fade ramps both edges to ~0 so the step vanishes.
  - **VERIFY:** re-measure the per-sample step at each silence/false-start splice — it must **ramp to ~0** (proven run: worst click 16132 → 76; all audible silence-cut clicks <400). Also confirm **A/V locked** (Δ <1 frame), **audio still Cam A**, and **0 new duplicate words**.
- **KNOWN LIMITATION (deferred by CEO 2026-06-20):** each **camera switch** inserts a **~17 ms audio gap** → a faint tic at switch points. Cause: audio is segmented per camera piece and `concat=v=1:a=1` reconciles the **frame-grid video duration vs sample-exact audio duration by padding silence** at the boundary. The declick does NOT catch it (camera splits are skipped, assumed seamless — but they are not under `concat v=1:a=1`). Measured: the highest-isolation discontinuities in the render land exactly on camera-switch times (isolation 45×–461×). **Not fixed (CEO deferred).** Fix direction when revisited: use a **single continuous Cam-A audio track / proper NLE timeline — do NOT segment the audio at camera switches** (switch video only); audio is segmented only at silence cuts (with declick).
- **Validate:** cadence (every change in 3–5 s), beat-alignment (cut within ~120 ms of a word/clause boundary), content-driven variety (holds present, not pure alternation), A/V lock, audio = Cam A, 100% content coverage; post the per-beat decision log for the CEO to audit.

### Step 10 — Zoom by HEAD-HEIGHT RATIO, SAM3-measured, VARIED + X-centered + Y by the AGGRESSIVE headroom→eyeline curve (CEO 2026-06-20, Y-curve 2026-06-21)
**Zoom is defined by HEAD-HEIGHT RATIO = vertical head height (crown→chin) ÷ screen height** — NOT a crop-factor (z25/z50). **SAM3-TRUTH ANCHORS (CEO 2026-06-24, folded — replaces the old face-box heuristic).** Prompt SAM3 for **`head`** and **`eyes`** and read the anchors DIRECTLY off the returned boxes: `crown = head.y1` (real top of hair/skull), `chin = head.y2`, `head_h = head.height`, `eyeline = eyes.center_y`. We do **NOT** derive crown / head-height / eyeline from the `face` box anymore — the old guesses (`crown = face.y1 − K·fh`, `head_h ≈ 1.25·fh`, `eyes_y = face.y1 + 0.35·fh`) are **DELETED**: they drifted from the real head frame-to-frame and caused both hair-clipping (crown too low) and over-rejection of good footage (crown too high). SAM3's segmentation is the SOURCE OF TRUTH; we trust it blindly (if SAM3 hands back a wrong box, the beat is framed to that box — that is SAM3's error, not the framing flow's). For a target ratio R, `crop_h = head_h / R`, `crop_w = crop_h · 16/9` (landscape), from the **full original 4K** (per the source-resolution discipline) → scale to delivery at export.
> **VERTICAL-FRAMING PARAMETERS — CEO-DECIDED (CEO 2026-06-21, folded from the prior pending note).** `MAX_HEAD = 0.70` (head-height cap) and `HEADROOM_FLOOR = 0.07` (minimum crown-to-top-of-frame gap = **7% of the output frame height**). These are the CEO's chosen values, now folded as real hard rules (no longer TBD). `HEADROOM_FLOOR` **SUBORDINATES** the eyeline target — the eyeline rises toward 33% only as far as it can without dropping headroom below 7%. The **RESOLUTION FLOOR = 1.4×FHD = 2688×1512** (CEO-decided 2026-06-22 — RAISED from 1.3×FHD = 2496×1404; history 1.2×/2304×1296 → 1.3×/2496×1404 → 1.4×/2688×1512) — see HARD RULE 2.
- **HARD RULE 1 — MAX head-height cap `MAX_HEAD = 0.70` (CEO-decided 2026-06-21).** Never tighter than `R = 0.70` (head fills ≤ 70% of screen height).
- **HARD RULE 2 — RESOLUTION FLOOR = 1.4×FHD = 2688×1512 (CEO-decided 2026-06-22, RAISED from 1.3×FHD = 2496×1404; original 1920×1080 plain FHD looked soft, and the 1.3× floor crops were the soft tight close-ups → 1.4× carries ~7.7% more linear px so punch-ins stay sharp at the 4K export).** Every crop region in the 4K source keeps **≥ 2688×1512 NATIVE** pixels (`crop_w ≥ 2688` AND `crop_h ≥ 1512`). This caps the max zoom (`R ≤ head_h/1512`): a crop that would fall below 2688×1512 is REJECTED — the shot stays wider but sharp. (Effect: a small-head WIDE camera can only punch in until its crop hits 1512 tall — e.g. head_h≈700 ⇒ max head ≈ 0.46 — so the wide cam never upscales a tiny soft region.)
- **HARD RULE 2b — HEADROOM FLOOR `HEADROOM_FLOOR = 0.07` (7%) (CEO-decided 2026-06-21).** Every crop must leave **≥ 7% of the frame height as headroom above the crown** (gap between crown and top of frame): `(crown − cy) / crop_h ≥ 0.07`. **A crop with < 7% headroom is REJECTED.** The eyeline target is **SUBORDINATE** to this floor (see the Y-placement below). If a level cannot satisfy 7% headroom with the head still in frame, reject that level — go wider.
- **HARD RULE 4 — MINIMUM CROP-CHANGE = 10% RELATIVE TO THE CURRENT CROP (CEO-decided 2026-06-22, picked from the HTML visualizer).** A same-camera crop/zoom STEP between consecutive shots must change the crop by **≥ 10% relative to the CURRENT crop** — i.e. the zoom RATIO, **`new_crop_w ≤ 0.90·current_crop_w`** to push in (or `≥ current_crop_w / 0.90` to pull out). **Below 10% relative → HOLD the crop (no change).** A sub-10% step reads as accidental jitter / a bug, not an intentional move, so we do **not** ship it. **The % is RELATIVE to the current crop (the zoom ratio), NOT % of the original 4K frame** — absolute-frame % is perceptually inconsistent (tiny when wide, huge when already tight). **QUANTIZE same-camera zoom steps to ≥ 10% relative:** snap every planned same-cam step to either a ≥10%-relative change or to NO change — this **replaces the old mechanical sub-10% arc steps** (de-mechanizes the zoom). A **camera SWITCH** may reset the zoom freely (this rule governs only **same-camera** consecutive shots).
- **VARIED, content-driven levels (NOT max-on-both).** Per camera run, choose R **by what is being said** (agent-reasoned, same spirit as the camera choice), within `[natural-wide, MAX_HEAD]` where natural-wide = `head_h/2160` (the full-frame ratio — can't go wider than the source) and `MAX_HEAD` = 0.70 (CEO-decided, above). Wider (smaller head) for setup / new questions / lists / transitions / outro; tighter (toward `MAX_HEAD`) for emphasis / key claims / punchlines / personal beats. Zoom LEVEL varies by content on whichever camera the beat lands on — **camera SELECTION is the 50/50-capped balance rule (Step 9), not a workhorse bias.** **Never put both cameras at max** — vary.
- **X — FACE CENTERED horizontally on every zoom.** The crop **moves toward the face** in X: `cx = face.center_x − crop_w/2`, clamped to frame. If the subject moves too much within a run for a fixed crop-x to stay centered, drop that run to the **widest gate-passing crop** (the reset framing below — still subject to the eyeline ≤ 42% gate, NOT a raw full-frame) rather than ship an off-center zoom (fluid follow is OPT-IN, Step 10b).
- **Y — SHOT-SIZE-DEPENDENT placement by the AGGRESSIVE headroom→eyeline CURVE (CEO-picked 2026-06-21). NOT face-centered in Y, NOT a hardcoded eyes number — DERIVED per run from SAM3.** The vertical placement changes with shot size (cinematography convention): at WIDE the face is **headroom-anchored** (generous space above the crown); as the shot tightens, the **headroom shrinks and the eyeline rises toward the upper third (~33%)** — **bounded by `HEADROOM_FLOOR = 7%`**: the eyeline only rises as far as it can while keeping ≥ 7% headroom above the crown, so the crown never kisses the top / leaves frame on natural head movement. *(Previously the curve let the crown reach ~0% headroom at the tight end; the 7% `HEADROOM_FLOOR` now supersedes that.)*
  - SAM3-TRUTH anchors (MEASURED off the SAM3 boxes, NOT guessed from the face box): `crown = head.y1`, `chin = head.y2`, `head_h = head.height` (SAM3 `head` box); `eyes_y = eyes.center_y` (SAM3 `eyes` box). *(The legacy `eyes_y = face.y1+0.35·fh` / `crown = face.y1−0.25·fh` derivations are DELETED — see Step 10 intro and HEAD-SAFE ANCHORS.)*
  - **The CURVE (AGGRESSIVE, k = 0.5):** `t = clamp((R − 0.40) / (0.70 − 0.40), 0, 1)`; `headroom_frac = H_WIDE · (1 − t^0.5)` with **`H_WIDE = 0.20`** (fraction of crop height; this is the approved eyeline-shape curve — eyes ~32–40% across head 40→`MAX_HEAD`). *(The `0.70` here is the curve's SHAPE asymptote constant, NOT the head cap — the head cap is `MAX_HEAD` = 0.70 (CEO-decided). The empirical ~32–40% eyes range was measured with the old 0.70 cap and is to be re-validated for the chosen `MAX_HEAD` + `HEADROOM_FLOOR` in the comparison HTML; the headroom floor will bound how high the eyeline rises.)* Then `cy = round(crown − headroom_frac · crop_h)`, **clamped** to `[0, 2160 − crop_h]` and head-safe (`cy ≤ crown` so the crown stays in frame; `cy + crop_h ≥ chin` so the chin stays in frame). The resulting eyeline is `eyes_line% = (eyes_y − cy) / crop_h`. (For R below 0.40 — wider than head 40%, e.g. the natural-wide level — `t` clamps to 0 → full `H_WIDE` headroom plateau.)
  - **CANONICAL OPERATIONAL Y-PLACEMENT — implement THIS single formula (CEO 2026-06-25 fold, harden #2 gap #5; resolves the curve-vs-worked-example contradiction).** Compute `cy = clamp(min(eyes_y − 0.36·crop_h, crown − 0.07·crop_h), 0, 2160 − crop_h)` (the Flow-3 worked-example). It encodes BOTH targets and takes whichever BINDS: eyes at **36%** when feasible, but never closer than **`HEADROOM_FLOOR = 7%`** above the crown — so at tight shots the headroom floor (`crown − 0.07·ch`) wins and the eyeline sits slightly lower, exactly as intended. **This SUPERSEDES the `H_WIDE`/`t^0.5` AGGRESSIVE curve above for IMPLEMENTATION** (that curve predates the 7% floor and lets the crown kiss the top at the tight end — do NOT implement it; it remains only as the shape-lineage/rationale). Both flows derive Y this one way — never two.
  - **ENFORCE THE HEADROOM FLOOR (precedence over eyeline) (CEO 2026-06-21):** the valid `cy` band is `eyes_y − 0.42·crop_h ≤ cy ≤ crown − 0.07·crop_h` (lower bound = eyeline ≤ 42%; upper bound = headroom ≥ 7%). Place `cy` at the curve/33%-target value **clamped into this band** (and into head-safe + frame). The 7% headroom wins over the 33% eyeline goal — at tighter heads the eyeline lands lower than 33% to preserve headroom. **Joint feasibility:** with SAM3-truth, `eyes_y` and `crown` are MEASURED per beat (no fixed `0.48·R` ratio), so evaluate the band on the real values. Empirically the eyeline-≤42% / headroom-≥7% pair stays satisfiable up to head ≈ 0.70–0.73, so the `MAX_HEAD = 0.70` cap is the binding limit in practice. **If the band is ever empty for the measured head, go wider — never drop a beat that has data.**
  - **GOAL:** eyeline as close to **33% (upper third)** as possible.
  - **HARD GATE — eyeline ≤ 42%, evaluated (CEO 2026-06-22, relaxed from 40%), at the BEAT START, for ALL move types with NO exemption (CEO 2026-06-21). The gate applies to every move, RESETS AND FULL-FRAME INCLUDED — there is NO untouched-original exemption.** Any crop whose **beat-start eyeline exceeds 42% is NOT a usable framing**: REJECT it and fall back to a level that keeps the (beat-start) eyeline ≤ 42%.
    - **A RESET / wide / "drop to full-frame" move resolves to the WIDEST framing that still keeps eyeline ≤ 42% — NOT a raw full-frame when full-frame exceeds 42%.** Compute the widest gate-passing crop directly: `crop_h_reset = min(2160, (2160 − eyes_y) / (1 − EYELINE_GATE))` where `EYELINE_GATE = 0.42` (so the denominator is `0.58` — it is `1 − the eyeline gate`, NOT a magic number; if the gate changes, this tracks it) — the largest crop whose top-of-frame placement `cy = 2160 − crop_h` still lands the eyeline at exactly the gate (42%); `crop_w = crop_h · 16/9`, X-centered. If full-frame already passes (`eyes_y / 2160 ≤ 0.42` — e.g. a high-sitting / small-head wide camera) then the reset stays full-frame; otherwise (e.g. a **cam-A that sits low**, full-frame eyeline ~46–51%) the reset becomes the slightly-tighter widest gate-passing crop (~head 55–60%) — this is correct per the rule, not a defect.
    - **If even the widest possible gate-passing crop cannot reach ≤ 42%** (geometry — head so large/low that no in-frame, head-safe, FHD crop satisfies it at the beat start), **FLAG it — do NOT silently ship a > 42% beat.** Two failure ends: the extreme-wide (small/high head) and the close-low camera (large/low head). When a close-cam beat-start sits below the line even at MAX_HEAD, switching that beat's video to the wider camera (audio stays Cam A) is a valid fallback.
    - **EVALUATE AT 3 SAMPLE POINTS — START / MIDDLE / END — COMBINE BY SIMPLE AVERAGING (CEO 2026-06-22; replaces the single beat-start sample).** One frame can misread where the head is (the subject moves during a beat), so SAM3 the **`head` AND `eyes` boxes** at **3 points**: `start + 0.15s`, the **midpoint**, and `end − 0.15s`. **AVERAGE (mean, or median) the 3 head boxes into ONE head box AND the 3 eyes boxes into ONE eyes box** — `center_x, center_y, width, height` each = the mean — and feed those single averaged boxes into the **UNCHANGED** single-frame crop/gate logic (`crown=head.y1 / chin=head.y2 / head_h=head.height / eyes_y=eyes.center_y`, then the same curve / gates). The averaged boxes are the ONLY change: better-estimated anchors instead of one possibly-misread frame. **Do NOT add any other constraint** — no "valid across all 3", no intersection-of-bands, no per-sample re-checking. **(VALIDITY: a per-frame sample counts only if SAM3 returns BOTH a `head` and an `eyes` box — see the SAM3-TRUTH VALIDITY + REJECTION rule under HEAD-SAFE ANCHORS; average only the valid samples. If NO sample in the beat has both, the beat has no data on that camera → use the other camera; see the rejection rule.)** **Cost: SAM3 `head`+`eyes` × 3 samples = 6 calls per beat per camera (was 3 `face` calls) — accepted (on a short/isolated run SAM3 inference is actually the larger half of framing — see PER-STEP TIMING's scale-dependent note; on long production runs extraction can dominate).** Still NOT dense per-frame sampling — just 3 anchors averaged. (The reset-applies-to-all-moves rule, commit `daf88c6`, still holds.)
    - **BATCHED FRAME EXTRACTION — HW DECODE (CEO 2026-06-24, folded; measured).** Extract the 3-sample frames with **`-hwaccel videotoolbox`**, concurrency **≤ 8** — HW decode (offloads to the media engine), NOT 14-way software decode (which saturates the CPU cores → thrash). **Measured 4.7×** (per-cam frame extraction ~22.7 min → ~4.8 min on the 06-11 4K-HEVC source), frame-accurate, zero extra disk. *(Rejected alternative: pre-transcoding the source to all-intra ProRes was measured SLOWER than this AND cost ~130 GB — the transcode still decodes the whole source once and ProRes extraction is not free.)*
- **Center is FIXED per run** (the run's median face) — crop SIZE/level steps between runs at speech boundaries (hard step), not a continuous pan.
- **VALID-CROP RULE SET (a crop must satisfy ALL, CEO 2026-06-21):** `eyeline ≤ 42%` AND `head ≤ MAX_HEAD (0.70)` AND `headroom ≥ HEADROOM_FLOOR (7%)` AND `crop ≥ 2688×1512` (1.4×FHD) AND `X-centered (face_cx ≈ 50%)`. If a level can't satisfy all, go wider (or switch camera); never ship a crop that violates any.
- **VERIFY GATE (per run; SAM3-TRUTH, CEO 2026-06-24; per-frame vs anchor split clarified 2026-06-25, harden #3 gap #3):** two distinct checks — do NOT conflate them:
  - **(per RENDERED FRAME, at START / MID / END) — HARD gate = HEAD FULLY IN FRAME, no clip:** re-run SAM3 (`head`+`eyes`) on the ACTUAL rendered crop and confirm `crown = head.y1` is below the crop top and `chin = head.y2` is above the crop bottom — the head is never decapitated. This is the only PER-FRAME hard gate. **The instantaneous rendered eyeline LEGITIMATELY drifts (e.g. to 43–47%) as the subject moves within a 2–4 s beat — that is the allowed mid-take drift; do NOT fail it and do NOT correct it.**
  - **(on the AVERAGED DECISION ANCHORS — the 3-sample mean the crop was computed from) — framing gates:** `eyeline ≤ 42% AND near 36%` (`eyes.center_y`), `headroom ≥ 7%`, `X-centered` (head center_x ≈ 50%), `crop ≥ 2688×1512` (1.4×FHD), `R ≤ MAX_HEAD (0.70)`. These are checked against the averaged anchors, NOT per rendered frame (per-frame would falsely fail a correct edit because of allowed drift).
  - **Also verify the validity/rejection contract:** every SHIPPED beat had at least one VALID sample (SAM3 returned head+eyes) and was cropped-to-valid — **flag any beat shipped without SAM3 data** (it should have been routed to the other camera, never shipped on a guess); and confirm no beat was DROPPED while it had data. (Flag any beat whose start cannot meet the rule set; do NOT re-check or correct mid-take drift.) Log per run: head% chosen, native crop size + FHD ✓, X-centered ✓, **measured eyeline% + headroom% (from the SAM3 boxes)**, valid-samples k/3 per camera, content reason.
- **DECISION DEBUG LOG — REQUIRED ARTIFACT, emitted with EVERY render (CEO 2026-06-22).** Every edit MUST write a human-readable per-beat decision log (HTML or txt) next to the output, for debugging. It must contain: a SUMMARY (source pair + durations + sync, the active rule values, beat count, cam-A/cam-B split, output duration, total dead-air removed, eyeline distribution, move distribution, a one-line diagnosis); a SILENCE/DEAD-AIR CUTS table (each cut: source out→in time + seconds removed); and a PER-BEAT table — output timestamp + source timestamp, content type, CAMERA chosen, MOVE, head% / eyeline% / headroom% / X-center%, native crop rect, WHY this camera was chosen, and **WHY the other camera was rejected (the binding rule: head>cap / eyeline>gate / headroom<floor / crop<res-floor / out-of-sync-range)**, plus the transcript text. (Requires SAM3 of BOTH cameras at each beat-start so the rejected camera's binding rule is recorded.)
- **PER-STEP TIMING BREAKDOWN — REQUIRED ARTIFACT, emitted with EVERY edit (CEO 2026-06-22; ATOMIC-TIMER contract added 2026-06-25).** Capture WALL-CLOCK per pipeline step and emit it as a clean breakdown in the manifest AND the decision-debug (and report it). Time these individually: **(a)** audio decode + GCC-PHAT sync (+ sign-validate), **(b)** transcription, **(c)** segment+match / EDL build, **(d) SAM3 framing pass (BOTH cameras — usually the #1 stage overall) — SPLIT into two sub-timers that MUST be reported separately: (d1) frame EXTRACTION (ffmpeg HW-decode of the sample frames) and (d2) SAM3 inference (probe seconds + calls/s + its share of d). WHICH of d1/d2 dominates is SCALE-DEPENDENT, so always report both and mark the larger (CEO 2026-06-25, harden #3 gap #6): on a SHORT/isolated run (e.g. the 3-min harness, ~180 frames) SAM3 inference dominates (~80% of d — measured 161–181 s SAM3 vs 43–49 s extraction across runs #2/#3 on an isolated GPU); on a LONG full-length production run (thousands of frames, often a contended/HEVC-bound decoder) extraction can dominate. Do NOT assert one as "the" bottleneck — measure d1 vs d2 per run.**, **(e)** 4K render (per-segment hw decode+encode — the HEVC-decode-bound stage), **(f)** mux/finalize (concat-demuxer + PTS realign), **(g)** TOTAL. Mark the BOTTLENECK step explicitly.
  - **ATOMIC, SELF-RECORDED — each stage stamps its OWN start/end at run time, into the timing artifact, as it runs (NOT reconstructed after the fact from file mtimes, log scraping, or wall-clock between unrelated steps).** A stage's timer is written by that stage's own code, around its own work.
  - **CACHED / SKIPPED ≠ ZERO (the bug this rule exists for).** If a stage is skipped because its output already exists (e.g. a re-launched `process.sh` finds `boxes.json` already computed, so framing does no work), it MUST record `{cached: true, work_wall_s: 0, prior_measured_wall_s: <the real framing time from the run that produced the cache, or null>}` — it must NOT report `~0 s` as if framing were free. A framing stage that emits `0.09 s` because boxes were cached is a FALSE timing, exactly the 06-24 gap: the dominant stage looked free.
  - **CLEANLINESS FLAG — REQUIRED.** The artifact records whether the run was `clean` (one uncontended pass, `restarts: 0`, no known GPU contention) or `contended` (record `restarts: <n>` and any known competing GPU job, e.g. a voice demo holding VRAM). A `contended`/restarted wall-clock is NEVER reported as a representative/clean stage time — label it. (On 06-24 the framing stage took ~47 min ELAPSED but that was GPU-contended + 5 in-loop restarts AND partly reconstructed from mtimes — it is NOT a clean framing number; a representative framing time needs one `clean` instrumented pass.)
  - **PROVENANCE per stage:** `measured` (atomic run-time timer) vs `reconstructed` (mtime/inference). Any `reconstructed` value MUST be flagged as such and MUST NOT be presented as measured.
  - Reference points (label provenance + cleanliness): C4789/C4843 fresh-footage run framing ≈ 30.5 min (bottleneck) ≫ render, transcription 25 s, sync/EDL minutes [measured]; 06-24 live-interativa setup 9.92 s / EDL 0.04 s / render 479 s / finish 244 s [measured], framing ~47 min [reconstructed + contended — NOT clean].
- *(Legacy `discover_framings` measured (zoom%,position) framings the same way — head-not-cut/face-visible via SAM3; this Step supersedes its crop-factor labels with the head-height-ratio metric + the two hard rules.)*

#### MOVE VOCABULARY — INSTANT or STATIC only; NO within-shot motion (CEO-approved 2026-06-20)
**HARD RULE 3 — framing changes are INSTANT (a step / cut) or STATIC. NO motion during a shot. Slow / creeping push-in & pull-out (gradual zoom while the shot holds) are BANNED.** (The CEO rejected slow zooms outright.) The approved move set, chosen **content-driven / agent-reasoned** (same spirit as the camera choice):
- **STATIC hold** — one head-height level held for the shot (the default; use on short beats and steady content).
- **STEP crop IN / OUT** — an INSTANT reframe to a tighter/wider level, then hold (use stepping IN on a new answer / to add intensity).
- **MULTI-STEP LADDER** — step level instantly, hold, step again (use to **step UP to a punchline/emphasis**, or **one step per item** through a list/enumeration).
- **RESET-to-wide** — instant jump back to the **widest gate-passing level** (use on a **new question / topic / the outro**). NOTE: the reset is **still subject to the eyeline ≤ 42% gate** — it is the widest framing that keeps eyeline ≤ 42%, NOT a raw untouched full-frame (see the HARD GATE above). On a low-sitting cam-A this means a reset is a slightly-tighter wide crop, not full-frame.
- **Agent-reasoned CAMERA SWITCHING** (Step 9) — the other instant change.
- **Steps land ON the silence-cut beats** (change-on-cut → reads intentional). Within a run, change the static level only at a piece boundary (a trimmed-pause beat); never ramp mid-shot. Map: ladder-up → punchline/admission; reset-wide → new question/outro (widest gate-passing crop, eyeline ≤ 42%); step-in → new answer/intensity; static → short beats; widest-gate-passing-crop → if the subject moved too much to keep a tight crop centered (NOT raw full-frame — the eyeline ≤ 42% gate applies to every move).
- **Implementation (the jitter fix):** each shot is a **STATIC crop** at its level (or an instant step between static crops at beats). Compute the crop SIZE from a **STABLE per-run head reference** (run-median `head_h`) — NOT the per-frame head height (that caused size jitter/breathing) — and center on the per-run / per-piece median face (smoothed). Render **clean from the full-original 4K** (input-seek the originals; no cropped/upscaled intermediate) → `crop=cw:ch:cx:cy, scale=<export>`. A/V stays locked via the single concat (`concat=v=1:a=1`), audio LOCKED Cam A, with the Step-9 declick at silence cuts + contiguity. Verified to generalize on fresh footage.

### Step 10b — Cam-A centering: STATIC baseline + OPT-IN fluid head-follow
- **OPT-IN (CEO directive):** the BASELINE is a **static center per cam-A run** (the run's median face; re-center only on a camera switch; no per-frame drift). The **fluid head-follow / "Fluid-Gimbal" below is OPT-IN — run it ONLY when the CEO explicitly asks for head-following.** It is **never the default and never a hard-gate**; its absence is NOT a defect and NOT an auto-reject. Everything below describes HOW to do the follow WHEN requested.
- **The RULE (per cam-A run):** a STATIC center holds ONE crop-x (the run's median face). As the head sways it slides off-center by ~(sway/2). Static is imperceptible only while the head's **sustained off-center stays ≤ ±40 px output** (1080-wide frame). Above that the head visibly drifts toward the edge → use a **FLUID follow**.
- **Measured on the accepted run** (output px, center=540): `hook1 camA` range 293 / sustained ±47; `body run3` 339 / ±80; `body run5` 481 / ±111; `cta1 camA` 346 / ±89 → **all four exceed ±40 → all fluid.** A person talking to camera essentially always exceeds it.
- **The follow method (PROVEN):** dense SAM3 head sample across the run → **one-euro filter** on `center_x` → interpolate to **30 fps** → ffmpeg **`sendcmd` crop x** (z0, full height `cw=even(1215), ch=even(2160)`, clamp x to 0..3840−cw) → `scale=1080:1920,setsar=1`. The one-euro filter **follows the slow drift** (head stays centered) while **dropping high-frequency sway** (the jitter source). NOT a plain moving average — see the LAG failure mode below.
- **STANDARD centering config = "centering 3" (CEO-chosen default):** sample **@0.1 s (10×/sec)**, `mincut=1.8, beta=0.25`, 30 fps interp. This **pins the head to DEAD CENTER even on fast moves** — measured vs a 0.1 s ground-truth: **off-center avg 0.7 px, max 5 px**. Use this for every cam-A run by default.
  - *Why so aggressive:* the CEO wants the head locked to center. The earlier looser config (`@0.3 s, mc=0.3, b=0.25→0.05`) drifts **avg ~7 px, up to ~78 px on FAST head moves** (it samples too slowly to catch them) — he rejected that as "not centered enough." The 0.1 s sampling is what catches fast moves; the high `mincut` makes the crop track tightly.
  - *Tradeoff (accepted):* tighter centering = the crop **chases every head movement** → snappier / less glidey, peak crop motion rises (~19 → ~28 px/frame). On the accepted footage this reads clean, but **VERIFY no objectionable micro-jitter** on each new run (it's the cost of pinning to center). If a run looks jittery, the dial is `mincut`/sample-rate — lower them for more glide, but the standard is centering 3.
- **When requested, fluid is movement-proportional:** the filter adds crop motion *in proportion to* head movement — a near-still head → ~0 motion (hook **3 px/frame** vs body **10–11 px/frame**) — so a requested fluid pass adds no pointless micro-jitter on calm windows. **But it is OPT-IN, not the default** (CEO directive: head-following is something he can ASK for, never done by default, never a hard-gate). **Default = static center per cam-A run.**
- **cam-B stays static + measured zoom (Step 10):** cam-B is the WIDE shot — head already small and roughly centered; its punch-in is the discovered ZOOM step, not a follow. Fluid-track **cam-A only**.

### Step 11 — Render + concat (the `multicam_render_v2` mechanism, adapted to vertical)
- Per piece, ONE `filter_complex`: **cam-B windows** → static `[1:v]trim=s:e,setpts=PTS-STARTPTS,crop=cw:ch:cx:cy,scale=1080:1920,setsar=1[vN]`; **cam-A windows** → **static center crop by default** (`crop=cw:ch:cx:cy` at the run's median face); use the **fluid `sendcmd` crop** from Step 10b ONLY when the CEO requested head-follow (render each cam-A run to a ProRes follow-clip first, then `trim`/concat it in as `[k:v]…setsar=1[vN]`). `concat=n=…:v=1:a=0[v]`; **`-map [v] -map 0:a`** (audio always cam-A). Encode ProRes per piece. (setsar=1 on EVERY segment incl. the fluid clips, or concat fails on SAR mismatch.)
- **Step-10 audio-pad** each piece: probe video dur, `-af apad -t <vdur> -c:v copy` so audio==video (zero A/V drift).
- Concat pieces with the **concat FILTER** (sample-accurate audio, no AAC drift): `[..v]concat=n=3:v=1:a=0[v];[..a]concat=n=3:v=0:a=1[a]`; H.264 IG export `h264_videotoolbox -b:v 12M -c:a aac -b:a 256k -movflags +faststart`. NEVER `-c copy` concat separately-encoded HEVC (green frames); NEVER libx264 for 4K.
- **BLACK FIRST FRAME — MANDATORY PTS realign after concat** (also the fast 200-combo path: encode each piece to H.264 once, then concat-demuxer `-c:v copy`). Concat / `-ss` / `-c copy` leaves the **video stream starting 10–40 ms AFTER audio** (`start_time` v:0 > a:0) → players show a **black frame** where there's audio but no video at the head. ALSO ensure **video duration ≥ audio** (a longer audio tail black-pads the end). Fix every output (it's `-c copy`, instant) — shift audio to the video start AND cap to the video duration:
  ```bash
  VS=$(ffprobe -v quiet -select_streams v:0 -show_entries stream=start_time -of csv=p=0 in.mp4)
  VD=$(ffprobe -v quiet -select_streams v:0 -show_entries stream=duration   -of csv=p=0 in.mp4)
  ffmpeg -y -i in.mp4 -itsoffset $VS -i in.mp4 -map 0:v -map 1:a -c copy -avoid_negative_ts make_zero -t $VD out.mp4
  ```
  Verify: `start_time` v:0 == a:0 == 0, and `duration` v:0 ≥ a:0. Do it for EVERY concat output, not just ones that "look wrong" (the first frame extracted with `-frames:v 1` is the first *video* frame = non-black; the black is what a player shows at t=0). Documented across the pipeline skills (segment-video-prores-v2, combine-ad-variations, tight-cut, sync-cameras, multicam-segment-matching).

## Phase-2 Verify (hard — verified on the accepted run)
- **Lip-sync:** `word_sync` cam-A↔cam-B **offset≈0, drift≈0**; spot-check a cam-B-shot frame's mouth vs the cam-A audio.
- **cam-A centered + z0 (static baseline):** sample several frames per cam-A run — face cx stays near center, cam-A face is BASE size (not zoomed/over-cropped), **no visible per-frame drift**. (Only IF the CEO requested fluid head-follow: additionally confirm centering-3 holds cx within ~±5 px even on fast moves with no objectionable micro-jitter.)
- **Head-safe on REAL 9:16 frames** at EVERY zoom step (SAM3 `face_top` inside crop, no cut).
- **No JITTER, no LAG:** cam-B = static center per run (zoom steps land on speech boundaries); cam-A = static center per run by default — no per-frame creep, re-center only on a switch. **Absence of fluid head-follow is NOT a defect and NEVER a hard-gate.** Only IF the CEO requested head-follow: additionally verify it keeps the head centered without recenter-snapping OR sliding off (centered AND smooth).
- **Vertical 1080×1920**, **audio cam-A only**, **A/V locked** (video-dur==audio-dur), clean joins, **no leading silence**, **FULL final sentence** (last word present — re-transcribe the tail and confirm).
- **TIMING ARTIFACT GATE (measurable — added 2026-06-25):** the per-step timing artifact exists next to the output AND **passes all of**: (1) every stage **(a)–(g)** has a wall-clock value with `provenance: measured` (run-time atomic timer) — any `reconstructed` value FAILS the gate unless the stage genuinely did no work; (2) the framing stage reports the **(d1) extraction vs (d2) SAM3** split, not a single lumped number; (3) a stage that did real work reports a **non-zero** `work_wall_s`, and a `cached:true` stage reports `work_wall_s:0` WITH `prior_measured_wall_s` (a framing stage reporting ≈0 s while NOT marked `cached` FAILS — that is the 06-24 false-timing bug); (4) the **cleanliness flag** is present (`clean` with `restarts:0`, or `contended` with a restart count / named contender); (5) the **BOTTLENECK** stage is named. A run whose timing artifact misses any of these is NOT shippable as a clean reference time.

## Phase-2 Failure modes (the CEO caught every one of these — fold them)
- **Whisper instead of Parakeet** → hallucinated a "Gravando" loop that HID a real take (CTA 4 reported "missing" when it was recorded). NEVER Whisper; if Parakeet/GPU down, restart it.
- **Hardcoded zoom levels** → over-cropped cam-A and cut his head. Use `discover_framings` MEASURED gates; a close cam-A = **z0**, zoom only where measured room (cam-B).
- **Raw per-frame dynamic crop** (`render_vertical`'s stock sendcmd: 1 fps sample + window=3 average) → the frame **creeps/jitters/steps** to recenter mid-take — the CEO HATES this. Do NOT ship it raw. The FIX is the **one-euro fluid follow** (Step 10b) at the **standard centering-3 config** (sample @0.1 s, mc=1.8, b=0.25, 30 fps interp) → head pinned to dead center, no creep. cam-B stays static.
- **Naive HEAVY smoothing LAGS the head** → if you "fix" jitter by smoothing crop_x harder (long moving average), the crop can't keep up with a fast head and the subject **slides ±200 px off-center** — WORSE than static, looks like the person drifting around the frame. More smoothing is the wrong lever. Use the **one-euro filter** (adaptive: follows when the head moves fast, smooths when it's still) — that's what hits "centered AND smooth." (Hit this on the real run before switching to one-euro.)
- **cam-B de-silenced with a different/re-transcription** → **lip-sync drift** on cam-B shots (we shipped a 36 ms divergence). Use the SAME cam-A source transcription for both; verify `word_sync` offset=0.
- **final_cut on a re-transcribed/cut clip trimmed the last word** → sentence clipped. De-silence from the SOURCE with the SOURCE transcription; trim silence AFTER the last word (`--pad-end 0.3`).
- **Trusting the 16:9 gate blindly** for a 9:16 reel → it's a PROXY; verify head-safety on REAL 9:16 frames and drop levels that crop.
- **High-movement window on a static center** (subject gestures ~300 px) → the head can wander within the fixed frame. This is acceptable at baseline (static is the default). The **one-euro fluid follow** (Step 10b) is available as an **OPT-IN** fix the CEO can request for such a take — it is NOT applied by default, NEVER a hard-gate, and its absence is NOT a reject.

## Phase-2 NOT in scope (CEO-removed — do not add)
- **VAD is OUT.** No `vad/generate_timestamps.py`, no TEN-VAD, no rough-cut tier. Silence removal is **transcription-driven only** (Parakeet word timestamps → `final_cut.py`), Step 7.
- **Energy / RMS onset detection is OUT.** Do NOT detect "audible onset" by audio energy/RMS to trim leads or define speech — it is VAD-class and it gives WRONG numbers (it falsely reported a ~0.3s "pre-roll"; Parakeet word.start is the ground truth, and it is precise/deterministic). The transcription defines speech; nothing measures audio energy.
- **Leading-silence trim is OUT.** Do NOT add a step that trims the start of a piece. `final_cut.py --pad-start 0.0` already starts the piece at the first word.
- **Tail-normalize is OUT.** Do NOT trim/pad piece tails toward a target (e.g. 0.22s). The tail is exactly `--pad-end 0.3` (clamped to source). No second pass.
- **Custom pad values are OUT.** Use EXACTLY `--pad-start 0.0 --pad-end 0.3`. If you think a pad is wrong, RAISE it to the CEO — never substitute your own value.
- **Relighting / fill-light is OUT.** No synthetic key/fill light, no mediapipe relight step. It was tested and the CEO removed it from the pipeline. Do not re-introduce.

---

# FLOW 3 — MULTICAM LONG-FORM (landscape) — raw Cam A / Cam B → one 16:9 video

> Proven on the CEO-accepted long-form runs: the 2-cam landscape edits (`danedelattre` live, Apple M5)
> and the content-aware `multicam-edit-v2` landscape renders. This is the **opposite deliverable from
> FLOW 2**: FLOW 2 makes a short **vertical 9:16 reel** per piece; FLOW 3 makes one long **landscape
> 16:9 3840×2160** video (a full talk/episode) by **cutting between Cam A and Cam B with per-shot
> landscape crops** — audio ALWAYS Cam A. The canonical chain is the `~/workspace/video-editing`
> skills `sync-cameras`/`find_offset` → `multicam-segment-matching` / `segment-video-prores-v2` →
> `edit-video-remove-silence` (rough+fine cut) → `framing-discovery` → `multicam-edit-v2`, with STT
> via the **fixed `/transcribe`**. Failure modes are folded from `README.md` "Hard-Won Lessons" and
> `LEARNINGS.md` — the CEO caught every one.

## Flow-3 Goal
One or two long raw Cam A / Cam B takes become **ONE landscape 16:9 3840×2160** edited video: silence
removed, **audio always Cam A** (better mic), cut between Cam A and Cam B at natural speech boundaries,
with **content-driven per-shot crops** (zoom arcs on the same camera, then a camera switch as the
payoff) — and no lip-sync drift, no head-crop, A/V locked, no black first frame.

## Transcription is the source-of-truth (the fixed `/transcribe`) — answers "is the transcription wired in?"
Every cut in FLOW 3 is defined by the **Cam-A transcription**, never by audio energy. Get it from the
**fixed STT API** `/transcribe` (NVIDIA Parakeet TDT 0.6B v3, port `:8101`):
```bash
ffmpeg -nostdin -v error -y -i camA.mp4 -ar 16000 -ac 1 -af dynaudnorm /tmp/a.wav
curl -s -X POST http://192.168.15.14:8101/transcribe -F "audio=@/tmp/a.wav" \
  [-F 'boost_phrases=["Camily","Plow"]'] -o camA_trans.json   # -> {text, words[], segments[], duration_s}
```
- **The long-form robustness FIX (server-side VAD) — why `/transcribe` is now trustworthy on long files.**
  Files **≥ 30 s** are transcribed **window-by-window using server-side Silero VAD segmentation**
  (`apps/stt-api/app/vad_segmentation.py`), NOT the old buffered/streaming RNNT decode. The streaming
  decode carried decoder state across fixed 10 s chunks and, after a pause, could latch into a
  blank-emitting / over-skipping state and **silently DELETE whole spans of full-level speech**
  (observed: **~32 s of real speech dropped across 4 spans on a 19-min file**, incl. an entire question
  at 17:18 returning **ZERO words**). The fix: Silero VAD detects speech regions, groups them into
  bounded **windows ≤ 24 s** (`WINDOW_MAX_S=24.0`, `VAD_MAX_SPEECH_S=22.0`, threshold `0.35`,
  `WINDOW_MAX_GAP_S=6.0`), decodes each window independently with the standard **non-streaming** NeMo
  path (no cross-window state → the decoder can't delete a passage), then offsets the word timestamps
  back to absolute file time and stitches them in order. Validated: a 24 s clip transcribes cleanly.
  Files **< 30 s** use the plain single-shot decode. (`model.py`: `LONG_FORM_MIN_DURATION_SECS=30.0`,
  routes to `transcribe_segmented(...)`.)
- **This server-side Silero VAD is NOT the banned VAD.** The ban (FLOW 2 + the global `final_cut` note)
  is on the **editing client** using energy/RMS VAD to *define speech or cut points*. The STT server's
  internal Silero VAD is just how it **chunks a long file** so the decoder stays reliable — it produces
  the word timestamps that ARE the source-of-truth. The module docstring states it: *"internal
  server-side chunking — distinct from the editing client's banned client-side energy heuristics."*
- **Send the WHOLE file in ONE `/transcribe` POST — NEVER window it client-side either.** The ban is
  not only on energy/RMS VAD; **client-side TIME windowing also drops words at the seam.** Slicing Cam-A
  into fixed windows on the client (the rejected `transcribe_short.py`: `WIN=30 s / STRIDE=25 s`, keep
  `word.start < STRIDE`) loses any word on a window boundary — it falls in window *i*'s discarded
  look-ahead `[STRIDE,WIN]` **and** lands on window *i+1*'s leading edge where the decoder has no
  left-context and never emits it. Proven loss: `'Certo?'` (abs 125.36–126.16) vanished from BOTH
  windows → the cut saw a 1.64 s gap and **deleted its 0.8 s of audio** (the word the CEO heard
  missing). The whole-file POST is the fix: the *server* VAD windows on speech (not blind time) and
  stitches absolute timestamps, so no real word sits on a discarded edge.
- **Pre-render GATE — every kept question/answer span MUST have words, or the cut deletes it.** Before
  building the EDL, for each planned keep-span confirm the transcript has words there. If a span is
  **empty but the audio has speech** — check it: `ffmpeg -ss S -t D -i camA_16k.wav -af volumedetect`
  ≈ **−17 dBFS** (dropped speech was full-level, vs the ~−50 dBFS floor; a real pause reads ≤ −26 dBFS)
  — that is a transcription DROP, not silence. **HOLD and report the exact timestamps; do NOT delete it
  silently and do NOT hand-recover/patch** (recovery edits reintroduce error). Fix the SOURCE — re-POST
  the whole file to the VAD `/transcribe` — then re-verify. This is the gate that caught the 17:18
  question (Q12) returning empty before it could be cut as silence.
- **NEVER Whisper** (it hallucinated a "Gravando" loop that HID a real take). If Parakeet/GPU is down,
  restart the STT API and re-transcribe; do not substitute another engine.

## Flow-3 pipeline (stages, in order)
1. **Sync Cam A / Cam B** (audio cross-correlation): `sync-cameras` / `src/sync/find_offset.py`.
   **VALIDATE the offset SIGN before any full render** — `find_offset`'s `trim_file` can be **inverted**
   (treat it as a hypothesis, not truth). A sign flip applies `-ss` to the wrong camera and *adds* to
   the skew → **~26.6 s desync (≈ 2× the true offset)**. Validate two ways: (a) **objective** —
   transcribe the SAME cut window from each camera independently and confirm they yield the **same
   words** (do it early/mid/late to also rule out drift; Cam B mic is quieter → `dynaudnorm` for STT but
   always keep Cam A audio in output); (b) **visual** — render one short side-by-side clip, eyeball
   lips/gestures L vs R. Cam A started first ⇒ feed `-hwaccel videotoolbox -ss <off> -i camA`, Cam B
   whole. The robust engine is the `multicam-segment-matching` sync core (`word_sync.py` +
   `refine_sync.py` `BAD_CONTENT` verdict + side-by-side sync_check clips) — use it even for unscripted
   edits.
2. **Transcribe Cam A** via the fixed `/transcribe` (above) — the source-of-truth for every cut.
3. **Segment + match takes** (`segment-video-prores-v2` / `multicam-segment-matching`): match plan
   paragraphs to the best take, extract **matched Cam A / Cam B ProRes segment pairs**, drift-corrected.
   Cam A is the reference clock + audio.
4. **Rough cut + fine cut (two-pass, transcription-driven)** — `edit-video-remove-silence`:
   - **Rough cut** = fast **keyframe-aligned stream copy** (`video/v8_keyframe_fast.py`, ~1 s precision,
     lossless, 50–100× realtime) → `rough_cut.mp4` for quick review.
   - **Fine (final) cut** = **frame-accurate `video/final_cut.py`** cutting at sentence boundaries on the
     **RAW `word.start`/`word.end`** AS-IS → `final_cut.mp4`. **This is the cut that ships.** The
     rough+fine rule, nothing on top: **break a segment when the inter-word gap `next.start − cur.end`
     exceeds the threshold; keep `[first.start, last.end + pad-end]`; drop the inter-segment pauses.**
     LOCKED long-form values **`--pad-start 0.0 --pad-end 0.35`**, break **threshold 0.35 s**. **Both
     raised from 0.30** on the CEO-accepted long-form run: at 0.30 the break fired on natural
     sentence-internal micro-pauses (e.g. `'com vocês,'` end 55.68 → `'Dani'` start 56.00 = a 0.32 s gap
     got cut mid-thought), and a 0.30 tail clipped the final consonant's release. **No `eff_end` cap, no
     onset trim, no silencedetect** — every hack added here came back as an audible artifact. (If a
     value seems wrong, RAISE it to the CEO — that is exactly how 0.30 → 0.35 happened — never substitute
     a heuristic.)
   - **Lip-sync lock:** de-silence Cam B with the **EXACT same Cam-A SOURCE transcription** (not a
     Cam-B / re-transcription) → identical frame cuts → Cam B frame-locked to Cam A. Verify
     `sync/word_sync.py` reports **offset≈0, drift≈0**.
5. **Framing discovery** (`framing-discovery`): SAM3 **MEASURES** the valid `(zoom, position)` framings
   per segment per camera (validates head-not-cut). Read each `_framing.json`. **Pick zoom levels ONLY
   from the measured set** — never hardcode (hardcoded levels cut his head).
6. **Content-aware EDL** (`multicam-edit-v2`): build the edit from the Cam-A transcript + the framing
   JSONs (see "The editing rules" below). Output is **landscape 3840×2160**, **per-window crops**, audio
   always Cam A.
7. **Render + concat** (`scripts/multicam_render_v2.py`, on **mac-pro**): per-segment ProRes, **pad audio
   per segment**, then **concat directly to H.264** (`-b:v 40M`). Hardware DECODE *and* ENCODE.

## The editing rules (multicam-edit-v2 — landscape per-shot crops)
Core insight: switching cameras without zoom variation looks flat. **BUILD zoom progressions on one
camera, then switch cameras as the visual payoff.** The viewer sees: build → switch → build → switch.
- **Cuts ONLY at natural speech boundaries** from the Cam-A transcript (`.!?`+pause>80 ms / breath>200 ms
  / comma once the window is ≥1.5 s). `MIN_CUT_INTERVAL = 1.0 s`. **There is NO MAX_WINDOW** — never
  force a cut mid-sentence; a 7 s unbroken phrase stays a 7 s window.
- **Camera runs of 2–4 windows** on the same camera before switching. **No single-window ABAB
  alternation** (a 1-window run is forbidden except the last window of a segment).
- **Zoom arc within each run** — **escalate** (push in) for building intensity / approaching a reveal;
  **de-escalate** (pull out) to reset after a climax / new topic. **Every same-camera step must be ≥ 10%
  RELATIVE to the current crop (the zoom ratio — `new_crop_w ≤ 0.90·current_crop_w`), else HOLD the crop
  (Step-10 HARD RULE 4, CEO 2026-06-22). The 10% is relative-to-current-crop, NOT % of the 4K frame.** No
  sub-10% micro-jitter — snap each step to ≥10% relative or to no change.
- **Camera switch resets zoom** — the new run starts at a zoom that contrasts with where the previous
  run ended (the switch is the "breath".)
- **Per-shot crop = the framing-JSON crop** (`crop_x/y/w/h`, **center positions only**, `pos_key=="center"`),
  scaled to 3840×2160. **z0 = full frame** (`crop_w=3840, crop_h=2160`, a no-op crop).
- **HEAD-SAFE ANCHORS — SAM3-TRUTH (CEO 2026-06-24, folded — supersedes the old face-box derivation).** Read the head geometry DIRECTLY from SAM3's `head` box and the eyeline from the `eyes` box — never from the `face` box. From the `head` box (`y1`=top, `y2`=bottom): `crown = head.y1` (real top of hair/skull), `chin = head.y2`, `head_h = head.height`. From the `eyes` box: `eyes_y = eyes.center_y`. The old derivations (`head_height = 1.25·fh`, `crown = face.y1 − 0.25·fh`, `eyes_y = face.y1 + 0.35·fh`, the worked-example `ftop − 0.30·fh`) are **DELETED** — they guessed the crown from the face box and drifted from the real hair, causing both clipping (crown too low) and over-rejection of good footage (crown too high). **Headroom (≥7%) and the head cap (≤70%) are measured to the SAM3 crown / head-height.** Eyeline target is **CEO-tuned 36%** (≤42% hard gate), subordinate to the 7% headroom floor.
  - **SAM3-TRUTH VALIDITY (CEO 2026-06-24, data-derived — NO magic numbers):** a per-frame SAM3 sample is **VALID iff SAM3 RETURNS both the `head` and `eyes` boxes** — with **NO confidence floor**. We do not impose our own cutoff (that would second-guess the model). *Evidence:* across a 37-frame masks-validation run SAM3 returned NONE on 7/7 truly-empty frames (no hallucinated boxes) and a correct box on every frame with a head present (real-head conf band 0.838–0.944, median 0.93); a confidence floor is therefore both unnecessary (no garbage to filter) and harmful (an 0.85 floor would wrongly discard a real 0.838 frame).
  - **REJECTION (NARROW — the ONLY two; CEO 2026-06-24):** reject a frame ONLY if **(a) NO face/person is DETECTED in it** (SAM3 returns no `head` — empty chair / person left frame) **OR (b) NO SAM3 data** (segmentation unavailable/failed). This is face **PRESENCE, not identity** — there is **NO identity/recognition** (we do not distinguish the subject from a guest by face-matching). Nothing else is ever a rejection.
  - **DATA PRESENT → ALWAYS CROP-TO-VALID, NEVER DROP (CEO 2026-06-24).** If SAM3 returns the head, the frame is USABLE. The ORIGINAL framing may be unusable as-is, but you **compute a valid crop** from the SAM3 head+eyes anchors via the go-wider ladder — recenter X → wider zoom → reset to the widest gate-passing crop → switch the beat's video to the other camera (audio stays Cam A) — until the **VALID-CROP RULE SET** (below) holds. A valid crop ALWAYS exists for an in-frame head (full-frame / the widest-gate-passing crop is the floor of the ladder). **"We have data but the original is bad" must NEVER drop the beat** — it crops to valid. *(There is NO "drop the beat / drop the span / invalid camera-beat" — that was an invented over-rejection and is removed.)*
  - **MULTIPLE HEADS (guest / reflection in frame):** when SAM3 returns more than one `head`, pick the subject **DETERMINISTICALLY** — the head consistent with the `eyes` box (eyes center inside the head box), else the most prominent/central head. This is **selection, not identity** (it resolves WHICH detection is one person, not WHO).
  - **CAMERA FOOTAGE BOUNDS — out-of-range = no-data (CEO 2026-06-24, crash-fix from the first SAM3-truth live).** A camera's footage exists ONLY in `[0, cam_dur]`. The two cameras can differ in length or start-offset — e.g. cam-B stopped recording ~14 s before cam-A (765 s vs 779 s), or a beat lands before cam-B's start so its `abs_time = beat − TAU < 0`. A frame request OUTSIDE `[0, cam_dur]` has **NO data** → treat it as a no-data sample (rejection-b), **NEVER extract it** (ffmpeg clamps an out-of-range `-ss` and returns a garbage frame that SAM3 will happily box — silently mis-framing) and **NEVER render it** (a negative or past-end seek produces an invalid/zero-length segment → render crash). So the SAM3 sampler MUST gate on `0 ≤ abs_time(cam,t) ≤ cam_dur` and return no-data otherwise. Consequences: (a) beats near the start/end where one camera lacks footage use the OTHER camera; (b) a no-valid-crop beat (no face on either camera — e.g. the intro/outro) defaults to **cam-A**, which always spans the edited content, NOT cam-B (which may be out of range → negative seek). **VERIFY:** no rendered segment seeks any camera outside `[0, cam_dur]`.
- **Available angles only** (the measured set); ~**40–60% each camera** by time (a balance skew is a
  **warning, not an error** — it comes from natural speech timing; do NOT re-run the EDL builder to "fix"
  it).
- **Audio is ALWAYS Cam A, passed through WHOLE** — the filtergraph is **VIDEO ONLY**
  (`concat=n=N:v=1:a=0[outv]`, then `-map "[outv]" -map 0:a`). **NEVER chop audio into per-window
  `atrim` segments** — that makes micro-glitches at every cut.
- **(Optional) TBP — Text Behind Person:** pick exactly **3** high-impact words; pre-render Cam TBP clips
  (`scripts/cam_tbp_prerender.py`, SAM3 masks via REST) and **overlay them WITHIN the window — NEVER
  split a concat entry** (splitting into 3 entries causes **41.7 ms PTS drift**; overlay keeps one entry
  per window → identical PTS to a no-TBP render).

## Flow-3 render rules (the proven ffmpeg discipline)
- **PLATFORM — macOS videotoolbox is PRIMARY; Linux/CUDA NVENC is a documented SECONDARY (CEO 2026-06-25, harden #2 gap #1).** The CEO's real production edits run on the **Mac**, and **Rule 6 mandates Apple hardware** — so **macOS `videotoolbox` (`h264_videotoolbox` / `hevc_videotoolbox` / `prores_videotoolbox`) is THE primary encode contract for production. Never libx264/x265 for 4K (either platform).** A **SECONDARY** path exists for running this pipeline on the **Linux/CUDA GPU server** (the harden-test substrate, or any non-Mac host): map the SAME intent (HW decode + HW encode + the concat-FILTER assembly) to CUDA — decode `-hwaccel cuda`, encode **`hevc_nvenc`** (`-rc vbr -cq 19`) or **`h264_nvenc`** (`-b:v 40–50M`); there is **no `prores_videotoolbox` on Linux**, so use the **concat-FILTER path** (each input decoded with its own params), not ProRes intermediates. Pick by host: `videotoolbox` available → primary (Mac, production); else CUDA NVENC (Linux, harden/server). Everything else in these rules (one `select`, even crop dims, per-segment apad, declick, resumable/error-checked passes, LOCAL-only + pre-render gate, the PTS-realign path rule) is platform-neutral and applies to both.
- **HW-DECODE CHROMA/CODEC FALLBACK — per-input, expected, NOT a failure (CEO 2026-06-25, harden #3 gap #1).** "`-hwaccel` before EVERY `-i`" is the goal, but a HW decoder can't always handle a source's codec/chroma: e.g. **4:2:2 10-bit HEVC** (Rext `yuv422p10le`) is **not supported by NVIDIA NVDEC on Ampere** (`hevc_cuvid` → "not supported with this chroma format"), and the two cameras may differ (one 8-bit 4:2:0 h264, the other 10-bit 4:2:2 HEVC). When a specific input rejects HW decode, **software-decode THAT input only** (drop its `-hwaccel`), keep HW decode on the others, and keep **HW ENCODE** on the output. This is normal, NOT the "forgot `-hwaccel` → speed<1x" mistake — **probe each source's codec/pix_fmt up front, decide HW-vs-SW decode per input, and record it in the timing artifact** (note which inputs fell back). It only slows that one decode, not the encode.
- **Hardware DECODE matters as much as ENCODE:** `-hwaccel videotoolbox` (Mac) / `-hwaccel cuda` (Linux) before **EVERY** `-i`, plus
  `-c:v h264_videotoolbox -b:v 40M` (or `hevc_videotoolbox -q:v 65 -tag:v hvc1`) on output [Mac; Linux = `h264_nvenc`/`hevc_nvenc`]. Measured
  M5: sw decode **97 s** vs hw **34 s** per 90 s dual-4K slice (**2.85×**); a 25-min 2-cam render
  dropped **~27 → ~9 min**. **`speed < 1x` on an M-series Mac = a CPU-bound stage** (software decode or
  filter fan-out).
- **One `select`, not N `trim`s** for many keep-windows (per-window trim fan-out ran at **0.18×**). Cut
  video by **integer frame INDEX** (`select='between(n,round(a*fps),round(b*fps)-1)'`) and audio by
  `atrim` — **time-based `select` desyncs** (saw **+1.33 s over 18 min**); aselect is silently ignored
  on PCM.
- **Render on mac-pro** (`prores_videotoolbox` is macOS-only). Use **`-filter_complex_script`** (filter
  strings exceed shell limits with many windows). **All crop dims even.** Do **NOT** use `settb=1/30`
  for same-session multicam (requantizes 41.7 ms frame PTS onto a mismatched grid → boundary
  artifacts); set `-video_track_timescale` on the container only.
- **Pad audio per segment before concat** (`-af apad -t <vdur> -c:v copy`); frame-boundary A/V drift
  accumulates (~20 ms/segment, ~480 ms over 24 segments). `-af apad -shortest` does **NOT** fix it.
- **DECLICK FADE at silence-cut splices — REQUIRED in FLOW 3 too (CEO 2026-06-22; folded after a real "tic noise" rejection). Same step as FLOW 2 Step 9 (L679).** One continuous Cam-A `atrim` per segment removes the *camera-switch* click (switches are video-only inside the segment), **but each silence-cut SEGMENT boundary is still a splice of two non-adjacent source samples**: Parakeet's `word.end` is conservative (the vowel sustains past the label), so the `0.35 s` pad-end frequently lands on **still-voiced, high-amplitude** audio → a non-zero-crossing step = a few-ms **tic**. Apply a **~4 ms fade-out at each segment's trailing edge** (`afade=t=out:st=<segdur−0.004>:d=0.004`) **+ ~4 ms fade-in at its leading edge** (`afade=t=in:st=0:d=0.004`), **BEFORE the per-segment `apad`** — **ONLY at silence-cut segment boundaries, NEVER at camera switches** (those rejoin the same Cam-A sample and are already seamless; fading them notches real speech). Fades ramp amplitude in place → segment durations UNCHANGED → no A/V shift. **VERIFY:** every silence-cut join ramps to a **near-zero dip** (`min|amp| → ~0` within ±6 ms of the join). *(NB: the worked-example claim that segment boundaries are "quiet" is true of the LEVEL but the pad can still be voiced — the declick is required, not optional.)*
- **NEVER `-c copy` concat separately-encoded HEVC clips** → **green frames** at joins (`hvc1` keeps
  SPS/PPS/VPS out-of-band, undecodable past clip 1). Use the **concat FILTER** (each input decoded with
  its own params) **or ProRes intermediates** (intra-only → every frame a keyframe → concats with
  `-c copy`).
- **Final concat goes DIRECTLY to H.264 MP4** — never an intermediate ProRes concat (~28 GB for 12 min;
  fills mac-pro's disk). Per-segment ProRes is fine (temporary).
- **PER-SEGMENT INTERMEDIATES = ProRes LT, NOT HQ (CEO 2026-06-22, folded after a disk-fill on a 33-min 4K edit).** Use `prores_videotoolbox -profile:v 1` (LT), never `-profile:v 3` (HQ). The intermediate is re-encoded to the H.264 50M final, so LT is **visually lossless for this purpose** but **~half the footprint**. Real numbers: 4K HQ ≈ 1 GB/min (276 segs/20 min ≈ 120 GB → filled the disk); 4K **LT ≈ 0.5 GB/min** (≈ 61 GB). (ProRes proxy/profile 0 only if disk is still tight — slightly softer.)
- **RENDER MUST BE RESUMABLE + EVERY PASS ERROR-CHECKED/RETRIED (CEO 2026-06-22).** (a) Before rendering a segment, if its `seg_NNN.mov` already exists AND `ffprobe` returns BOTH a v:0 and a:0 duration, **skip it** (resume after any crash — never re-render 100+ good segments). (b) **Check the return code / validate the output of EVERY ffmpeg pass** (the ProRes pass AND the audio-pad/apad pass) and **retry** on failure; a pass that "succeeds" with rc 0 can still write a **truncated, moov-less file** when the disk fills — validate with `ffprobe`, do not trust rc alone. (c) Name segments **zero-padded to 3 digits** (`seg_{i:03d}`) and concat in **NUMERIC** order — lexical `sorted(glob)` puts `seg_100` between `seg_10` and `seg_11` and silently ships the video out of order.
- **RENDER LOCAL-ONLY + PRE-RENDER SPACE GATE — FAIL UPFRONT, NO FALLBACK (CEO 2026-06-24, supersedes the earlier disk-headroom note).** Render the video to **LOCAL disk ONLY** — segs + concat (`_ord`) + realign (`_r`) intermediates AND the final master all on a local scratch on the render host. **NEVER render to the NAS or any remotely-mounted disk** (NFS is far too slow — measured: the final `+faststart` ran **21 min** on a 5.5 GB file over NFS vs **~13 s** local; a whole render went ~1h34m on NFS, wedged, vs ~25-35 min local). **There is NO fallback of any kind.** Because the encode params are KNOWN, the required space is **predictable up front** — so this is a **PRE-RENDER GATE**: estimate the peak LOCAL footprint and **ABORT BEFORE the segment loop starts** (never mid-render) if local free space is insufficient. **Estimate (peak = segs + `_ord` + `_r` + final all coexisting before cleanup):** `content_min = sum(kept window durations)/60`; `seg_GB = 0.5 × content_min` (ProRes LT 4K ≈ 0.5 GB/min, `prores_videotoolbox -profile:v 1`); `h264_GB = (50e6/8)·(content_min·60)/1e9 = 0.375 × content_min` PER H.264 file at `-b:v 50M` (the `_ord`, `_r`, and final each); `need_GB = (seg_GB + 3·h264_GB) × 1.3` (+30% for mux temps + filesystem overhead). If `local_free_GB < need_GB` → **abort with a clear message** ("need X GB, have Y GB, render is LOCAL-ONLY — free space and retry"); do NOT start, do NOT fall back to NAS. (Worked: a 14.8-min live ⇒ need ≈ 31 GB; a 50-min live ⇒ ≈ 106 GB — won't fit a 44-80 GB disk, so it aborts cleanly instead of disk-filling or crawling over NFS.) **Per-segment intermediates = ProRes LT** (see the rule above). After the final mux+realign verifies, **delete the local scratch (per-segment ProRes + intermediates + frames)** and any prior completed-job work dirs so the next edit starts clean.
- **PIPELINE SCRIPTS RUN FROM THE WORK DIR — copy them in at orchestration start (CEO 2026-06-24, crash-fix).** The framing / render / upload stages run as `python3 framing.py` / `render.py` / `upload_clean.py` from **inside the per-live work dir** (they read `config.json`, `edl.json`, `boxes.json` from cwd). So the orchestrator (`process.sh`) MUST **copy the canonical scripts into the work dir** before running them (`cp <scripts>/{framing,render,upload_clean}.py "$WORKDIR"/`). Without this the run dies immediately with `can't open file 'framing.py'`. (This copy used to live in a top-level `run_all.sh`; that file was deleted per the no-daemon rule, so the copy now belongs in `process.sh` itself.) Copying the canonical scripts in each run also guarantees the work dir always uses the LATEST scripts (no stale per-live copy).
- **Black-first-frame PTS realign — PATH-DEPENDENT (CEO 2026-06-25 fold, harden #2 gap #11). The itsoffset realign is ONLY for the `-c copy` concat-DEMUXER path; on the concat-FILTER path it REINTRODUCES the offset.** Goal both ways: v:0 `start_time == a:0 == 0` AND `dur(v) ≥ dur(a)` (audio capped to video).
  - **Concat-FILTER path (what FLOW 3 uses — see "Use the concat FILTER" above):** the `concat` filter already lands `start_time = 0`. Do **NOT** apply `-itsoffset $VS … -avoid_negative_ts make_zero` — measured, it puts a **~0.12 s** offset BACK (v:0 `start_time` 0.000 → 0.125) = the very black-first-frame it was meant to fix. (This is exactly FLOW 1's warning, L229/L258 — "the itsoffset realign puts the offset back here" — it applies to FLOW 3's filter path too.) Instead, ONLY **cap audio to the video duration**: `-c copy -t $VD` (or re-mux `-map 0:v -map 0:a -c copy -t $VD`). That keeps `start_time = 0` and gives `dur(v) ≥ dur(a)`.
  - **Concat-DEMUXER (`-c copy` of ProRes/intra segments) path:** here the realign IS needed — `-itsoffset $VS -i in … -map 0:v -map 1:a -c copy -avoid_negative_ts make_zero -t $VD`.
  - **VERIFY (both paths):** `ffprobe` v:0 `start_time == a:0 == 0` and `dur(v) ≥ dur(a)`. If `start_time` is non-zero you applied the wrong path's fix — switch.

## Flow-3 worked example — exact filtergraph, crop math & real numbers from the accepted v8 (first-hand)
> Written by the engineer who actually rendered the accepted long-form v8 (`build_v8.py` → `edl_v8.json`
> → `render_v8.py`). These are the reproducible specifics behind rules 4–7. **v8 = 120 segments, 245
> windows, 118 camera-runs, SAM3 245/245 head-safe, Cam A 48.5% / Cam B 51.5%, zooms {0,15,20,25,35},
> de-silence threshold 0.35 / pad 0.0/0.35, HEVC 1920×1080 23.976 fps, 14.16 min.** (v8 delivered
> 1920×1080; the crop math and the audio rule are resolution-independent — scale to your delivery size.)

- **(4) Continuous Cam-A audio per SEGMENT — the exact filtergraph that kills the click.** The unit of
  audio is the **speech SEGMENT** (a silence-cut run), NOT the window. Per segment, decode both 4K cams
  once, build video by concatenating the per-window crops with **`a=0`** (video only), and pull **ONE
  Cam-A `atrim` for the whole segment**:
  ```
  # per window k  (Cam A = input 0, Cam B = input 1):
  [{inp}:v]trim=start={rel_start}:end={rel_end},setpts=PTS-STARTPTS,crop={cw}:{ch}:{cx}:{cy},scale=1920:1080,setsar=1[v{k}]
  # stitch windows VIDEO-ONLY, plus ONE continuous Cam-A audio for the whole segment:
  [v0][v1]...[v{K-1}]concat=n={K}:v=1:a=0[outv]
  [0:a]atrim=start=0:end={segdur},asetpts=PTS-STARTPTS[outa]      # -map [outv] -map [outa]
  ```
  Decode windows with `-hwaccel videotoolbox -ss {seg_start} -t {segdur+0.5} -i A` and
  `-ss {seg_start-offset} -t {segdur+0.5} -i B` (the +0.5 tail keeps the last frame); then **pad audio to
  the video length per segment** before concat (`-af apad -t {vdur} -c:v copy`).
  **The error it avoids:** an A↔B switch lands **mid-speech**; if you also cut audio there (per-window
  `atrim`, or splitting `[0:a]`/`[1:a]` per window) you get a PCM sample discontinuity = an audible
  **click** at every switch (and you'd be splicing in Cam-B's worse mic). One atrim per segment means
  audio is only ever cut at **segment boundaries**, which ARE the silence-cut points (quiet) — switches
  inside a segment never touch audio. (Skipping the per-segment `apad` lets frame-boundary A/V drift
  accumulate ~20 ms/segment.)

- **(5) SAM3 head-safe crop — the exact math (`valid_crop`), why nothing gets decapitated.** *(SAM3-TRUTH, CEO 2026-06-24 — the legacy face-box version of this example, which guessed `crown = ftop − 0.30·fh` and gated on face-span heuristics, is superseded by the SAM3-TRUTH anchors + the VALID-CROP RULE SET at L704.)* Per window,
  sample **3 frames** (start+0.15 s, midpoint, end−0.15 s) → SAM3 `:8103` `prompt=head` AND `prompt=eyes` → the subject's boxes (highest-confidence head consistent with the eyes box). Run SAM3 on a 1280×720 downscale and scale coords **×3** back to 3840×2160; average the 3 head boxes and 3 eyes boxes. Read the anchors directly:
  `crown = head.y1`, `chin = head.y2`, `head_h = head.height`, `eyes_y = eyes.center_y`.
  A zoom level is **valid only if it satisfies the VALID-CROP RULE SET** (L704): `eyeline ≤ 42% AND head ≤ 0.70 AND headroom ≥ 7% AND crop ≥ 2688×1512 AND X-centered`. Pick the tightest zoom that passes; if none passes, **go wider** down the ladder (discrete zooms → widest gate-passing crop → switch camera) — never drop a beat that has data. Centre on
  the head, clamp the top so the crown stays in: `cx = clamp(head.center_x − cw/2, 0, 3840−cw)`,
  `cy = clamp(min(eyes_y − 0.36·ch, crown − 0.07·ch), 0, 2160−ch)`, **all dims even**.
  - **EVEN-ROUNDING — positions allow 0; clamp in-bounds AFTER rounding (CEO 2026-06-24, crash-fix).** The `even()` used for crop **sizes** floors at a 2px minimum (a 0-size crop is invalid). Do NOT apply that same floor to crop **positions** (`crop_x`/`crop_y`): a position of 0 is legal, and forcing `0 → 2` pushes a **full-frame** crop (`crop_w = 3840` / `crop_h = 2160`) 2px past the frame edge (`2 + 3840 > 3840`) → ffmpeg `crop` errors → render crash. Round positions to even **allowing 0** and then **clamp in-bounds**: `crop_x = clamp(even₀(crop_x), 0, W−crop_w)`, `crop_y = clamp(even₀(crop_y), 0, H−crop_h)` (where `even₀` rounds to even without the 2px floor). This bites specifically on full-frame / widest-gate-passing crops (the no-face and source-limited beats).
  **The error it avoids:** a hardcoded zoom (or centring on the body) shears the crown off a tall /
  gesturing subject. Measuring the REAL head box per window (not a face-box guess) is what keeps every
  window head-safe. Sample **3 points** across the window — the head moves within a 2–4 s shot; one frame misses it.

- **(6) Camera-runs driven by CONTENT, not a blind A/B toggle or a %-counter.** Group windows into **runs
  of 2–4** on one camera; run length and zoom-arc come from the text's **energy** (emphasis words, `!`,
  CTA terms, `?`/pivot words), then alternate A/B **between** runs. v8 produced **118 runs** and a **48.5
  / 51.5** A/B split that **emerged from the content**. **The error it avoids:** per-window ABAB toggling
  looks flat/epileptic; "fixing" the split by re-running the EDL to force 50/50 re-cuts against the
  speech and destroys the build→switch rhythm. A balance skew is a **warning, not an error** — leave it.

- **(7) Landscape crop is STATIC per shot — one rectangle per window, no pan.** Each window = one fixed
  crop rectangle at the run's zoom, `scale=1920:1080`. **No** per-frame pan / head-follow in landscape
  (that is the FLOW-2 vertical technique). **The error it avoids:** an automatic pan in a landscape
  close-up **creeps/jitters** to re-centre mid-take (the CEO rejects it); the crop is chosen ONCE from the
  SAM3 box for the whole window and **steps only at the next speech boundary**.

## Flow-3 Verify (hard — verified on the accepted runs)
- **Inter-camera sync VALIDATED (not just A/V duration):** per-window Cam-A vs Cam-B transcriptions match
  early/mid/late, residual audio lag a few ms, offset constant out to the end (no drift). **A/V-length
  match alone is NOT sync** (a 26.6 s desync once passed that check).
- **Lip-sync** Cam A↔Cam B: `word_sync` **offset≈0, drift≈0**; spot-check a Cam-B-shot mouth vs Cam-A
  audio.
- **Transcription COMPLETE — no dropped spans** (the long-form failure this flow exists to prevent):
  re-check the tail and any post-pause passage actually have words. This is exactly what the server-side
  VAD fix guarantees. **Close the loop on the DELIVERED file, not the plan:** for each marquee span
  (e.g. each question), extract it from the FINAL mp4 at its output-timeline offset and **re-transcribe
  that clip** to prove the words are really there — `ffmpeg -ss <out_t> -t 8 -i final.mp4 -ar 16000 -ac 1
  /tmp/q.wav ; curl -F audio=@/tmp/q.wav .../transcribe`. The EDL claiming "kept" is not proof; the
  rendered audio is. (Q12 confirmed this way: the output clip transcribed back to "O que você
  configurou uma vez e salvou o tempo…".)
  - **ALLOW ASR NORMALIZATION — match on SEMANTIC markers, not exact substrings (CEO 2026-06-25 fold, harden #2 gap #14).** Parakeet renders the SAME speech differently between a whole-file pass and a short clip (e.g. "três mil reais" → "R$3.000,00"; "seed" → "CID"; punctuation/casing vary). So an exact-substring Verify gives FALSE FAILs. Confirm completeness by **content markers** (the distinctive words of each marquee span are present, in order) and per-span re-transcription, tolerating number/format normalization — not byte-equality with the EDL text.
- **Landscape 3840×2160**, **audio Cam A only**, **A/V locked** (`dur(v) ≥ dur(a)`), clean joins, **no
  black first frame**.
- **Edit rules hold:** camera runs 2–4, zoom progresses within each run and resets on switch, every angle
  exists in the measured framing JSON, head-safe (SAM3 face inside crop). **TBP (if used):** exactly 3
  words, each window kept as ONE concat entry (validated 572/572 frames, 0 PTS diff vs no-TBP).
- **MINIMUM CROP-CHANGE held (Step-10 HARD RULE 4, CEO 2026-06-22):** walk the EDL; for every pair of
  **consecutive SAME-camera shots**, the relative crop change `1 − min(w1,w2)/max(w1,w2)` is **either ≥ 0.10
  (a real ≥10% push-in/pull-out) or exactly 0 (held)** — **FLAG any same-cam consecutive crop change that is
  > 0 and < 10% relative** (a banned sub-10% micro-jitter). Camera-switch boundaries are exempt (zoom resets
  freely across a switch).
  - **FEASIBILITY — when neither "equal" nor "≥10% apart" is reachable, HOLD via a stable per-run crop size (CEO 2026-06-25 fold, harden #2 gap #7).** On a tight close camera the gate-valid crop band can be so narrow that two consecutive same-cam windows can be made *neither* equal *nor* ≥10% apart — the rule has no feasible deliberate move. Resolution: compute `crop_h` from a **stable per-run head reference** — one `crop_h` that is gate-valid for ALL windows in the run (brute-force the common-valid size; this is the run-median idea in Step 10's jitter-fix note) — so the size is **constant within the run = HOLD (0 change)**, which satisfies the rule. Only step crop sizes at deliberate ≥10% beats; otherwise hold one per-run size.
- **TIMING ARTIFACT GATE (measurable — added 2026-06-25; same contract as Step 10 PER-STEP TIMING).** This 16:9 long-form flow IS the production pipeline, so it ships the per-step timing artifact and it passes the same gate: every stage **(a)–(g)** has a `provenance: measured` run-time atomic timer; framing reports the **(d1) extraction vs (d2) SAM3** split; a working stage reports non-zero `work_wall_s` while a `cached:true` stage reports `work_wall_s:0` + `prior_measured_wall_s` (a framing stage at ≈0 s NOT marked `cached` FAILS — the 06-24 false-timing bug); the **cleanliness flag** (`clean`/`contended` + restart count / named contender) is present; the **BOTTLENECK** stage is named. A `reconstructed`/`contended` framing time is NEVER reported as a clean reference.

## Flow-3 Failure modes (folded from the real runs — the CEO caught these)
- **Trusted `find_offset`'s sign blindly** → `-ss` applied to the WRONG camera → **~26.6 s desync** (2×
  the offset). VALIDATE the sign (transcription + visual) before rendering.
- **"Sync verified" meant A/V length match** → the cameras were 26.6 s apart while the check passed. A/V
  duration ≠ inter-camera sync — different failure mode entirely.
- **Streaming/buffered RNNT on a long file** → silently **DELETED ~32 s of speech** (a whole question =
  ZERO words). Use the **fixed `/transcribe`** (server-side VAD kicks in at ≥30 s).
- **Client-side windowing of the audio before transcribing** (`WIN=30/STRIDE=25`) → dropped a word on
  the window seam (`'Certo?'`) → its 0.8 s of audio cut as "silence". POST the WHOLE file once; let the
  server VAD do the windowing. Empty span + full-level audio ⇒ HOLD, never delete or hand-patch.
- **Silence threshold / pad too tight at 0.30** → broke segments on natural sentence-internal pauses
  (0.32 s gap cut mid-thought) and clipped final-consonant releases. Long-form locked at **0.35 / pad
  0.0/0.35** on raw `word.start/end`; RAISE to the CEO if a value seems off, never swap in a heuristic.
- **Whisper instead of Parakeet** → hallucinated a "Gravando" loop that hid a real take. Never Whisper.
- **Cam B de-silenced with a different / re-transcription** → **lip-sync drift** (shipped a 36 ms
  divergence once). Use the SAME Cam-A SOURCE transcription for both cameras; verify `word_sync` offset≈0.
- **Software decode of 4K inputs** → `speed≈0.5x` (forgot `-hwaccel` per input). **Per-window `trim`
  fan-out** → `0.18×`. **`-c copy` concat of separately-encoded HEVC** → green frames. **Time-based
  `select`** → A/V drift (+1.33 s/18 min) — cut by frame index.
- **Hardcoded zoom levels** → over-cropped, cut his head. Use `framing-discovery`'s MEASURED gates.
- **Split a TBP window into 3 concat entries** → **41.7 ms PTS drift**. Overlay within the single window
  (`eof_action=pass`); NEVER apply TBP after the render (re-encodes the segment, destroys PTS).
- **The vertical-flow crop lessons (centering-3 / one-euro fluid follow / static↔fluid) do NOT apply to
  FLOW 3.** FLOW 3 crops are **fixed per shot** from the framing JSON and **stepped at speech
  boundaries** — there is no per-frame head-follow in landscape.

## Flow-3 NOT in scope (do not add)
- **No vertical reframe / centering-3 / SAM3 vertical 9:16 crop** — that is FLOW 2. FLOW 3 is landscape;
  SAM3 here is used only by `framing-discovery` (measure crops) and optional TBP masks.
- **No client-side energy/RMS VAD** for silence/speech (same ban as FLOW 2). The transcription is the
  source-of-truth; the ONLY VAD allowed is the STT server's internal Silero windowing inside
  `/transcribe`.
- **No relight / fill-light, no custom `final_cut` pad/threshold values** (same CEO-removal discipline as
  FLOW 2 — FLOW 3 long-form is LOCKED at `--pad-start 0.0 --pad-end 0.35`, break threshold `0.35`; if a
  value seems wrong, RAISE it to the CEO, never substitute a heuristic).
