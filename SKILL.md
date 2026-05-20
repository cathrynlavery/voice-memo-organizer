---
name: voice-memo-organizer
description: "Find, transcribe, summarize, and organize all Apple Voice Memos into a searchable archive. Batch processes hundreds of recordings locally using whisper.cpp — no API keys needed. Use when the user wants to organize, transcribe, or search their voice memos, mentions untitled recordings, or asks about finding Apple Voice Memos on macOS."
---

# Voice Memo Organizer

Turn hundreds of untitled Apple Voice Memos into a searchable, organized archive with descriptive titles, summaries, themes, and key quotes.

## Prerequisites

- macOS with Voice Memos app (recordings synced via iCloud from iPhone)
- **Full Disk Access** for Terminal: System Settings > Privacy & Security > Full Disk Access > enable the terminal app
- Homebrew installed

## Step-by-Step Process

### Step 1: Access Voice Memos

Voice Memos are stored at:
```
~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/
```

This path requires Full Disk Access. Verify access:
```bash
ls ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/ | head -5
```

If permission denied → user must enable Full Disk Access for their terminal app. If the directory is empty → user should open Voice Memos on Mac and wait for iCloud sync.

The folder also contains `CloudRecordings.db` — a SQLite database with metadata:
```bash
sqlite3 ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db \
  "SELECT COUNT(*), ROUND(SUM(ZDURATION)/3600,1) as hours FROM ZCLOUDRECORDING WHERE ZPATH IS NOT NULL;"
```

### Step 2: Copy Recordings & Export Metadata

```bash
mkdir -p ~/Documents/Voice-Memos-Organized/{transcripts,summaries,models}
mkdir -p ~/Documents/Voice-Memos-Raw

# Copy all recordings
cp ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/*.m4a ~/Documents/Voice-Memos-Raw/ 2>/dev/null
cp ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/*.qta ~/Documents/Voice-Memos-Raw/ 2>/dev/null

# Export metadata
sqlite3 -header -csv ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db \
  "SELECT Z_PK as id, ZCUSTOMLABEL as label, ZPATH as filename, ROUND(ZDURATION,1) as duration_secs, datetime(ZDATE + 978307200, 'unixepoch', 'localtime') as recorded_date FROM ZCLOUDRECORDING WHERE ZPATH IS NOT NULL AND ZPATH != '' ORDER BY ZDATE ASC;" \
  > ~/Documents/Voice-Memos-Organized/recordings-metadata.csv
```

### Step 3: Install Transcription Tools

```bash
brew install ffmpeg whisper-cpp

# Download whisper.cpp base.en model (~150MB, good speed/accuracy for English)
curl -L -o ~/Documents/Voice-Memos-Organized/models/ggml-base.en.bin \
  "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin"
```

> **Note:** The Homebrew formula installs the binary as `whisper-cli` (not `whisper-cpp`).

### Step 4: Batch Transcribe

For each audio file:
1. Convert to 16kHz mono WAV using ffmpeg
2. Run whisper.cpp with the base.en model
3. Save transcript as .txt

```bash
WHISPER="$(command -v whisper-cli)"  # auto-detect path
MODEL="$HOME/Documents/Voice-Memos-Organized/models/ggml-base.en.bin"

for audiofile in ~/Documents/Voice-Memos-Raw/*; do
    [ -f "$audiofile" ] || continue
    BASENAME=$(basename "$audiofile" | sed 's/\.[^.]*$//')
    TEMP_WAV="/tmp/vm_${BASENAME}.wav"

    # Skip if already transcribed
    [ -s ~/Documents/Voice-Memos-Organized/transcripts/"${BASENAME}.txt" ] && continue

    # Convert to WAV (skip on failure to avoid stale data)
    if ! ffmpeg -y -i "$audiofile" -ar 16000 -ac 1 -c:a pcm_s16le "$TEMP_WAV" 2>/dev/null; then
        echo "SKIP: Could not convert $BASENAME"
        continue
    fi

    # Transcribe
    "$WHISPER" -m "$MODEL" -f "$TEMP_WAV" --output-txt \
      -of ~/Documents/Voice-Memos-Organized/transcripts/"$BASENAME" -t 8 --no-timestamps 2>/dev/null

    rm -f "$TEMP_WAV"
done
```

**Performance:** ~1 min of audio per second on Apple Silicon. 67 hours of memos ≈ 1 hour to transcribe.

### Step 5: Summarize Each Transcript

For each transcript, generate a JSON summary with:
- **title**: Short descriptive title (5-10 words)
- **summary**: 2-3 sentence summary of the content
- **themes**: Array of tags (e.g. "business", "personal", "health", "brainstorm")
- **key_quotes**: Array of 1-3 notable verbatim quotes
- **type**: One of "conversation", "brainstorm", "voice_note", "meeting", "personal", "phone_call"

Save each as a JSON file in the summaries/ folder.

Process in batches of 10-20 transcripts at a time using parallel agents for speed. Skip transcripts that are too short (<10 words) or contain only silence/noise.

### Step 6: Build Master Index

Create `voice-memos-master-index.md` with:
- Stats header (count, date range, total hours)
- Theme breakdown
- Type breakdown
- Chronological entries with: date, title, duration, type, themes, summary, key quotes, filename
- "Best Quotes" section at the end

## Output Structure

```
~/Documents/Voice-Memos-Organized/
├── voice-memos-master-index.md    ← searchable master document
├── transcripts/                   ← full text of every recording
├── summaries/                     ← JSON summaries with titles/themes/quotes
├── recordings-metadata.csv        ← database export
└── models/                        ← whisper model

~/Documents/Voice-Memos-Raw/       ← original audio files
```

## Tips

- For non-English memos, use `ggml-base.bin` (multilingual) instead of `ggml-base.en.bin`
- For higher accuracy on important recordings, use `ggml-medium.en.bin` (~500MB)
- The SQLite database also contains folder info if the user organized memos into folders in the app
- See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for Full Disk Access setup, iCloud sync, model selection, and common issues
