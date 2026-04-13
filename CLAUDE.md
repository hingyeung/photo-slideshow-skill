# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code skill (plugin) that generates a 4K MP4 photo slideshow. It does not contain runnable application code — the primary artefacts are the skill definition and its evals.

## Key files

- `skills/photo-slideshow/SKILL.md` — the skill definition: instructions, Python script template, configuration map, and troubleshooting. This is the file to edit when changing skill behaviour.
- `skills/photo-slideshow/evals/evals.json` — eval test cases for the skill.
- `.claude-plugin/plugin.json` — plugin metadata for the Claude Code marketplace.
- `.claude-plugin/marketplace.json` — marketplace index (references `plugin.json`).

## Local testing setup

The eval runner looks for skills under `.claude/skills/`. Since `.claude/skills/` is gitignored, create a symlink after cloning:

```bash
ln -s ../../skills/photo-slideshow .claude/skills/photo-slideshow
```

Run evals via Claude Code's eval runner against `skills/photo-slideshow/evals/evals.json`.

## Skill behaviour

When invoked, the skill:
1. Creates a Python venv at `slideshow-env/` in the working directory
2. Installs `Pillow` and `pillow-heif` into the venv
3. Writes `slideshow-env/generate_slideshow.py` from the template in `SKILL.md` with user settings filled in
4. Runs the script to produce the MP4

The script template in `SKILL.md` is the source of truth for all configuration constants (`MATT_COLOUR`, `SLIDE_SECS`, `FADE_SECS`, `FRAME_W/H`, `SORT_BY`, etc.). Changes to default behaviour belong there.

## Prerequisites (for running evals)

- FFmpeg (`brew install ffmpeg`)
- Python 3.8+
- Test photos in `test_photos/` (gitignored; not redistributed)
