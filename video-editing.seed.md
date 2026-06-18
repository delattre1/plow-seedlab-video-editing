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

- **DEV/VALIDATION ONLY — fast seed check:** cut the final **3:00** from each input (e.g. Cam A `C4779`,
  Cam B `C4831`), A/B-offset aligned so both cameras cover the SAME window, and run the chosen flow on
  that slice. A clean 3-min run proves the seed works; iterate on the seed using this fast loop.
- **PRODUCTION — edit the FULL video:** once the seed is validated, run it on the ENTIRE source (full
  length). The whole-video edit is the deliverable. **Never ship the 3-min slice as the output.**
- Applies to every flow (FLOW 1 / 2 / 3): the 3-min slice is the dev harness; production is full-length.
  The same gold-standard bar applies either way.

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
- **SPLICE RULE — preserve the previous valid word's tail (pad-end).** When you drop a false-start
  fragment, do **NOT** cut at the previous word's `end` timestamp — that clips the word and the splice
  mismatches (abrupt). Keep the previous valid word PLUS its pad-end: the out-point is
  `prev_word.end + min(pad-end, silence_before_the_fragment)`, then resume at the next valid word's
  `start`. This keeps the real breath after the last good word and never bleeds the dropped fragment's
  audio in. (Real audit: cutting at `prev.end` with 0 pad produced clipped splices — fixed by preserving
  the tail; e.g. "ajuda" kept its full 0.30 s, "configurei," kept the 0.16 s that actually existed.)
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
  (c) any interpreter where `import numpy` succeeds. Record which interpreter you bound to and
  use it for every Step-3 computation. This is preflight, not a mid-run "fix".

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
> cc=irfft(R)`, search ±10 s, `argmax|cc|` → integer sample `shift`; **`tau = shift/fs`**
> is the delay where **`camB(t) ≈ camA(t − tau)`**. Report `tau` in **seconds** (sub-frame
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

```
  cam-A SOURCE segment (4K 3840x2160, one take = one hook/body/cta)
        |
        v
  [1] PARAKEET transcription  -->  word-level timestamps          (NO VAD)
        |                          speech = word spans
        |                          silence = gaps between words
        v
  [2] TRANSCRIPTION-DRIVEN silence removal  (final_cut.py --pad-start 0.0 --pad-end 0.3)
        cut points = Parakeet word/segment timestamps ONLY
        pad-start 0.0 -> piece starts AT the first word (final_cut cuts at segment.start = word.start)
        pad-end   0.3 -> keep up to 0.3s after the last word (clamped to source clip end)
        NOTHING on top: no energy/RMS onset, no leading-silence trim, no tail-normalize
        |
        v
  [3] SAM3 VERTICAL reframe   (face bbox -> 9:16 crop 1215x2160 -> scale 1080x1920)
        |
        v
  [4] CENTERING-3 fluid       (SAM3 head @0.1s -> one-euro mc=1.8 b=0.25 -> 30fps sendcmd crop x)
        head pinned dead-center (off-center ~0.7px avg)
        |
        v
  EDITED PIECE  (vertical 1080x1920, audio cam-A; lead/tail = pad-start 0.0 / pad-end 0.3)   [NO relight]

  ...repeat for all 22 pieces:  10 hooks (H1..H10) | 2 bodies (B1,B2) | 10 CTAs (C1..C10)...

        |
        v
  [5] CONCAT  hookX + bodyY + ctaZ        (encode each piece -> H.264 once ; concat-demuxer -c:v copy ; PTS-realign)
        seam = previous piece pad-end tail + next piece (starts at its first word) ; no clipped words
        |
        v
  [6] 200 COMBOS = 10 hooks  x  2 bodies  x  10 CTAs

            B1                         B2
        +---------- C1..C10        +---------- C1..C10
   H1 --+                     H1 --+
   H2 --+   = 10x10 = 100     H2 --+   = 10x10 = 100      -->  200 videos total
   ...  |                     ...  |
   H10 -+                     H10 -+
                                                hookX-bodyY-ctaZ.mp4  (9:16, ~55-58s each)
