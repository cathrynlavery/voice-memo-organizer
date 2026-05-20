# Voice Memo Organizer

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that turns hundreds of untitled Apple Voice Memos into a searchable, organized archive — with descriptive titles, summaries, themes, and key quotes.

Transcription runs 100% locally on your Mac using NVIDIA Parakeet (MLX port for Apple Silicon) — no API keys needed. Summarization is handled by Claude.

## What It Does

Your Apple Voice Memos are buried in a hidden macOS folder, named things like "New Recording 247." This skill:

1. **Finds** all your Voice Memos (they're at a non-obvious path that requires Full Disk Access)
2. **Extracts** metadata from Apple's SQLite database (dates, durations, labels)
3. **Transcribes** every recording locally using [Parakeet MLX](https://github.com/senstella/parakeet-mlx) (~60× realtime on Apple Silicon in batch mode, with proper punctuation)
4. **Summarizes** each transcript with a descriptive title, themes, key quotes, and type classification
5. **Builds** a searchable master index document
6. **(Optional)** Writes the new titles back to iCloud so they sync to your iPhone

## Before → After

**Before:** 329 recordings named "New Recording 1" through "New Recording 329"

**After:** A searchable markdown index with entries like:

- "90-Day Focus Sprint Framework" (productivity, goal-setting)
- "Bedtime Story About the Brave Little Fox" (personal, family)
- "Legal Call Re: Buying Back Company from PE" (business, legal)

Each with summaries, themes, and key quotes extracted. With Step 7, the same titles appear in the Voice Memos app on your iPhone.

## Install

### Prerequisites

- macOS on Apple Silicon (M1/M2/M3/M4) with Apple Voice Memos
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [Homebrew](https://brew.sh)

### Setup

1. **Enable Full Disk Access for your terminal:**
   System Settings → Privacy & Security → Full Disk Access → enable your terminal app (Terminal, iTerm2, etc.)

2. **Install the skill:**

   ```bash
   # Create the skills directory
   mkdir -p ~/.claude/skills/voice-memo-organizer

   # Download the skill and troubleshooting reference
   curl -fL -o ~/.claude/skills/voice-memo-organizer/SKILL.md \
     https://raw.githubusercontent.com/cathrynlavery/voice-memo-organizer/main/SKILL.md
   curl -fL -o ~/.claude/skills/voice-memo-organizer/TROUBLESHOOTING.md \
     https://raw.githubusercontent.com/cathrynlavery/voice-memo-organizer/main/TROUBLESHOOTING.md
   ```

3. **Tell Claude Code to use it:**

   ```
   claude "organize my voice memos"
   ```

   Claude will walk through each step: finding your memos, installing parakeet-mlx and ffmpeg via Homebrew, transcribing everything, summarizing, and building the index.

## What You'll Get

```
~/Documents/Voice-Memos-Organized/
├── voice-memos-master-index.md    ← searchable master document
├── transcripts/                   ← full text of every recording
├── summaries/                     ← JSON summaries with titles/themes/quotes
└── recordings-metadata.csv        ← database export with dates & durations

~/Documents/Voice-Memos-Raw/       ← copies of your original audio files
```

The Parakeet model itself lives in `~/.cache/huggingface/hub/` (~2.3 GB, one-time download).

## Performance

Measured on an M3 Ultra in May 2026: **~60× realtime** in batch mode. A library of 329 memos (67 hours) takes roughly 70 minutes end-to-end, including the one-time model download.

## Tips

- The master index works great in Obsidian, VS Code, or any text editor with Cmd+F
- Parakeet's v3 model handles English plus 24 European languages
- For non-European languages (Mandarin, Japanese, Arabic, etc.), use the whisper.cpp fallback documented in TROUBLESHOOTING.md
- Works with memos organized into folders in the Voice Memos app
- Step 7 (optional) renames memos in iCloud so the new titles appear on your iPhone

## Credits

Built by [Cathryn Lavery](https://x.com/cathrynlavery) — founder, builder, and voice memo hoarder.

More tools and frameworks at [founder.codes](https://founder.codes).
