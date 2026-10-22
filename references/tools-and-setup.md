# Tools and setup

Every external tool this skill uses, with its official link. Follow Hard rule 2 from SKILL.md: ask the user's permission before downloading, installing, or signing up for any of these. Show the user what you want to install, why, and where the link goes.

## Required tools

### yt-dlp - video and audio download
- Repo: https://github.com/yt-dlp/yt-dlp
- Docs: https://github.com/yt-dlp/yt-dlp#readme
- Install: `pip install -U yt-dlp`, `brew install yt-dlp`, or the standalone binary from the repo releases.
- Used for: metadata checks (step 1), audio-only download (step 2), section download of approved segments (step 7).
- Note: keep it updated; video platforms change often. A JavaScript runtime (such as deno, https://deno.land) may be needed for some sites.

### ffmpeg with libass - rendering and subtitles
- Site: https://ffmpeg.org
- Static Linux builds: https://johnvansickle.com/ffmpeg/
- macOS: `brew install ffmpeg` (includes libass). Windows: https://www.gyan.dev/ffmpeg/builds/
- Check libass is present: `ffmpeg -version | grep libass` (look for `--enable-libass`). The `ass` subtitle filter needs it.
- Used for: audio extraction, layout compositing, ASS caption rendering, final encode (H.264 + AAC).

### Python 3.11 or newer - glue scripts
- Site: https://www.python.org/downloads/
- Used for: building the ASS caption file from word timings, transcript correction alignment, and small utilities. Standard library only; no packages required.

## Services (user's own account and API key)

The user creates their own account and key for each service. Keys go in environment variables or the local config file, never in the skill.

### AssemblyAI - transcription with word timestamps
- Site: https://www.assemblyai.com
- API docs: https://www.assemblyai.com/docs
- Used for: step 3 (transcribe the audio, per-word start/end times) and as the timing source for captions.
- Alternatives: any speech-to-text API that returns word-level timestamps (for example Deepgram, https://deepgram.com, or OpenAI Whisper run locally, https://github.com/openai/whisper). Whisper word timings are less precise; prefer a word-timed API for karaoke captions.
- Cost: check the provider's current pricing before transcribing long files, and tell the user the estimate.

### Google Gemini API - moment picking and transcript correction
- Site: https://ai.google.dev
- Used for: step 4 (picking the best moments from the transcript) and step 6 (correcting ASR mistakes, including listening to short audio segments for a second opinion on dialect words).
- Alternatives: any capable LLM for moment picking; any multimodal (audio-capable) LLM for the correction pass.

## Fonts

### Noto Kufi Arabic - Arabic caption font
- Site: https://fonts.google.com/noto/specimen/Noto+Kufi+Arabic
- License: SIL Open Font License (free for commercial use).
- Install system-wide, or keep the .ttf files in a folder and point libass at it with `fontsdir` (see `references/caption-rendering.md`).
- Why it matters: default fonts like DejaVu have no Arabic glyphs, and without a real Arabic font libass cannot shape or join letters. Any well-made Arabic font works; Noto Kufi Arabic is the tested default.

## Optional

### deno - JavaScript runtime for yt-dlp
- Site: https://deno.land
- Some video sites need a JS runtime for format extraction. Install only if yt-dlp reports it is needed.

### Google Drive or other cloud storage - optional delivery target
- Only if the user asks for it. The default delivery target is a local folder. If the user wants clips in Drive, use `rclone` (https://rclone.org) or the platform's own upload, after asking permission.
