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
| Ken Burns motion | Enabled (auto) | Cycles zoom-in → zoom-out → pan L→R → pan R→L per slide; pan styles aim toward detected faces |

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

`cd` into the photo directory first, then create the virtual environment there. All relative paths in the generated script resolve from this directory, and the output video lands next to the photos by default.

```bash
cd /path/to/photos
python3 -m venv slideshow-env
slideshow-env/bin/pip install -q Pillow pillow-heif opencv-python-headless
```

`pillow-heif` adds HEIC support (iPhone photos); it's harmless to install even if not needed.

### Step 3: Generate and run the script

Write `slideshow-env/generate_slideshow.py` (see template below) with the user's settings filled in, then run it from the photo directory via the venv:

```bash
slideshow-env/bin/python slideshow-env/generate_slideshow.py
```

Placing the script inside `slideshow-env/` keeps the photo directory clean and prevents it being accidentally committed.

### Step 4: Report result

Confirm the output file path and total duration (number of photos × slide duration).

---

## Script template

Write this as `slideshow-env/generate_slideshow.py`, with the configuration values filled in:

```python
#!/usr/bin/env python3
"""Photography slideshow video generator.
Produces a 4K MP4 with each photo centred on a gallery matt.
Crossfades are pre-baked in PIL; FFmpeg uses the concat demuxer with a single
input stream — avoids OOM on large slideshows that a deep xfade filter chain causes.
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
from concurrent.futures import ProcessPoolExecutor, as_completed

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
MATT_COLOUR  = (245, 245, 240)     # RGB — warm white #F5F5F0
PADDING      = 0.10                # fraction of frame for padding on each side
SLIDE_SECS   = 5.0                 # seconds per slide
FADE_SECS    = 0.5                 # crossfade duration (0 = hard cut)
FPS          = 30
SORT_BY      = "date"              # "date" = EXIF creation date; "filename" = alphabetical
AUDIO_FILES  = []                  # list of MP3 paths — leave empty for silent video
KEN_BURNS    = True                # slow pan-and-zoom on each slide
KB_ZOOM_MAX  = 1.08                # max zoom factor (1.0 = none, 1.08 = 8%)
KB_STYLE     = "auto"              # "zoom_in" | "zoom_out" | "pan_lr" | "pan_rl" | "auto"
KB_FACE_DETECT = True              # pan toward detected faces when True
# ───────────────────────────────────────────────────────────────────────────────

SUPPORTED = {'.jpg', '.jpeg', '.png', '.heic', '.heif', '.tiff', '.tif', '.webp'}


def get_photo_date(path: Path):
    try:
        img = Image.open(path)
        exif = img.getexif()
        date_str = exif.get(36867) or exif.get(306)
        if date_str:
            return datetime.strptime(date_str, "%Y:%m:%d %H:%M:%S")
    except Exception:
        pass
    # Fall back to file system date (st_birthtime on macOS, st_mtime elsewhere)
    try:
        st = path.stat()
        ts = getattr(st, "st_birthtime", None) or st.st_mtime
        return datetime.fromtimestamp(ts)
    except Exception:
        pass
    return None


def get_photos(directory: str) -> list[Path]:
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
    undated.sort(key=lambda p: p[1].name)

    if undated:
        print(f"  {len(undated)} photo(s) have no date metadata and will appear at the end:")
        for _, p in undated:
            print(f"    - {p.name}")

    return [p for _, p in dated] + undated


def load_photo(path: Path) -> Image.Image:
    img = Image.open(path)
    img = ImageOps.exif_transpose(img)
    return img.convert("RGB")


def _avail_dims():
    pad_x = int(FRAME_W * PADDING)
    pad_y = int(FRAME_H * PADDING)
    return FRAME_W - 2 * pad_x, FRAME_H - 2 * pad_y, pad_x, pad_y


def make_frame_kb(photo_path: Path) -> Image.Image:
    """Prepare an oversized source image for Ken Burns frame rendering.
    Sized at fit_w*KB_ZOOM_MAX × fit_h*KB_ZOOM_MAX, preserving aspect ratio.
    """
    avail_w, avail_h, _, _ = _avail_dims()
    photo = load_photo(photo_path)

    fit_ratio = min(avail_w / photo.width, avail_h / photo.height)
    fit_w = round(photo.width * fit_ratio)
    fit_h = round(photo.height * fit_ratio)

    src_w = round(fit_w * KB_ZOOM_MAX)
    src_h = round(fit_h * KB_ZOOM_MAX)

    ratio = max(src_w / photo.width, src_h / photo.height)
    scaled_w = round(photo.width * ratio)
    scaled_h = round(photo.height * ratio)
    photo = photo.resize((scaled_w, scaled_h), Image.LANCZOS)

    x_off = (scaled_w - src_w) // 2
    y_off = (scaled_h - src_h) // 2
    return photo.crop((x_off, y_off, x_off + src_w, y_off + src_h))


def _detect_hw_encoder() -> str:
    result = subprocess.run(
        ["ffmpeg", "-hide_banner", "-encoders"],
        capture_output=True, text=True
    )
    return "hevc_videotoolbox" if "hevc_videotoolbox" in result.stdout else "libx265"


def _detect_subject(slide_path: Path):
    if not KB_FACE_DETECT:
        return None
    try:
        import cv2
    except ImportError:
        return None

    img = cv2.imread(str(slide_path), cv2.IMREAD_GRAYSCALE)
    if img is None:
        return None

    cascade_path = cv2.data.haarcascades + "haarcascade_frontalface_default.xml"
    detector = cv2.CascadeClassifier(cascade_path)

    scale = 960 / img.shape[1]
    small = cv2.resize(img, (960, int(img.shape[0] * scale)))
    faces = detector.detectMultiScale(small, scaleFactor=1.1, minNeighbors=5, minSize=(20, 20))

    if not len(faces):
        return None

    x, y, w, h = max(faces, key=lambda f: f[2] * f[3])
    cx = (x + w / 2) / scale
    cy = (y + h / 2) / scale
    src_w = img.shape[1]
    src_h = img.shape[0]
    return (cx / src_w, cy / src_h)


def make_ken_burns_frames(source_path: Path, slide_index: int,
                          subject_point, tmpdir: Path, slide_num: int) -> list[Path]:
    """Render every frame of a Ken Burns slide in PIL; returns list of frame paths."""
    avail_w, avail_h, pad_x, pad_y = _avail_dims()

    source       = Image.open(source_path).convert("RGB")
    src_w, src_h = source.size
    fit_w = round(src_w / KB_ZOOM_MAX)
    fit_h = round(src_h / KB_ZOOM_MAX)

    d         = int(SLIDE_SECS * FPS)
    excursion = KB_ZOOM_MAX - 1.0

    style_cycle = ["zoom_in", "zoom_out", "pan_lr", "pan_rl"]
    style = style_cycle[slide_index % 4] if KB_STYLE == "auto" else KB_STYLE

    matt_base = Image.new("RGB", (FRAME_W, FRAME_H), MATT_COLOUR)
    frame_paths = []

    for f in range(d):
        t = f / max(d - 1, 1)
        t_eased = (1 - math.cos(math.pi * t)) / 2

        if style == "zoom_in":
            zoom = 1.0 + excursion * t_eased
        elif style == "zoom_out":
            zoom = KB_ZOOM_MAX - excursion * t_eased
        else:
            zoom = 1.0 + excursion / 2

        crop_w = round(src_w / zoom)
        crop_h = round(src_h / zoom)

        if subject_point is not None:
            fx, fy = subject_point
            face_cx = fx * src_w
            face_cy = fy * src_h
            if style == "zoom_in":
                # Lead viewer's eye from image centre toward the face as we zoom in
                cx = src_w / 2 + (face_cx - src_w / 2) * t_eased
                cy = src_h / 2 + (face_cy - src_h / 2) * t_eased
            elif style == "zoom_out":
                # Reveal outward from the face toward the full scene
                cx = face_cx + (src_w / 2 - face_cx) * t_eased
                cy = face_cy + (src_h / 2 - face_cy) * t_eased
            else:  # pan_lr / pan_rl — pan toward face
                cx_min, cx_max = crop_w / 2.0, src_w - crop_w / 2.0
                cy_min, cy_max = crop_h / 2.0, src_h - crop_h / 2.0
                target_cx = max(cx_min, min(cx_max, face_cx))
                target_cy = max(cy_min, min(cy_max, face_cy))
                cx = src_w / 2 + (target_cx - src_w / 2) * t_eased
                cy = src_h / 2 + (target_cy - src_h / 2) * t_eased
        elif style == "pan_lr":
            drift = (src_w - crop_w) * 0.4
            cx    = src_w / 2 - drift / 2 + drift * t_eased
            cy    = src_h / 2
        elif style == "pan_rl":
            drift = (src_w - crop_w) * 0.4
            cx    = src_w / 2 + drift / 2 - drift * t_eased
            cy    = src_h / 2
        else:  # zoom_in / zoom_out, no face detected
            cx, cy = src_w / 2, src_h / 2

        x1 = round(cx - crop_w / 2)
        y1 = round(cy - crop_h / 2)
        x1 = max(0, min(x1, src_w - crop_w))
        y1 = max(0, min(y1, src_h - crop_h))

        crop        = source.crop((x1, y1, x1 + crop_w, y1 + crop_h))
        photo_frame = crop.resize((fit_w, fit_h), Image.LANCZOS)

        paste_x = pad_x + (avail_w - fit_w) // 2
        paste_y = pad_y + (avail_h - fit_h) // 2
        frame = matt_base.copy()
        frame.paste(photo_frame, (paste_x, paste_y))
        out_path = tmpdir / f"slide_{slide_num:04d}_f{f:04d}.jpg"
        frame.save(out_path, "JPEG", quality=85)
        frame_paths.append(out_path)

    return frame_paths


def _render_slide_task(args):
    """Top-level worker so ProcessPoolExecutor can pickle it on macOS (spawn mode)."""
    photo_path, slide_index, tmpdir_str = args
    photo  = Path(photo_path)
    tmpdir = Path(tmpdir_str)
    source_frame = make_frame_kb(photo)
    source_path  = tmpdir / f"slide_{slide_index:04d}_src.jpg"
    source_frame.save(source_path, "JPEG", quality=95)
    sp    = _detect_subject(source_path)
    paths = make_ken_burns_frames(source_path, slide_index, sp, tmpdir, slide_index)
    source_path.unlink(missing_ok=True)  # no longer needed; free space
    return slide_index, paths


def make_crossfade_frames(frames_a: list[Path], frames_b: list[Path],
                          fade_frames: int, tmpdir: Path, label: str) -> list[Path]:
    """Blend the tail of slide A with the head of slide B into crossfade frames."""
    tail_a = frames_a[-fade_frames:]
    head_b = frames_b[:fade_frames]
    out_paths = []
    for f, (pa, pb) in enumerate(zip(tail_a, head_b)):
        alpha = (f + 1) / (fade_frames + 1)  # excludes 0 and 1 endpoints
        img_a = Image.open(pa).convert("RGB")
        img_b = Image.open(pb).convert("RGB")
        blended = Image.blend(img_a, img_b, alpha)
        out_path = tmpdir / f"fade_{label}_f{f:04d}.jpg"
        blended.save(out_path, "JPEG", quality=85)
        out_paths.append(out_path)
    return out_paths


def build_concat_file(all_slide_frames: list[list[Path]], tmpdir: Path) -> Path:
    """Write a concat demuxer file assembling all frames with pre-baked crossfades.

    Crossfades replace the tail of slide i and head of slide i+1 with blended frames,
    keeping a single continuous frame stream that FFmpeg reads as one input — avoiding
    the per-slide input limit that causes OOM with large slideshows.
    """
    n = len(all_slide_frames)
    fade_frames = max(1, round(FADE_SECS * FPS)) if FADE_SECS > 0 and n > 1 else 0
    frame_dur = 1.0 / FPS

    concat_path = tmpdir / "frames.txt"
    with open(concat_path, "w") as f:
        for i, slide_frames in enumerate(all_slide_frames):
            is_last = (i == n - 1)
            plain_end   = len(slide_frames) if is_last else len(slide_frames) - fade_frames
            plain_start = 0 if i == 0 else fade_frames

            for frame_path in slide_frames[plain_start:plain_end]:
                f.write(f"file '{frame_path}'\nduration {frame_dur}\n")

            if not is_last:
                label = f"{i:04d}_{i+1:04d}"
                xfade = make_crossfade_frames(slide_frames, all_slide_frames[i + 1],
                                              fade_frames, tmpdir, label)
                for frame_path in xfade:
                    f.write(f"file '{frame_path}'\nduration {frame_dur}\n")

        # concat demuxer requires the last entry listed twice (duration quirk)
        if all_slide_frames:
            f.write(f"file '{all_slide_frames[-1][-1]}'\n")

    return concat_path


def get_audio_duration_secs(path: Path) -> float:
    result = subprocess.run(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "default=noprint_wrappers=1:nokey=1", str(path)],
        capture_output=True, text=True
    )
    return float(result.stdout.strip())


def encode_video(concat_path: Path, output: str, actual_video_secs: float, encoder: str):
    """Encode from concat demuxer file with optional audio. Streams FFmpeg stderr live."""
    cmd = ["ffmpeg", "-y", "-f", "concat", "-safe", "0", "-i", str(concat_path)]

    audio_inputs = []
    audio_filter = None
    if AUDIO_FILES:
        total_audio_secs = sum(get_audio_duration_secs(Path(f)) for f in AUDIO_FILES)
        loop_count = math.ceil(actual_video_secs / total_audio_secs) + 1
        repeated_audio = list(AUDIO_FILES) * loop_count
        for af in repeated_audio:
            audio_inputs += ["-i", str(af)]
        n_audio = len(repeated_audio)
        audio_map_str = "".join(f"[{1 + i}:a]" for i in range(n_audio))
        audio_filter = (
            f"{audio_map_str}concat=n={n_audio}:v=0:a=1[concat_audio];"
            f"[concat_audio]atrim=duration={actual_video_secs},asetpts=PTS-STARTPTS[aout]"
        )

    cmd += audio_inputs

    if encoder == "hevc_videotoolbox":
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
        cmd += ["-filter_complex", audio_filter]
        encode_flags += ["-map", "0:v", "-map", "[aout]", "-acodec", "aac", "-b:a", "192k"]
    else:
        encode_flags += ["-map", "0:v"]

    encode_flags += ["-movflags", "+faststart", output]
    cmd += encode_flags

    # Stream stderr live so progress is visible and the full error appears on failure
    proc = subprocess.Popen(cmd, stderr=subprocess.PIPE, text=True)
    stderr_lines = []
    for line in proc.stderr:
        print(line, end="", flush=True)
        stderr_lines.append(line)
    proc.wait()

    if proc.returncode != 0:
        print("\nFFmpeg failed. Last output:", file=sys.stderr)
        print("".join(stderr_lines[-30:]), file=sys.stderr)
        sys.exit(1)


def main():
    photos = get_photos(PHOTO_DIR)
    if not photos:
        print(f"No supported photos found in: {PHOTO_DIR}", file=sys.stderr)
        sys.exit(1)

    encoder = _detect_hw_encoder()
    n = len(photos)
    print(f"Found {n} photo(s) -> {OUTPUT_FILE}")

    fade_frames = max(1, round(FADE_SECS * FPS)) if FADE_SECS > 0 and n > 1 else 0
    total_frames = n * int(SLIDE_SECS * FPS) - max(0, n - 1) * fade_frames
    actual_video_secs = total_frames / FPS
    print(f"Duration: {actual_video_secs:.1f}s  |  Resolution: {FRAME_W}x{FRAME_H}  |  Encoder: {encoder}")

    tmpdir = Path(tempfile.mkdtemp(prefix="slideshow_"))
    try:
        all_slide_frames: dict[int, list[Path]] = {}

        n_workers = min(n, os.cpu_count() or 4)
        args_list = [(str(photo), i, str(tmpdir)) for i, photo in enumerate(photos)]

        with ProcessPoolExecutor(max_workers=n_workers) as executor:
            future_to_idx = {executor.submit(_render_slide_task, a): a[1] for a in args_list}
            done = 0
            for future in as_completed(future_to_idx):
                idx, frame_paths = future.result()
                all_slide_frames[idx] = frame_paths
                done += 1
                print(f"  [{done}/{n}] Slide {idx + 1} done")

        ordered_frames = [all_slide_frames[i] for i in range(n)]

        print("Building frame sequence with pre-baked crossfades ...")
        concat_path = build_concat_file(ordered_frames, tmpdir)

        print("Encoding video ...")
        encode_video(concat_path, OUTPUT_FILE, actual_video_secs, encoder)

    finally:
        shutil.rmtree(tmpdir, ignore_errors=True)

    print(f"\nDone! -> {OUTPUT_FILE}")


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
| "Force CPU encoding / smaller file" | Set `encoder = "libx265"` in `main()` before calling `encode_video` |
| "No fade / hard cut" | `FADE_SECS = 0` |
| "Slower fade" | `FADE_SECS = 1.0` |
| "Sort by filename" | `SORT_BY = "filename"` |
| "Include only landscape photos" | Filter in `get_photos()`: keep only photos where `width > height` after EXIF transpose (not yet implemented) |
| "Add background music" | `AUDIO_FILES = ["/path/to/track1.mp3", "/path/to/track2.mp3"]` |
| "No music / silent video" | `AUDIO_FILES = []` |
| "No Ken Burns / static slides" | `KEN_BURNS = False` |
| "Only zoom in on every slide" | `KB_STYLE = "zoom_in"` |
| "Gentler zoom" | `KB_ZOOM_MAX = 1.04` |
| "More dramatic zoom" | `KB_ZOOM_MAX = 1.25` |
| "Disable face detection" | `KB_FACE_DETECT = False` |

## Common issues

**Photos appear sideways** — `ImageOps.exif_transpose()` in the template fixes this. If still happening, the EXIF orientation tag is missing; manually rotate the source photos.

**HEIC files fail** — run `slideshow-env/bin/pip install pillow-heif` in the project directory.

**FFmpeg not found** — run `brew install ffmpeg`.

**Video file is very large** — when using `hevc_videotoolbox`, increase `-q:v` above 65 (e.g. `80`) for smaller files at some quality cost. When using `libx265`, increase CRF to 22–28. Hardware encoders produce slightly larger files than libx265 at the same perceived quality.

**Script is slow on many photos** — on Apple Silicon the script automatically uses `hevc_videotoolbox` (hardware encoder), which is ~2-3× faster. If still slow, consider reducing frame resolution during testing (`FRAME_W = 1920`, `FRAME_H = 1080`).

**Some photos have no creation date** — The script warns about these and places them at the end of the slideshow. To add EXIF dates, use a photo editor or `exiftool`. Alternatively, switch to filename sorting with `SORT_BY = "filename"`.

**Audio missing from output** — Ensure all paths in `AUDIO_FILES` exist before running; `ffprobe` will error on a missing file.

**Unsupported audio format** — The skill expects MP3. For AAC, WAV, or FLAC files, either ask the user to convert to MP3 first, or adjust the AUDIO_FILES entries accordingly (FFmpeg can decode most formats without code changes).

**Ken Burns shows a black border flash** — Means the zoom dropped below 1.0, exposing the canvas edge. Ensure `KB_ZOOM_MAX >= 1.0`. The default 1.08 with `"auto"` style will not trigger this.

**Face detection not working** — Face detection requires `opencv-python-headless`. Re-run the pip install step to confirm it installed. Detection runs on a 960px-wide downscale of each slide (~50ms per photo). Set `KB_FACE_DETECT = False` to disable it entirely.

**Video file corrupt / "moov atom not found"** — FFmpeg was killed before it could finish writing. Do NOT use an xfade filter chain for more than a handful of slides: each slide becomes a separate FFmpeg input stream and the chain grows O(n) in memory, causing OOM on large slideshows. The template above avoids this entirely by pre-baking crossfades in PIL and using the concat demuxer (single input stream). If you encounter this with a hand-edited script that uses xfade, revert to the template approach.
