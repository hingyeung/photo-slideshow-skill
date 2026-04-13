---
name: photo-slideshow
description: Creates a photography slideshow video from a folder of photos. Use this skill whenever the user wants to make a slideshow, photo video, or turn a folder of photos into a video — including when they want to change timing, resolution, matt colour, transition style, or other slideshow settings. Also use when the user says things like "make a video from my photos", "export as a video", or "create a slideshow". Handles portrait, landscape, and square photos automatically, centering each on a warm gallery matt background.
---

# Photo Slideshow Skill

This skill produces a 4K MP4 video from a folder of photos. Each photo is displayed centred on a neutral gallery matt — like a fine-art print on presentation board — with even padding on all four sides. The output is a portable H.265 video that plays on any TV, phone, or computer.

## Default style

| Setting | Default | Notes |
|---|---|---|
| Matt colour | `#F5F5F0` (warm white) | Evokes fine-art paper; non-distracting on large screens |
| Padding | 10% on each side | Photo fills the central 80% of the frame |
| Duration per slide | 5 seconds | |
| Transition | 0.5s crossfade | `0` = hard cut |
| Resolution | 3840×2160 (4K UHD) | |
| Encoding | H.265 (HEVC) via hardware if available | Uses `hevc_videotoolbox` on Apple Silicon for ~2-3× speedup; falls back to `libx265` elsewhere |
| Frame rate | 30 fps | |
| Sort order | EXIF creation date (oldest first) | Photos without a timestamp appear at the end |
| Audio | _(none — silent)_ | List of MP3 paths; audio is concatenated and looped/trimmed to match video duration |

The photo is always shown at its natural aspect ratio — never cropped or stretched.

## Workflow

### Step 1: Gather requirements

Ask the user (or infer from context) for:
- **Photo folder path** — required
- **Output file path** — default: `slideshow.mp4` in the same directory as the photos
- **Any style overrides** — matt colour, timing, resolution, transitions
- **Audio files** — list of MP3 paths (optional). If omitted, video is silent.

### Step 2: Check FFmpeg and set up venv

```bash
which ffmpeg || brew install ffmpeg
```

Create a virtual environment next to the script so dependencies don't touch the system Python:

```bash
python3 -m venv slideshow-env
slideshow-env/bin/pip install -q Pillow pillow-heif
```

`pillow-heif` adds HEIC support (iPhone photos); it's harmless to install even if not needed.

### Step 3: Generate and run the script

Write `slideshow-env/generate_slideshow.py` (see template below) with the user's settings filled in, then run it via the venv:

```bash
slideshow-env/bin/python slideshow-env/generate_slideshow.py
```

Placing the script inside `slideshow-env/` keeps the user's working directory clean and prevents it being accidentally committed.

### Step 4: Report result

Confirm the output file path and total duration (number of photos × slide duration).

---

## Script template

Write this as `slideshow-env/generate_slideshow.py`, with the configuration values filled in:

