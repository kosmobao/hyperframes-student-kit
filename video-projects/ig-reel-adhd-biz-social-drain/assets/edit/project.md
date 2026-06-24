## Session 1 — 2026-06-25

**Strategy:** Single-take talking-head IG reel. Transcribe with ElevenLabs Scribe (word-level), pack to phrases, present full cut proposal for approval before rendering. Clean cut only — no grade, no captions yet. User approved all cuts; confirmed dramatic pauses tight (~0.15s before "crashes." and "Super tired."), kept both "the food" and "everything was good" detail lines.

**Decisions:**
- 18 segments extracted from raw-take-03.mp4 (2:02 raw → 1:28 clean cut)
- Fillers removed: "Um," (×2 occurrences), "busi-" false start, "en-" false start, "um, you know," (×3), "Mm." + 4.6s dead air, "And," trailing in outro, "Yeah." at end
- Structural simplification: cut fragmented "that I observed about this is that, you know," restart (~3.74s); cut isolated "that"/And fragments in battery explanation section; cut "also when," redundant restart before home scene
- Dramatic pauses tightened to ~0.15s before "crashes." and "Super tired." per user request (originally 1.24s and 1.90s)
- Two 5s dead-air gaps removed (after "Super tired." and before battery explanation)
- Audio normalised to -14 LUFS / -1 dBTP / LRA 11 (social-ready)

**Reasoning log:**
- Source has 90° rotation display matrix (Android phone, landscape-coded portrait footage). render.py `is_portrait_source()` was using `-show_entries` which returns empty side_data, so rotation was invisible and portrait detection failed, producing 1920×3414 output. Fixed by switching to `-show_streams` which populates `side_data_list` fully. Portrait scale (`scale=-2:1920`) then correctly yields 1080×1920.

**Output:**
- `edit/preview.mp4` — 1080×1920, 88.2s, H.264 CRF 22, -14 LUFS
- `edit/edl.json` — 18-segment EDL, grade: none
- `edit/transcripts/raw-take-03.json` — cached Scribe transcript (515 words)
- `edit/takes_packed.md` — phrase-level reading view

**Outstanding:**
- Possible further trim — at 1:28 it's on the long side for an IG reel; user may want a second pass to tighten

---

## Session 2 — 2026-06-25

**Tasks:** Fix composite (base video missing in previous render) + VP9 encoding timeout fix + verify final output.

**Bug 1 — black background in composite:**
- Root cause: previous render didn't include `preview.mp4` as the base layer — overlays were composited onto nothing
- Fix: VP9 WebM overlays with alpha require `-c:v libvpx-vp9` decoder flag placed BEFORE the `-i` on each WebM input. The native `vp9` decoder ignores the alpha plane; libvpx-vp9 renders it correctly
- Also required: `eof_action=pass` on the hook overlay so the main video passes through cleanly after the 2.5s hook stream ends

**Bug 2 — VP9 overlay encoding timeout:**
- Rendering captions.webm (88.2s VP9 with alpha) at default quality settings timed out
- Fix: `-deadline realtime -cpu-used 8` on the VP9 encoder for the overlay track — trades quality for speed on a track that's mostly transparent anyway

**Final composite command (key structure):**
```
ffmpeg -i preview.mp4 \
  -c:v libvpx-vp9 -i hook.webm \
  -c:v libvpx-vp9 -i captions.webm \
  -filter_complex "[0:v]<grade>[graded]; [graded][1:v]overlay=format=auto:eof_action=pass[with_hook]; [with_hook][2:v]overlay=format=auto[out]" \
  -map "[out]" -map 0:a -c:v libx264 -crf 18 -preset medium -c:a copy -t 88.2 preview-v2.mp4
```

**Output:**
- `edit/preview-v2.mp4` — 1080×1920, 88.2s, H.264 CRF 18, 73MB
- Hook text overlay (first 2.5s) + branded captions throughout + warm cinematic grade

**Verified frames:**
- t=1s: hook text ("Life of the party. Dead on the couch. Every time.") with dark vignette over base video
- t=15s: base video + warm grade + caption "energetic when she's around other"
- t=45s: base video + caption "Exhausted. Every time."

**Outstanding:**
- Possible further trim — at 1:28 it's on the long side for an IG reel; user may want a second pass to tighten