```

(A/B multicam, Steps 8–10, stays available for future videos; this launch-reels batch is 100% cam-A.)

## Phase-2 RUNBOOK (reproducible — exactly what was run; a fresh engineer follows this verbatim on any new cam-A footage)

Inputs: cam-A source segment clips (4K 3840×2160, one take = one piece). APIs up: Parakeet STT `:8101`, SAM3 `:8100`. ffmpeg = `/opt/homebrew/bin/ffmpeg`. Per piece, in order:

```bash
# 1) PARAKEET transcription (NEVER Whisper, NEVER VAD/energy/RMS)
ffmpeg -nostdin -v error -y -i SRC.mp4 -ar 16000 -ac 1 -af dynaudnorm /tmp/a.wav
curl -s -X POST http://192.168.15.14:8101/transcribe -F "audio=@/tmp/a.wav" -o SRC_trans.json

# 2) SILENCE CUT — final_cut.py, LOCKED pad-start 0.0 / pad-end 0.3 (transcription-driven, nothing on top)
python3 <video-editing>/src/video/final_cut.py SRC.mp4 \
  --transcription SRC_trans.json --output PIECE_desil.mp4 \
  --pad-start 0.0 --pad-end 0.3 --crf 12 --preset veryfast

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
- Health: `curl -sf http://192.168.15.14:8100/health`. If down: `ssh server "cd <video-editing>/apps/sam3-api && docker compose up -d"`, wait ~30s for `{"status":"ready", "model":"facebook/sam3"}`.
- Detect: `curl -X POST http://192.168.15.14:8100/segment -F image=@frame.jpg -F prompt=face` → `bounding_box{center_x, center_y, height}`.
- Vertical crop at zoom Z: `cw=even(1215·(1−Z/100))`, `ch=even(2160·(1−Z/100))`, `x=even(center_cx−cw/2)` (clamp 0..3840−cw); for Z=0 `y=0` (full height); for Z>0 position face ~45% from top (`y=center_cy−0.45·ch`) clamped so `head_top=center_cy−0.65·h` stays inside (`y ≤ head_top−20`). Scale crop → `1080:1920,setsar=1`.

### Step 9 — A/B multicam (camera-runs, audio ALWAYS cam-A)
- Cut points = sentence boundaries from the **cam-A transcription** (`.!?`+pause>80ms / breath>200ms / comma; `MIN_CUT_INTERVAL`≈1–1.5s). Deterministic — the pipeline, not authored.
- Group windows into **CAMERA-RUNS** of 2–4 consecutive windows on the same camera; **switch cameras between runs** (the switch is the big event). The per-run camera choice is a CONTENT call → propose agent-reasoned, surface to the boss, never hardcode a blind A/B/A table.
- **AUDIO IS ALWAYS CAM-A** (better mic). cam-B = video only, frame-locked (Step 7).
- **Framing split:** cam-A windows = **static center per run** (re-center ONLY on a camera switch); cam-B windows = **static center + measured zoom** (Step 10). **Fluid head-follow (Step 10b) is OPT-IN — apply only if the CEO explicitly requests it; it is NEVER a default and NEVER a hard-gate.**

### Step 10 — Measurement-driven in-camera zoom (NEVER hardcode levels)
- `discover_framings` MEASURES the valid (zoom,position) framings per segment per camera (validates head-not-cut/face-visible). Run it; read the `_framing.json`:
  `python framing/discover_framings.py --segment-dir <dir with camA/ camB/ as .mov> --sam3-url http://192.168.15.14:8100 --ffmpeg /opt/homebrew/bin/ffmpeg`
