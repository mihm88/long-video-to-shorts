# Transcription, moment picking, and correction

Details for steps 3, 4, 6, and 7 of SKILL.md.

## Step 3: Transcribe with word timestamps (AssemblyAI)

1. Upload the audio: `POST https://api.assemblyai.com/v2/upload` with header `authorization: $ASSEMBLYAI_API_KEY` and the file as the body. The response gives an `upload_url`.
2. Start the transcript: `POST https://api.assemblyai.com/v2/transcript` with JSON `{"audio_url": "<upload_url>", "language_code": "ar"}` (set the real language; omit `language_code` to auto-detect).
3. Poll `GET https://api.assemblyai.com/v2/transcript/<id>` until `status` is `completed` or `error`. Long files finish fast; a 2.5-hour file typically takes under a minute.
4. Save the whole JSON. The `words` array (`text`, `start`, `end` in milliseconds) is the timing source for captions.

Keep the raw transcript JSON until the project is done. Re-transcribing costs money and time.

## Step 4: Pick the best moments

Give the LLM the transcript with timestamps and ask for `num_clips` moments (default 5), each 30 to 60 seconds, that:

- express one complete thought,
- open with a hook in the first seconds,
- need no earlier context to understand.

Ask for strict JSON back: start, end, score (0-10), one-line reason, and a suggested short title. The user approves or edits this list at gate 5 before any video is downloaded.

## Step 6: Correct the transcript, keep the timing

ASR output has errors, especially on dialect speech, loanwords, and technical terms. Correct the text but keep the ASR word timings, because the captions are timed by them.

1. For each approved moment, run a correction pass. Best: a multimodal LLM listens to the audio segment and returns the corrected text. Also good: a text-only LLM pass with context (topic, speaker, slide text if any).
2. Keep the speaker's dialect and word choices. Fix only clear errors (for example a misheard word that breaks the meaning).
3. Mark every word you cannot verify. These go into the review list at step 11. Never silently guess and move on.

### Aligning corrected words back to ASR timings

The corrected text no longer matches the ASR words one to one. Align them:

1. Normalize both sides before comparing: lowercase; strip punctuation, Arabic diacritics (U+064B to U+065F) and tatweel (U+0640); unify Arabic letter variants (alef forms to plain alef, alef-maqsura to ya, ta-marbuta to ha); optionally drop a leading `ال`.
2. Align with `difflib.SequenceMatcher` on the normalized tokens.
3. Equal tokens keep their ASR timing. A replaced span distributes the span's total time across the replacement words. Inserted words borrow time from a neighboring gap or steal a small slice from the previous word; deleted ASR words drop out.
4. Enforce a minimum word duration (about 0.12 s) and a strictly monotonic timeline.
5. Merge a standalone Arabic article token (`الـ`, `للـ`) into the following word so captions never show it alone.

## Step 7: Download only the approved segments

```
yt-dlp -f 'bv*[height<=1080][ext=mp4]+ba/b[height<=1080]' \
  --download-sections '*START-END' --force-keyframes-at-cuts \
  --concurrent-fragments 4 -o 'segment.%(ext)s' URL
```

- Pad each range by 1 to 2 seconds on both sides; trim exactly when rendering.
- `--force-keyframes-at-cuts` re-encodes the cut points so the segment starts cleanly.
- Section downloads can be slow on long videos. If one stalls, retry with more concurrent fragments or download a low-resolution full copy and cut locally with ffmpeg.
- Verify the audio offset: the segment's audio must line up with the transcript timings for that range. Check by listening or by audio matching, and correct for any constant offset before building captions.
- Endpoint rule: nudge start and end so the clip begins and ends on a complete sentence. A clip that cuts mid-thought feels broken even when the timing list approved it.
