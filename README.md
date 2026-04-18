# video-subtitle

A Claude Code skill that adds subtitles to videos using local Whisper speech-to-text and ffmpeg.

## What it does

1. Extracts audio from your video
2. Transcribes speech using a local Whisper model (no API calls, fully offline)
3. Generates SRT subtitle files with automatic cleanup (hallucination removal, proper noun correction)
4. Burns hardcoded subtitles into the video with ffmpeg

## Install

```bash
claude install gh:Tofuswang/video-subtitle-skill
```

## Prerequisites

### Whisper (speech-to-text)

**Apple Silicon Mac (recommended):**
```bash
pip install mlx-whisper
```

**Any platform:**
```bash
pip install openai-whisper
```

### ffmpeg with subtitle support

**macOS:**
```bash
brew install ffmpeg-full
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install ffmpeg
```

> Note: The default `brew install ffmpeg` on macOS does **not** include `libass`, which is required for the `subtitles` filter. You need `ffmpeg-full`.

## Usage

```
/video-subtitle path/to/video.mov          # Full pipeline: transcribe + burn subtitles
/video-subtitle srt path/to/video.mov      # Generate SRT file only
/video-subtitle burn video.mov sub.srt     # Burn existing SRT into video
```

## Supported languages

Chinese, English, Japanese, and [90+ languages supported by Whisper](https://github.com/openai/whisper#available-models-and-languages). The skill will ask you to confirm the language before transcribing.

## How it works

- Audio is extracted as 16kHz mono WAV (optimal for Whisper)
- Whisper runs entirely on your machine — no data leaves your computer
- The skill auto-corrects common Whisper mistakes (hallucinated silence, proper noun errors) and lets you review before burning
- Subtitles are burned with white text, black outline, positioned at the bottom

## License

MIT