```python
#!/usr/bin/env python3
"""Photography slideshow video generator.
Produces a 4K MP4 with each photo centred on a gallery matt.
"""

import math
import os
import sys
import subprocess
import tempfile
import shutil
from pathlib import Path
from datetime import datetime
from PIL import Image, ImageOps

# ── HEIC support (installed into venv via pip install pillow-heif) ─────────────
try:
    from pillow_heif import register_heif_opener
    register_heif_opener()
except ImportError:
    pass  # Silently skip — JPEG/PNG will still work

# ── Configuration (filled in by Claude) ────────────────────────────────────────
PHOTO_DIR    = "/path/to/photos"   # folder containing photos
OUTPUT_FILE  = "slideshow.mp4"     # output video path
FRAME_W      = 3840                # frame width  (3840 = 4K, 1920 = 1080p)
FRAME_H      = 2160                # frame height (2160 = 4K, 1080 = 1080p)
MATT_COLOUR  = (245, 245, 240)     # RGB — warm white #F5F5F0  ← change this for different background colour
PADDING      = 0.10                # fraction of frame for padding on each side (10% = gallery border)
SLIDE_SECS   = 5.0                 # seconds per slide
FADE_SECS    = 0.5                 # crossfade duration (0 = hard cut)
FPS          = 30
SORT_BY      = "date"              # "date" = EXIF creation date (oldest first); "filename" = alphabetical
AUDIO_FILES  = []                  # list of MP3 paths — leave empty for silent video
# ───────────────────────────────────────────────────────────────────────────────

SUPPORTED = {'.jpg', '.jpeg', '.png', '.heic', '.heif', '.tiff', '.tif', '.webp'}


def get_photo_date(path: Path):
    """Extract EXIF DateTimeOriginal from a photo, or None if unavailable."""
    try:
        img = Image.open(path)
        exif = img.getexif()
        # Try DateTimeOriginal (tag 36867) first, fall back to DateTime (tag 306)
        date_str = exif.get(36867) or exif.get(306)
        if date_str:
            return datetime.strptime(date_str, "%Y:%m:%d %H:%M:%S")
    except Exception:
        pass
    return None


def get_photos(directory: str) -> list[Path]:
    """Return photos sorted by EXIF creation date (oldest first) or by filename.
    When sorting by date, photos without a creation timestamp are placed at the end.
    """
    all_photos = [
        p for p in Path(directory).iterdir()
        if p.suffix.lower() in SUPPORTED and not p.name.startswith('.')
    ]

    if SORT_BY == "filename":
        return sorted(all_photos, key=lambda p: p.name)

    dated = []
    undated = []
    for p in all_photos:
        dt = get_photo_date(p)
        if dt:
            dated.append((dt, p))
        else:
            undated.append(p)

    dated.sort(key=lambda x: x[0])
    undated.sort(key=lambda x: x.name)

    if undated:
        print(f"  ⚠ {len(undated)} photo(s) have no EXIF creation date and will appear at the end:")
        for p in undated:
            print(f"    - {p.name}")

    return [p for _, p in dated] + undated


def load_photo(path: Path) -> Image.Image:
    """Load image and apply EXIF orientation so it displays correctly."""
    img = Image.open(path)
    img = ImageOps.exif_transpose(img)
    return img.convert("RGB")


def make_frame(photo_path: Path) -> Image.Image:
    """Composite photo centred on matt canvas, preserving its aspect ratio."""
    canvas = Image.new("RGB", (FRAME_W, FRAME_H), MATT_COLOUR)
    photo = load_photo(photo_path)

    pad_px_x = int(FRAME_W * PADDING)
    pad_px_y = int(FRAME_H * PADDING)
    avail_w = FRAME_W - 2 * pad_px_x
    avail_h = FRAME_H - 2 * pad_px_y

    # Scale to fill available area — always resize (upscale low-res photos too)
    ratio = min(avail_w / photo.width, avail_h / photo.height)
    photo = photo.resize((int(photo.width * ratio), int(photo.height * ratio)), Image.LANCZOS)

    x = (FRAME_W - photo.width) // 2
    y = (FRAME_H - photo.height) // 2
    canvas.paste(photo, (x, y))
    return canvas


def _detect_hw_encoder() -> str:
    """Return 'hevc_videotoolbox' if Apple Silicon HW encoder is available, else 'libx265'."""
    result = subprocess.run(
        ["ffmpeg", "-hide_banner", "-encoders"],
        capture_output=True, text=True
    )
    return "hevc_videotoolbox" if "hevc_videotoolbox" in result.stdout else "libx265"


def get_audio_duration_secs(path: Path) -> float:
    """Return duration of an audio file in seconds using ffprobe."""
    result = subprocess.run(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "default=noprint_wrappers=1:nokey=1", str(path)],
        capture_output=True, text=True
    )
    return float(result.stdout.strip())


def build_ffmpeg_cmd(slide_paths: list[Path], output: str, total_video_secs: float, encoder: str = "") -> list[str]:
    """Build FFmpeg command. Uses concat (hard cut) when FADE_SECS=0, xfade otherwise.
    If AUDIO_FILES is set, concatenates and loops audio to match video duration."""
    cmd = ["ffmpeg", "-y"]
    n = len(slide_paths)

    for p in slide_paths:
        cmd += ["-loop", "1", "-t", str(SLIDE_SECS), "-i", str(p)]

    # Actual video duration is shorter than n*SLIDE_SECS when xfade overlaps are applied
    if FADE_SECS > 0 and n > 1:
        actual_video_secs = round(n * SLIDE_SECS - (n - 1) * FADE_SECS, 3)
    else:
        actual_video_secs = total_video_secs

    # --- Audio inputs ---
    audio_inputs = []
    audio_filter = None
    if AUDIO_FILES:
        total_audio_secs = sum(get_audio_duration_secs(Path(f)) for f in AUDIO_FILES)
        # Repeat the playlist enough times to exceed actual video duration, then trim
        loop_count = math.ceil(actual_video_secs / total_audio_secs) + 1
        repeated_audio = list(AUDIO_FILES) * loop_count
        for f in repeated_audio:
            audio_inputs += ["-i", str(f)]
        n_audio = len(repeated_audio)
        # Audio inputs start after all video inputs (index n)
        audio_map_str = "".join(f"[{n + i}:a]" for i in range(n_audio))
        audio_filter = (
            f"{audio_map_str}concat=n={n_audio}:v=0:a=1[concat_audio];"
            f"[concat_audio]atrim=duration={actual_video_secs},asetpts=PTS-STARTPTS[aout]"
        )

    cmd += audio_inputs

    # --- Encoding flags ---
    if not encoder:
        encoder = _detect_hw_encoder()
    if encoder == "hevc_videotoolbox":
        # Apple Silicon hardware encoder — ~2-3× faster than libx265; no -preset support
        encode_flags = [
            "-vcodec", "hevc_videotoolbox", "-q:v", "65", "-tag:v", "hvc1",
            "-pix_fmt", "yuv420p", "-dn", "-map_chapters", "-1",
        ]
    else:
        encode_flags = [
            "-vcodec", "libx265", "-crf", "18", "-preset", "medium",
            "-pix_fmt", "yuv420p", "-dn", "-map_chapters", "-1",
        ]
    if audio_filter:
        encode_flags += ["-map", "[vout]", "-map", "[aout]", "-acodec", "aac", "-b:a", "192k"]
    else:
        encode_flags += ["-map", "[vout]"]
    encode_flags += ["-movflags", "+faststart", output]

    if n == 1 or FADE_SECS == 0:
        # Hard cut: use concat filter
        video_filter = f"concat=n={n}:v=1:a=0[vout]"
    else:
        # Crossfade: build xfade filter chain
        filter_parts = []
        last_label = "[0:v]"
        for i in range(1, n):
            offset = round(SLIDE_SECS * i - FADE_SECS * i, 3)
            out_label = f"[v{i}]" if i < n - 1 else "[vout]"
            filter_parts.append(
                f"{last_label}[{i}:v]xfade=transition=fade:duration={FADE_SECS}:offset={offset}{out_label}"
            )
            last_label = f"[v{i}]"
        video_filter = ";".join(filter_parts)

    full_filter = video_filter if not audio_filter else video_filter + ";" + audio_filter
    cmd += ["-filter_complex", full_filter] + encode_flags
    return cmd


def main():
    photos = get_photos(PHOTO_DIR)
    if not photos:
        print(f"No supported photos found in: {PHOTO_DIR}", file=sys.stderr)
        sys.exit(1)

    encoder = _detect_hw_encoder()
    print(f"Found {len(photos)} photo(s) → {OUTPUT_FILE}")
    total_video_secs = len(photos) * SLIDE_SECS
    print(f"Duration: {total_video_secs:.0f}s  |  Resolution: {FRAME_W}×{FRAME_H}  |  Matt: #{MATT_COLOUR[0]:02X}{MATT_COLOUR[1]:02X}{MATT_COLOUR[2]:02X}  |  Encoder: {encoder}")

    tmpdir = tempfile.mkdtemp(prefix="slideshow_")
    try:
        slide_paths = []
        for i, photo in enumerate(photos):
            print(f"  [{i+1}/{len(photos)}] Processing {photo.name} …")
            frame = make_frame(photo)
            slide_path = Path(tmpdir) / f"slide_{i:04d}.jpg"
            frame.save(slide_path, "JPEG", quality=95)
            slide_paths.append(slide_path)

        print("Encoding video …")
        cmd = build_ffmpeg_cmd(slide_paths, OUTPUT_FILE, total_video_secs, encoder)
        result = subprocess.run(cmd, capture_output=True, text=True)
        if result.returncode != 0:
            print("FFmpeg error:", result.stderr[-2000:], file=sys.stderr)
            sys.exit(1)

    finally:
        shutil.rmtree(tmpdir, ignore_errors=True)

    print(f"\nDone! → {OUTPUT_FILE}")


if __name__ == "__main__":
    main()
```

