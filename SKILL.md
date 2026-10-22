---
name: long-video-to-shorts
description: Turn long videos (talks, webinars, podcasts, interviews) into short vertical clips with synced word-by-word captions, including correct Arabic and RTL rendering. Use when the user wants to cut a long video into shorts, reels, or TikToks. Runs 12 steps with approval gates - rights check, transcription with word timestamps, AI moment picking, one proof clip, caption review, delivery. Renders with ffmpeg and ASS subtitles (libass) and includes fixes for mixed Arabic/English line order. Needs yt-dlp, ffmpeg, and the user's own API keys; asks permission before any download or install and never auto-publishes.
---

# Long Video to Shorts

Turn one long video into several short vertical clips (9:16) with word-by-word captions that highlight as they are spoken. Works for Arabic and other RTL languages as well as English, including lines that mix both.

This skill runs in Claude Code, Claude Cowork, Codex, and similar agent environments. It is a procedure, not a program: the agent follows the 12 steps below and uses the tools linked in `references/tools-and-setup.md`.

## Hard rules

These rules protect the user. Never skip them.

1. **Copyright warning before any download.** Before downloading any video or audio, show the user this warning and wait for a clear yes:

   > Downloading a video you do not own, or do not have permission to reuse, can infringe copyright and break the platform's terms of service. Please confirm that you own this content or have permission to clip it.

   Proceed only after the user confirms. If they are unsure, stop.

2. **Ask before installing anything.** Every tool this skill uses is linked in `references/tools-and-setup.md`. Never download, install, or sign up for a tool without asking the user first. Tell them what you want to install, why, and where the link goes.

3. **The user supplies their own API keys.** This skill ships with no keys. Keys come from environment variables or a local config file the user creates. Never paste a key into the skill files, and never commit one anywhere.

4. **No auto-publishing.** Deliver finished clips to the user's chosen folder and stop. Never upload, post, or share clips without an explicit instruction from the user for that specific action.

5. **Logo and colors are user settings.** This skill ships with no logo and no brand assets. Ask the user for their logo file and color choices, or use the defaults in `config.example.json`.

## First run: ask where to save

On the first run, before step 1, ask the user:

- Where should working files and finished clips be saved? (A local folder. Google Drive or any cloud storage is optional, never required.)
- Do you have a logo image to place on the clips? (Optional. PNG with a transparent background works best.)
- Keep the default style (dark background, white captions, amber highlight) or pick your own colors?

Save the answers in `long-video-to-shorts.config.json` in the working folder (start from `config.example.json`). On later runs, reuse the saved answers; ask again only if the folder no longer exists or the user wants a change.

## The 12 steps

Steps 5, 9, and 12 are approval gates: stop and wait for the user.

### 1. Check rights and metadata (no download yet)
Show the copyright warning (Hard rule 1) and get confirmation. Then fetch metadata only: title, duration, resolution, language, channel or source. For YouTube use `yt-dlp --skip-download --print ...`. Nothing is downloaded in this step.

### 2. Download audio only
Download the audio track at low quality (it is only for transcription, not for the final clips). For YouTube: `yt-dlp -f 'bestaudio[abr<=64]/worst' -x --audio-format mp3`. For a local file: extract audio with ffmpeg. Tell the user the file size before proceeding if the video is very long.

### 3. Transcribe with word timestamps
Send the audio to a transcription service that returns per-word start and end times (AssemblyAI works well; see `references/transcription-and-clipping.md`). Set the language explicitly when known. Keep the raw JSON; the word timings drive the captions later.

### 4. Pick the best moments
Give an LLM the timestamped transcript and ask for the requested number of moments (default 5) that each stand alone in 30 to 60 seconds. For each moment return: start time, end time, a score, and a one-line reason. Selection criteria: a complete thought, a strong hook in the first seconds, no dependence on earlier context.

### 5. GATE: user approves the moment list
Send the user the list with times, scores, and reasons. Wait for approval or adjustments. Do not download video segments before this gate.

### 6. Correct the transcript
ASR makes mistakes, especially on dialect and technical terms. Run a correction pass (a multimodal LLM listening to each chosen segment, or a text-only pass with context). Keep the speaker's dialect and wording; fix only clear errors. Mark every word you cannot verify and list them for the user at step 11. Align the corrected words back onto the ASR timings (alignment recipe in `references/transcription-and-clipping.md`).

### 7. Download only the needed segments
Download just the approved time ranges at full quality: `yt-dlp --download-sections '*START-END' --force-keyframes-at-cuts`. Pad each range by a second or two for caption lead-in and audio tail. Verify the offset between the segment audio and the transcript timings by listening or audio matching.

### 8. Build one proof clip
Render a single clip end to end: layout, logo, captions, colors. Use the ffmpeg + ASS recipe in `references/caption-rendering.md`. For Arabic or mixed lines, apply the RTL fixes in that file before showing anything to the user.

### 9. GATE: user approves the proof
Send the proof clip. Ask specifically about: layout, logo size and position, caption font and size, highlight color, line order on mixed-language lines. Wait for approval before rendering the rest. Apply any fixes to the shared settings, not one clip at a time.

### 10. Build the remaining clips
Render the rest with the approved settings. Adjust each clip's start and end so it begins and ends on a complete thought, even if that shifts the approved times slightly. If the source has a burned-in logo or watermark that clashes with the user's logo, crop it out or cover it cleanly.

### 11. Review the captions
Check every clip's captions against the audio one more time. List for the user: every corrected word and every word still marked unverifiable, with its clip and time. The user reads the list and sends corrections; re-render only the affected captions.

### 12. GATE: deliver, do not publish
Copy the finished clips to the user's save folder with clear file names (clip number + short title). Report what was delivered and where. Ask before any upload or share; "delivered" never means "published".

## Style defaults

- 1080 x 1920 (9:16), 30 fps, H.264 + AAC, one clip per moment, 30 to 60 seconds.
- Dark background (#1A1A1A), the source frame scaled to full width and centered vertically.
- Logo (if the user provides one): far top-left, about 20% of frame width, white monochrome on a transparent background so the video stays visible behind it.
- Captions: 2 to 4 words per line, white, the currently spoken word highlighted (default amber #FFC840), positioned in the lower third but clear of platform UI.
- Arabic captions: Noto Kufi Arabic. Never rely on a default sans font for Arabic; see the font tip in `references/caption-rendering.md`.

## References

- `references/tools-and-setup.md` - every tool and service, with links, install notes, and the permission-asking rule.
- `references/transcription-and-clipping.md` - transcription with word timings, moment picking, transcript correction, and word alignment.
- `references/caption-rendering.md` - the ffmpeg + ASS render recipe, karaoke captions, and the Arabic/RTL fixes (including the RLM/RLE line-order fix and the Arabic font tip).
- `config.example.json` - every user setting in one file.
