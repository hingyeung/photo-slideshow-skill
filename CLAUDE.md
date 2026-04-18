# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code skill (plugin) that generates a 4K MP4 photo slideshow. It does not contain runnable application code — the primary artefacts are the skill definition and its evals.

## Key files

- `skills/photo-slideshow/SKILL.md` — the skill definition: step-by-step instructions, Python script template, configuration map, and troubleshooting. **This is the only file to edit when changing skill behaviour.**
- `skills/photo-slideshow/evals/evals.json` — eval test cases for the skill.
- `.claude-plugin/plugin.json` — plugin metadata for the Claude Code marketplace.
- `.claude-plugin/marketplace.json` — marketplace index (references `plugin.json`).

## Local testing setup

The eval runner looks for skills under `.claude/skills/`. Since `.claude/skills/` is gitignored, create a symlink after cloning:

```bash
ln -s ../../skills/photo-slideshow .claude/skills/photo-slideshow
```

Run evals via Claude Code's eval runner against `skills/photo-slideshow/evals/evals.json`.

To run a single eval by ID (e.g. eval `0`), filter by the `id` field in `evals.json` using the eval runner's `--filter` or equivalent flag.

### Prerequisites

- FFmpeg (`brew install ffmpeg`)
- Python 3.8+
- Test photos in `test_photos/` (gitignored; not redistributed)
- Test audio in `test_audio/track.mp3` (gitignored; used by eval 3)

## Skill behaviour

When invoked, the skill:
1. Asks the user for the photo folder, output path, and any style overrides
2. Checks FFmpeg is installed, then creates a Python venv at `slideshow-env/` in the photo directory
3. Installs `Pillow`, `pillow-heif`, and `opencv-python-headless` into the venv
4. Writes `slideshow-env/generate_slideshow.py` from the template in `SKILL.md` with user settings filled in
5. Runs the script to produce the MP4

## Script template architecture

The Python script template (embedded in `SKILL.md`) is the source of truth for all logic and defaults. Key sections:

**Configuration block** (filled in by the skill before running):
```
PHOTO_DIR, OUTPUT_FILE, FRAME_W/H, MATT_COLOUR, PADDING,
SLIDE_SECS, FADE_SECS, FPS, SORT_BY, AUDIO_FILES,
KEN_BURNS, KB_ZOOM_MAX, KB_STYLE, KB_FACE_DETECT
```

**Rendering pipeline:**
1. `get_photos()` — loads and sorts photos (by EXIF date or filename)
2. `make_ken_burns_frames()` — renders per-slide frames using `ProcessPoolExecutor` (parallel); each frame is a full-resolution PIL image with pan/zoom motion baked in
3. `make_crossfade_frames()` — pre-bakes crossfade blends between adjacent slides using PIL (avoids FFmpeg xfade OOM on large frame counts)
4. `build_concat_file()` — writes an FFmpeg concat demuxer file referencing the pre-baked PNG sequences
5. `encode_video()` — runs FFmpeg with hardware encoder detection: `hevc_videotoolbox` on Apple Silicon, fallback to `libx265`

**Face detection:** `_detect_subject()` uses OpenCV on a 960px downscale to find a face centre point, which biases the Ken Burns pan target.

**Audio:** If `AUDIO_FILES` is set, the script uses `ffprobe` to measure track duration and an FFmpeg `amix`/trim filter to loop or cut audio to match video length.

## Eval structure

Each eval in `evals.json` asserts:
- The output MP4 file exists and is over 1 MB
- Specific config values appear in the generated script (e.g. `MATT_COLOUR`, `FRAME_W`)
- Video duration meets a minimum (computed from slide count × `SLIDE_SECS`)

The five evals cover: default settings, custom colour/timing, 1080p preview, audio, and Ken Burns motion.