---

## Customisation map

| User request | What to change |
|---|---|
| "Slower / 8 seconds per slide" | `SLIDE_SECS = 8.0` |
| "Light grey background" | `MATT_COLOUR = (232, 232, 232)` |
| "Dark / cinema background" | `MATT_COLOUR = (26, 26, 26)` |
| "More border / more breathing room" | `PADDING = 0.15` |
| "Less border / photos feel larger" | `PADDING = 0.04` |
| "1080p output" | `FRAME_W = 1920`, `FRAME_H = 1080` |
| "Force CPU encoding / smaller file" | Pass `encoder="libx265"` to `build_ffmpeg_cmd`, or set `encoder` in `main()` before the call |
| "No fade / hard cut" | `FADE_SECS = 0` |
| "Slower fade" | `FADE_SECS = 1.0` |
| "Sort by filename" | `SORT_BY = "filename"` |
| "Include only landscape photos" | Filter in `get_photos()`: keep only photos where `width > height` after EXIF transpose (not yet implemented) |
| "Add background music" | `AUDIO_FILES = ["/path/to/track1.mp3", "/path/to/track2.mp3"]` |
| "No music / silent video" | `AUDIO_FILES = []` |

## Common issues

**Photos appear sideways** — `ImageOps.exif_transpose()` in the template fixes this. If still happening, the EXIF orientation tag is missing; manually rotate the source photos.

**HEIC files fail** — run `slideshow-env/bin/pip install pillow-heif` in the project directory.

**FFmpeg not found** — run `brew install ffmpeg`.

**Video file is very large** — when using `hevc_videotoolbox`, increase `-q:v` above 65 (e.g. `80`) for smaller files at some quality cost. When using `libx265`, increase CRF to 22–28. Hardware encoders produce slightly larger files than libx265 at the same perceived quality.

**Script is slow on many photos** — on Apple Silicon the script automatically uses `hevc_videotoolbox` (hardware encoder), which is ~2-3× faster. If still slow, consider reducing frame resolution during testing (`FRAME_W = 1920`, `FRAME_H = 1080`).

**Some photos have no creation date** — The script warns about these and places them at the end of the slideshow. To add EXIF dates, use a photo editor or `exiftool`. Alternatively, switch to filename sorting with `SORT_BY = "filename"`.

**Audio missing from output** — Ensure all paths in `AUDIO_FILES` exist before running; `ffprobe` will error on a missing file.

**Unsupported audio format** — The skill expects MP3. For AAC, WAV, or FLAC files, either ask the user to convert to MP3 first, or adjust the AUDIO_FILES entries accordingly (FFmpeg can decode most formats without code changes).
