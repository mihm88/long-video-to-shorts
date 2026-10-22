# Caption rendering: ffmpeg + ASS (libass)

Details for steps 8 and 10 of SKILL.md.

## Why ffmpeg + ASS

ASS subtitles rendered by libass (ffmpeg's `ass` filter) handle complex scripts, including Arabic shaping and bidirectional text, with no browser or JavaScript stack. It renders fast on small machines and the whole layout is one ffmpeg command.

## The Arabic font tip

Do not rely on default fonts for Arabic. DejaVu and most default sans fonts have no Arabic glyphs, and without a real Arabic font libass cannot shape or join letters. Install Noto Kufi Arabic (link in `tools-and-setup.md`) or another well-made Arabic font, and always pass both the font directory and complex shaping to the filter:

```
ass=captions.ass:fontsdir=/path/to/fonts:shaping=complex
```

`shaping=complex` is required for correct Arabic rendering. Check the font is found: `fc-list | grep -i kufi`.

## Karaoke captions (word-by-word highlight)

Build the .ass file with one Dialogue event per word. Each event displays the whole line, with only the active word wrapped in a color override:

```
Dialogue: 0,0:00:01.20,0:00:01.65,Cap,,0,0,0,,﻿first {\c&H0040C8FF&}spoken{\c&H00FFFFFF&} words here
```

- Event start = the word's start; event end = the next word's start (cap the gap at about 1.2 s so a pause does not freeze the highlight).
- ASS colors are `&HAABBGGRR&` (blue-green-red order). Amber #FFC840 is `&H0040C8FF&`; white is `&H00FFFFFF&`.
- Group words into lines of 2 to 4 words; break a line at punctuation or a pause longer than about 0.6 s.
- Keep the words in logical (spoken) order inside the event text. libass handles display order from there.

### Style block (tested defaults for 1080x1920)

```
[Script Info]
ScriptType: v4.00+
PlayResX: 1080
PlayResY: 1920
WrapStyle: 2

[V4+ Styles]
Format: Name,Fontname,Fontsize,PrimaryColour,SecondaryColour,OutlineColour,BackColour,Bold,Italic,Underline,StrikeOut,ScaleX,ScaleY,Spacing,Angle,BorderStyle,Outline,Shadow,Alignment,MarginL,MarginR,MarginV,Encoding
Style: Cap,Noto Kufi Arabic,74,&H00FFFFFF,&H00FFFFFF,&H00000000,&H80000000,-1,0,0,0,100,100,0,0,1,6,2,2,60,60,330,1
```

Alignment 2 is bottom-center; MarginV 330 lifts captions into the lower third. Adjust MarginV and font size to taste, then keep them identical across all clips.

## Mixed Arabic/English lines: the line-order fix

libass picks a line's base direction from its first strong character. When a mixed line starts with a Latin word ("AI" or an English term), the base direction flips to LTR and the Arabic words display in reverse order.

Fixes that worked in practice:

1. **Prepend RLE (U+202B) to every mixed Arabic/English line** to force an RTL base direction. RLM (U+200F) is the weaker hint: some libass builds strip or ignore it. RLE plus `shaping=complex` rendered correctly. Keep the override tag after the mark.
2. **Merge or drop the Arabic article before a Latin word.** A standalone `الـ` in front of "AI" can break ordering. Merge it into the next token for display, or drop it.
3. **Wrap long Latin-only lines by hand** (about 22 characters) so nothing is clipped at the frame edge.
4. **Always verify on real frames.** Extract frames at several timestamps and look at them: letter joining, line order, highlight on the right word, nothing cut off. Do not trust the file or the exit code. Render a few seconds first, check, then render the full clip.

Frame extraction for checking:

```
ffmpeg -v error -y -ss 38 -i clip.mp4 -frames:v 1 frame38.png
```

## Full render command (9:16 layout)

One ffmpeg command composites background, scaled source frame, logo, and captions:

```
ffmpeg -v error -y -ss START -t DURATION -i segment.mp4 -loop 1 -i logo.png -filter_complex "\
color=c=0x1a1a1a:s=1080x1920:r=30[bg];\
[0:v]scale=1080:-2[sl];\
[bg][sl]overlay=0:(H-h)/2:shortest=1[a];\
[1:v]scale=220:220[lg];\
[a][lg]overlay=30:40[b];\
[b]ass=captions.ass:fontsdir=/path/to/fonts:shaping=complex[v]" \
-map "[v]" -map 0:a -c:v libx264 -preset veryfast -crf 24 -r 30 \
-pix_fmt yuv420p -c:a aac -b:a 128k -shortest clip.mp4
```

- `overlay=0:(H-h)/2` centers the source vertically; use a fixed y (for example 300) if you want the frame higher.
- Skip `-loop 1 -i logo.png` and the logo overlay lines when the user has no logo.
- `-pix_fmt yuv420p` keeps the file playable everywhere, including WhatsApp and social apps.
- Keep the result small: CRF 24 at 1080x1920 lands around 1 to 3 MB for a 60 s clip, inside chat upload limits.

## Source frames with burned-in logos or margins

- If the source is a slide with a black margin, crop to the content (`crop=W:H:X:Y`) instead of scaling the whole frame.
- If a burned-in logo clashes with the user's logo, remove it: crop it out when it sits in a margin, or cover it with ffmpeg's `delogo` filter (`delogo=x=..:y=..:w=..:h=..`) on flat or dark areas. Check the patched frames visually.
