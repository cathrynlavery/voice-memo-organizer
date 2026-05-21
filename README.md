# Voice Memo Organizer

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that turns hundreds of untitled Apple Voice Memos into a searchable, organized archive — with descriptive titles, summaries, themes, and key quotes.

Transcription runs 100% locally on your Mac — no API keys needed. The skill supports two engines: **Parakeet** (recommended, faster + better punctuation, ~2.3 GB model) or **whisper.cpp** (lighter, ~150 MB model, better fit for Macs with less RAM). Summarization is handled by Claude.

## What It Does

Your Apple Voice Memos are buried in a hidden macOS folder, named things like "New Recording 247." This skill:

1. **Finds** all your Voice Memos (they're at a non-obvious path that requires Full Disk Access)
2. **Extracts** metadata from Apple's SQLite database (dates, durations, labels)
3. **Transcribes** every recording locally — choose [Parakeet MLX](https://github.com/senstella/parakeet-mlx) (recommended) or [whisper.cpp](https://github.com/ggerganov/whisper.cpp) (lighter); both run ~60× realtime on Apple Silicon in batch mode
4. **Summarizes** each transcript with a descriptive title, themes, key quotes, and type classification
5. **Builds** a searchable master index document
6. **(Optional)** Renames memos through the Voice Memos app on Mac so Apple's own iCloud sync pushes the new titles to your iPhone

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

Model files live in `~/.cache/huggingface/hub/` (Parakeet, ~2.3 GB) or `~/Documents/Voice-Memos-Organized/models/` (whisper, ~150 MB) — one-time downloads.

## Choosing a Transcription Engine

|                              | Parakeet MLX (recommended)  | whisper.cpp (lighter)                       |
| ---------------------------- | --------------------------- | ------------------------------------------- |
| Model size on disk           | ~2.3 GB                     | ~150 MB (base)                              |
| RAM during transcription     | ~3-4 GB                     | ~500 MB                                     |
| Speed (batch, Apple Silicon) | ~60× realtime               | ~60× realtime                               |
| LibriSpeech test-clean WER   | 1.69%                       | ~5%                                         |
| Punctuation/capitalization   | proper                      | basic                                       |
| Language coverage            | English + 24 European       | per-model (en or multilingual)              |
| Setup                        | `pipx install parakeet-mlx` | `brew install whisper-cpp` + model download |

Default to Parakeet. Choose whisper if your Mac has less than ~16 GB RAM, you want a much smaller disk footprint, or you need a language Parakeet v3 doesn't cover (Mandarin, Japanese, Arabic, etc.).

## Performance

Measured on an M3 Ultra in May 2026: **~60× realtime** in batch mode with Parakeet. A library of 329 memos (67 hours) takes roughly 70 minutes end-to-end, including the one-time model download. Whisper.cpp `base.en` hits similar throughput.

## Tips

- The master index works great in Obsidian, VS Code, or any text editor with Cmd+F
- Parakeet's v3 model handles English plus 24 European languages; whisper supports many more via its multilingual `ggml-base.bin`
- Works with memos organized into folders in the Voice Memos app
- Step 7 (optional) uses Voice Memos UI automation for renames. Direct SQLite title edits do not sync to iPhone.

## Credits

Built by [Cathryn Lavery](https://x.com/cathrynlavery) — founder, builder, and voice memo hoarder.

More tools and frameworks at [founder.codes](https://founder.codes).
