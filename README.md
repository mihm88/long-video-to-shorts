# long-video-to-shorts

An agent skill that turns long videos (talks, webinars, podcasts, interviews) into short vertical clips (9:16) with synced word-by-word captions. Arabic and RTL captions render correctly, including lines that mix Arabic and English.

Works in Claude Code, Claude Cowork, Codex, and any agent environment that reads Agent Skills (`SKILL.md`).

## What it does

Runs a 12-step pipeline with approval gates:

1. Rights and metadata check (copyright warning first, nothing downloaded)
2. Audio-only download
3. Transcription with per-word timestamps
4. AI picks the best moments (times, score, reason)
5. **Gate: you approve the moment list**
6. Transcript correction (dialect-safe, uncertain words flagged)
7. Downloads only the approved segments at full quality
8. Builds one proof clip
9. **Gate: you approve the proof**
10. Builds the remaining clips
11. Caption review, with a list of every corrected and unverifiable word
12. **Gate: delivery to your chosen folder. Never auto-publishes.**

## Install

Copy the `long-video-to-shorts` folder into your agent's skills directory, for example:

- Claude Code: `~/.claude/skills/long-video-to-shorts/`
- Project level: `.claude/skills/long-video-to-shorts/`

## What it needs

The skill asks permission before installing anything and uses your own API keys (none are included):

- yt-dlp, ffmpeg (with libass), Python 3.11+
- A word-timed transcription API (AssemblyAI or similar)
- An LLM API (Gemini or similar) for moment picking and transcript correction
- Noto Kufi Arabic (free, OFL) for Arabic captions

Links and install notes: `references/tools-and-setup.md`.

## Your settings

On the first run the skill asks where to save clips (a local folder; cloud storage optional) and your logo and color choices, then remembers them. See `config.example.json`.

## License

MIT. See LICENSE.