- Pick zoom levels ONLY from the measured set. **Reality from the accepted run:** a CLOSE subject → cam-A has almost no room (`camA/body` validated only z15, `camA/cta` NONE) → **cam-A = z0 (no zoom).** cam-B (wide) had room (up to z50 on the hook, z15 on body). So **in-camera zoom lives on cam-B; cam-A is z0.**
- The 16:9 framing measure is a **PROXY** for the 9:16 crop. **VERIFY head-safety on the REAL 9:16 frames** (extract the rendered frame, SAM3, `face_top` must be > ~40px below the crop top) and **DROP** any level that crops the head in vertical.
- **Center is FIXED per run on CAM-B** (the run's median face) — only the crop SIZE steps between windows → no recenter creep. Zoom changes are clean **STEPS at speech boundaries** (hard step), not a continuous pan. (Cam-A framing baseline = static center per run; fluid head-follow is OPT-IN, Step 10b.)

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
  **de-escalate** (pull out) to reset after a climax / new topic. ≥10% zoom change per step.
- **Camera switch resets zoom** — the new run starts at a zoom that contrasts with where the previous
  run ended (the switch is the "breath".)
- **Per-shot crop = the framing-JSON crop** (`crop_x/y/w/h`, **center positions only**, `pos_key=="center"`),
  scaled to 3840×2160. **z0 = full frame** (`crop_w=3840, crop_h=2160`, a no-op crop).
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
- **Hardware DECODE matters as much as ENCODE:** `-hwaccel videotoolbox` before **EVERY** `-i`, plus
  `-c:v h264_videotoolbox -b:v 40M` (or `hevc_videotoolbox -q:v 65 -tag:v hvc1`) on output. Measured
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
- **NEVER `-c copy` concat separately-encoded HEVC clips** → **green frames** at joins (`hvc1` keeps
  SPS/PPS/VPS out-of-band, undecodable past clip 1). Use the **concat FILTER** (each input decoded with
  its own params) **or ProRes intermediates** (intra-only → every frame a keyframe → concats with
  `-c copy`).
- **Final concat goes DIRECTLY to H.264 MP4** — never an intermediate ProRes concat (~28 GB for 12 min;
  fills mac-pro's disk). Per-segment ProRes is fine (temporary, 1–3 GB each).
- **Black-first-frame PTS realign after concat** (same as the other flows): shift video start→0 and cap
  audio to the video duration — `ffprobe` the v:0 `start_time`, `-itsoffset $VS -i in … -map 0:v -map 1:a
  -c copy -avoid_negative_ts make_zero -t $VD`. Verify v:0 `start_time == a:0 == 0`, `dur(v) ≥ dur(a)`.

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

- **(5) SAM3 head-safe crop — the exact math (`valid_crop`), why nothing gets decapitated.** Per window,
  sample **2 frames** (start+0.2 s and end−0.2 s) → SAM3 `:8100` `prompt=face` → highest-confidence box.
  Run SAM3 on a 1280×720 downscale and scale coords **×3** back to 3840×2160. From the two boxes:
  `fx_lo,fx_hi` = min/max face-center-x, `fh` = max face height, `ftop` = min top-y,
  `crown = ftop − 0.30·fh`. A zoom `z` (crop `cw=3840·(1−z/100)`, `ch=2160·(1−z/100)`) is **valid only if**:
  ```
  cw >= (fx_hi − fx_lo) + 1.15*fh + 80     # horizontal: face span + ~1.15 head-widths + 80px
  ch >= 1.9*fh + 60                        # vertical:   ~1.9 head-heights (head+shoulders) + 60px
  ```
  Pick the **largest** `z` in `[0,15,20,25,35]` that is ≤ the run's requested zoom AND passes. Centre on
  the face, clamp the top so the crown stays in: `cx = clamp(cxc − cw/2, 0, 3840−cw)`,
  `cy = clamp(min(fy − 0.36·ch, crown − 0.04·ch), 0, 2160−ch)`, **all dims even**.
  **The error it avoids:** a hardcoded zoom (or centring on the body) shears the crown off a tall /
  gesturing subject. Measuring per window and gating on the head box made **245/245** v8 windows
  head-safe. Sample **both** ends of the window — the head moves within a 2–4 s shot; one frame misses it.

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
- **Landscape 3840×2160**, **audio Cam A only**, **A/V locked** (`dur(v) ≥ dur(a)`), clean joins, **no
  black first frame**.
- **Edit rules hold:** camera runs 2–4, zoom progresses within each run and resets on switch, every angle
  exists in the measured framing JSON, head-safe (SAM3 face inside crop). **TBP (if used):** exactly 3
  words, each window kept as ONE concat entry (validated 572/572 frames, 0 PTS diff vs no-TBP).

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
