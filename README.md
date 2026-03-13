# Voice Memo Organizer

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that turns hundreds of untitled Apple Voice Memos into a searchable, organized archive — with descriptive titles, summaries, themes, and key quotes.

Everything runs locally on your Mac. No API keys needed for transcription.

## What It Does

Your Apple Voice Memos are buried in a hidden macOS folder, named things like "New Recording 247." This skill:

1. **Finds** all your Voice Memos (they're at a non-obvious path that requires Full Disk Access)
2. **Extracts** metadata from Apple's SQLite database (dates, durations, folder info)
3. **Transcribes** every recording locally using [whisper.cpp](https://github.com/ggerganov/whisper.cpp) (~1 min of audio per second on Apple Silicon)
4. **Summarizes** each transcript with a descriptive title, themes, key quotes, and type classification
5. **Builds** a searchable master index document

## Before → After

**Before:** 329 recordings named "New Recording 1" through "New Recording 329"

**After:** A searchable markdown index with entries like:
- "90-Day Focus Sprint Framework" (productivity, goal-setting)
- "Bedtime Story About the Brave Little Fox" (personal, family)
- "Legal Call Re: Buying Back Company from PE" (business, legal)

Each with summaries, themes, and key quotes extracted.

## Install

### Prerequisites

- macOS with Apple Voice Memos (recordings sync via iCloud from iPhone)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [Homebrew](https://brew.sh)

### Setup

1. **Enable Full Disk Access for your terminal:**
   System Settings → Privacy & Security → Full Disk Access → enable your terminal app (Terminal, iTerm2, etc.)

2. **Install the skill:**
   ```bash
   # Create the skills directory
   mkdir -p ~/.claude/skills/voice-memo-organizer

   # Download the skill
   curl -o ~/.claude/skills/voice-memo-organizer/SKILL.md \
     https://raw.githubusercontent.com/cathrynlavery/voice-memo-organizer/main/SKILL.md
   ```

3. **Tell Claude Code to use it:**
   ```
   claude "organize my voice memos"
   ```

   Claude will walk through each step: finding your memos, installing whisper.cpp and ffmpeg via Homebrew, transcribing everything, summarizing, and building the index.

## What You'll Get

```
~/Documents/Voice-Memos-Organized/
├── voice-memos-master-index.md    ← searchable master document
├── transcripts/                   ← full text of every recording
├── summaries/                     ← JSON summaries with titles/themes/quotes
├── recordings-metadata.csv        ← database export with dates & durations
└── models/                        ← whisper model (downloaded automatically)

~/Documents/Voice-Memos-Raw/       ← copies of your original audio files
```

## Performance

On Apple Silicon, whisper.cpp transcribes ~1 minute of audio per second. A library of 329 memos (67 hours) takes roughly 1 hour to fully transcribe.

## Tips

- The master index works great in Obsidian, VS Code, or any text editor with Cmd+F
- For non-English memos, the skill can use the multilingual whisper model instead
- For higher accuracy on important recordings, request the medium model (~500MB)
- Works with memos organized into folders in the Voice Memos app

## Credits

Built by [Cathryn Lavery](https://x.com/cathrynlavery) — founder, builder, and voice memo hoarder.

More tools and frameworks at [founder.codes](https://founder.codes).
