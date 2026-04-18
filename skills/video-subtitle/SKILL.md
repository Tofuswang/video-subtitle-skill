---
name: video-subtitle
description: |
  Use when adding subtitles to a video file. Use when user says "上字幕",
  "add subtitles", "transcribe video", or provides a video file (.mov, .mp4,
  .mkv) and wants captions burned in. Also use when user wants to extract
  transcript from video with timestamps.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# /video-subtitle — Video Subtitle Tool

Transcribe video audio to SRT subtitles using a local Whisper model, then burn them into the video.

## User-invocable

When the user types `/video-subtitle`, run this skill.

## Prerequisites

This skill requires two tools. Check availability before proceeding:

### 1. mlx-whisper (Apple Silicon) or openai-whisper

Detect which is available:

```bash
which mlx_whisper whisper 2>/dev/null
```

**If neither is found**, guide the user to install one:

- **Apple Silicon Mac (recommended):** `pip install mlx-whisper`
- **Any platform:** `pip install openai-whisper`

### 2. ffmpeg with libass

Check for libass support:

```bash
ffmpeg -filters 2>&1 | grep subtitles
```

If no output (libass missing), the `subtitles` filter is unavailable:

- **macOS:** `brew install ffmpeg-full` (keg-only, use `/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg`)
- **Linux:** `sudo apt install ffmpeg` (usually includes libass)
- **Fallback:** If the user has regular ffmpeg without libass, soft-embed the SRT as a subtitle track instead of burning in.

### Whisper model selection

Discover cached models:

```bash
ls ~/.cache/huggingface/hub/ 2>/dev/null | grep whisper
```

Prefer the largest available model. If none cached, use:
- mlx-whisper: `mlx-community/whisper-large-v3-mlx`
- openai-whisper: `large-v3`

## Modes

- `/video-subtitle path/to/video.mov` → **Full pipeline** (transcribe + correct + burn)
- `/video-subtitle srt path/to/video.mov` → **SRT only** (transcribe, no burn)
- `/video-subtitle burn path/to/video.mov path/to/sub.srt` → **Burn only** (use existing SRT)
- `/video-subtitle` (no args) → Interactive, ask for video path

---

## Full Pipeline

### Step 1: Verify video

If no path provided, ask with AskUserQuestion. Confirm the file exists and read metadata:

```bash
ffmpeg -i "<video_path>" 2>&1 | grep -E "Duration|Stream.*(Video|Audio)"
```

Report duration, resolution, and file size to the user.

### Step 2: Extract audio

```bash
ffmpeg -i "<video_path>" -vn -acodec pcm_s16le -ar 16000 -ac 1 "<output_dir>/<basename>_audio.wav" -y
```

16kHz mono WAV is the optimal input format for Whisper.

### Step 3: Confirm language

Ask the user with AskUserQuestion. Common options:

| Language | Flag |
|----------|------|
| Chinese | `--language zh` |
| English | `--language en` |
| Japanese | `--language ja` |
| Auto-detect | omit `--language` |

### Step 4: Transcribe

**mlx-whisper:**
```bash
mlx_whisper --model "<model>" --language <lang> --output-format srt --output-dir "<output_dir>" "<audio_wav>"
```

**openai-whisper:**
```bash
whisper "<audio_wav>" --model large-v3 --language <lang> --output_format srt --output_dir "<output_dir>"
```

Tell the user: "Transcribing, please wait..."

### Step 5: Clean up SRT

Read the generated SRT and fix:

1. **Remove hallucinations**: Delete entries with zero-duration timestamps (e.g. `00:00:57,640 --> 00:00:57,640`) or empty text
2. **Fix proper nouns**: Whisper commonly misrecognizes brand names and technical terms — use conversation context to correct
3. **Merge fragments**: Combine entries shorter than 0.5s that are semantically connected
4. **Renumber**: Run this Python snippet to renumber all entries:

```python
python3 -c "
import re
with open('<srt_path>', 'r') as f:
    content = f.read()
blocks = re.split(r'\n\n+', content.strip())
out = []
for i, block in enumerate(blocks, 1):
    lines = block.strip().split('\n')
    if len(lines) >= 2:
        lines[0] = str(i)
        out.append('\n'.join(lines))
with open('<srt_path>', 'w') as f:
    f.write('\n\n'.join(out) + '\n')
"
```

Present the corrected SRT to the user and ask if further edits are needed before burning.

### Step 6: Burn subtitles

Determine the correct ffmpeg binary (prefer ffmpeg-full if regular ffmpeg lacks libass):

```bash
FFMPEG="ffmpeg"
if ! ffmpeg -filters 2>&1 | grep -q subtitles; then
  if [ -x /opt/homebrew/opt/ffmpeg-full/bin/ffmpeg ]; then
    FFMPEG="/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg"
  fi
fi
```

Then burn:

```bash
$FFMPEG \
  -i "<video_path>" \
  -vf "subtitles=<srt_path>:force_style='FontSize=24,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,Outline=2,MarginV=30'" \
  -c:v libx264 -preset medium -crf 23 \
  -c:a aac -b:a 128k \
  -pix_fmt yuv420p \
  "<output_dir>/<basename>_subtitled.mp4" -y
```

**force_style options:**

| Param | Default | Description |
|-------|---------|-------------|
| FontSize | 24 | Font size |
| PrimaryColour | &H00FFFFFF | White text |
| OutlineColour | &H00000000 | Black outline |
| Outline | 2 | Outline thickness |
| MarginV | 30 | Bottom margin |

**Important:** The SRT file path must not contain special characters that confuse the ffmpeg filter parser. Use relative paths or symlink if needed.

### Step 7: Done

Report the output file path and size. Suggest `open <path>` to preview. Remind the user that the intermediate `_audio.wav` file can be deleted.

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `No such filter: 'subtitles'` | ffmpeg missing libass | Use ffmpeg-full or install libass |
| `No such filter: 'drawtext'` | ffmpeg missing libfreetype | Use ffmpeg-full |
| Duplicate/empty SRT entries | Whisper hallucination | Clean in Step 5 |
| Chinese chars show as boxes | Missing CJK font | macOS: auto-fallback to PingFang. Linux: `apt install fonts-noto-cjk` |
| `Error parsing filterchain` | Special chars in SRT path | Use relative path, avoid spaces |

## Quality Rules

- **Faithful transcription**: Only transcribe what was actually said — do not add or rewrite meaning
- **Proper noun correction**: Fix common Whisper misrecognitions, but confirm with the user
- **Subtitle rhythm**: Max two lines per entry, max ~20 CJK characters per line
- **Clean up**: Remind user to delete intermediate files (_audio.wav) after completion
