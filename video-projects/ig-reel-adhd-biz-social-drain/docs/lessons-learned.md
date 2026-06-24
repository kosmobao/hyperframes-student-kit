# Lessons Learned — ig-reel-adhd-biz-social-drain

First production run of the video-use + Hyperframes talking-head pipeline. These are reusable learnings for future projects.

---

## Footage & Source Media

**Android phone footage has rotated dimensions.**
Android records landscape-dimension video (e.g. 1920×1080) with a 90° rotation tag in the display matrix. FFmpeg's `-show_entries` does not populate `side_data`, so the rotation is invisible and portrait detection fails — producing garbled output like 1920×3414. Use `-show_streams` instead, which populates `side_data_list` fully and makes the rotation visible. The `render.py` portrait detection (`is_portrait_source()`) was patched for this; treat it as a local patch that needs checking before merging any upstream update.

---

## Transcription

**ElevenLabs API key only needs Speech-to-Text permission.**
When using ElevenLabs Scribe for video-use transcription, the API key only requires the Speech-to-Text scope — not a full production key. Word-level timestamps are accurate enough for tight cut editing.

---

## Caption Rendering

**Custom PIL rendering is required for per-word colour highlighting and font switching.**
Standard SRT/ASS subtitle formats can't express mid-phrase colour changes (e.g. one highlighted word in Fraunces rust, rest in Atkinson cream). Custom PIL-based frame rendering is needed for this. Build caption images as PNGs per cue, then assemble into a WebM overlay.

**Always preview caption and hook styling on a single frame before rendering the full overlay track.**
Rendering a full 88s VP9 WebM just to check caption style wastes minutes. Extract one representative PNG from the cue sequence first — confirm font, colour, size, and position are correct before committing to the full encode.

---

## VP9 / WebM Overlays in FFmpeg

**VP9 WebM with alpha transparency requires the `libvpx-vp9` decoder, not the native `vp9` decoder.**
When overlaying a WebM with an alpha channel (`alpha_mode=1` tag in stream metadata), FFmpeg's native `vp9` decoder ignores the alpha plane — the overlay renders as opaque black. Specify `-c:v libvpx-vp9` BEFORE the `-i` flag on each WebM input to use the libvpx decoder, which correctly surfaces the alpha.

```bash
# Correct — libvpx-vp9 honours alpha
ffmpeg -i base.mp4 -c:v libvpx-vp9 -i overlay.webm ...

# Wrong — native vp9 ignores alpha, overlay renders solid black
ffmpeg -i base.mp4 -i overlay.webm ...
```

**VP9 encoding at default quality settings times out on long overlay tracks.**
Encoding an 88s VP9 WebM (captions overlay) at default `-deadline good` can time out. Use `-deadline realtime -cpu-used 8` for overlay tracks — the visual difference is negligible on a mostly-transparent track, and encode time drops from minutes to seconds.

**`eof_action=pass` is required when a short overlay sits on a long base video.**
The hook overlay is 2.5s on an 88.2s video. Without `eof_action=pass` on the overlay filter, FFmpeg holds the last frame of the hook for the remaining 85s. Set `eof_action=pass` so the base video passes through unchanged after the overlay stream ends.

---

## Composite Workflow

**Always extract a test frame before a full composite render.**
The full composite (grade + hook + captions + base video) takes several minutes. Before committing, extract one frame from the filter_complex to verify all layers are compositing correctly:

```bash
ffmpeg -ss 15 -i base.mp4 -c:v libvpx-vp9 -ss 15 -i captions.webm \
  -filter_complex "[0:v]<grade>[g]; [g][1:v]overlay=format=auto[out]" \
  -map "[out]" -frames:v 1 -update 1 -q:v 2 test_t15.png
```

**The correct composite layer order is: base → grade → hook overlay → captions overlay.**
Grade goes in the filter_complex on the base video stream, not as a separate pass. Applying it separately and then overlaying creates an extra encode generation and risks sync drift.

---

## Tooling

**Sub-agents help manage context in long rendering sessions.**
When a session accumulates many tool calls (ffprobe, ffmpeg, frame reads), spawning a sub-agent for the composite step keeps the main context clean and avoids hitting context limits mid-render.

**`tools/remove-silence.sh` is now redundant as a primary tool.**
The silence-removal FFmpeg script at `tools/remove-silence.sh` predates the video-use pipeline. video-use handles filler removal more accurately (word-level transcript-guided cuts vs. amplitude thresholding). Keep the script as a lightweight fallback for quick jobs that don't need ElevenLabs, but don't use it as the primary cut method.
