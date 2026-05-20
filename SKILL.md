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

The skill supports two transcription engines. **Parakeet** is recommended for most users; **whisper.cpp** is the lighter alternative for Macs with less RAM/disk.

|                              | Parakeet MLX (recommended)  | whisper.cpp (lighter)                       |
| ---------------------------- | --------------------------- | ------------------------------------------- |
| Model size on disk           | ~2.3 GB                     | ~150 MB (base)                              |
| RAM during transcription     | ~3-4 GB                     | ~500 MB                                     |
| Speed (batch, Apple Silicon) | ~60× realtime               | ~60× realtime                               |
| LibriSpeech test-clean WER   | 1.69%                       | ~5%                                         |
| Punctuation/capitalization   | proper                      | basic                                       |
| Language coverage            | English + 24 European       | per-model (en or multilingual)              |
| Setup                        | `pipx install parakeet-mlx` | `brew install whisper-cpp` + model download |

Ask the user which they want. If they don't say, default to Parakeet unless their Mac has <16 GB RAM, in which case suggest whisper.

**Parakeet (recommended):**

```bash
brew install python@3.14 ffmpeg pipx
pipx install parakeet-mlx
```

No manual model download — first run fetches the ~2.3 GB MLX model from Hugging Face into `~/.cache/huggingface/hub/`.

**Whisper.cpp (lighter alternative):**

```bash
brew install ffmpeg whisper-cpp
mkdir -p ~/Documents/Voice-Memos-Organized/models
curl -L -o ~/Documents/Voice-Memos-Organized/models/ggml-base.en.bin \
  "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin"
```

For non-English memos, swap `ggml-base.en.bin` → `ggml-base.bin` (multilingual). The Homebrew formula installs the binary as `whisper-cli` (not `whisper-cpp`).

### Step 4: Batch Transcribe

Run the batch script for whichever engine the user installed in Step 3.

**Parakeet batch (recommended):** `parakeet-mlx` accepts `.m4a` and `.qta` directly (handles decoding via ffmpeg internally) and loads the model once per invocation, so batch processing is fastest.

```bash
mkdir -p ~/Documents/Voice-Memos-Organized/transcripts

# Collect files that haven't been transcribed yet
TODO=()
for audiofile in ~/Documents/Voice-Memos-Raw/*; do
    [ -f "$audiofile" ] || continue
    BASENAME=$(basename "$audiofile" | sed 's/\.[^.]*$//')
    [ -s ~/Documents/Voice-Memos-Organized/transcripts/"${BASENAME}.txt" ] && continue
    TODO+=("$audiofile")
done

echo "Transcribing ${#TODO[@]} memos..."

# Process in chunks of 50 to keep memory bounded
for ((i=0; i<${#TODO[@]}; i+=50)); do
    chunk=("${TODO[@]:i:50}")
    parakeet-mlx --output-format txt \
        --output-dir ~/Documents/Voice-Memos-Organized/transcripts \
        "${chunk[@]}" || echo "Chunk starting at $i had errors, continuing"
done
```

**Performance** (verified 2026-05 on M3 Ultra): 8 min of audio in ~8 sec in batch mode = ~60× realtime. 67 hours of memos ≈ 70 min end-to-end including the one-time 2.3 GB model download.

**Whisper batch (lighter alternative):** loads the model per file but uses ~500 MB RAM instead of 3-4 GB. Each file is converted to 16 kHz mono WAV first.

```bash
WHISPER="$(command -v whisper-cli)"
MODEL="$HOME/Documents/Voice-Memos-Organized/models/ggml-base.en.bin"
mkdir -p ~/Documents/Voice-Memos-Organized/transcripts

for audiofile in ~/Documents/Voice-Memos-Raw/*; do
    [ -f "$audiofile" ] || continue
    BASENAME=$(basename "$audiofile" | sed 's/\.[^.]*$//')
    TEMP_WAV="/tmp/vm_${BASENAME}.wav"

    [ -s ~/Documents/Voice-Memos-Organized/transcripts/"${BASENAME}.txt" ] && continue

    if ! ffmpeg -y -i "$audiofile" -ar 16000 -ac 1 -c:a pcm_s16le "$TEMP_WAV" 2>/dev/null; then
        echo "SKIP: Could not convert $BASENAME"
        continue
    fi

    "$WHISPER" -m "$MODEL" -f "$TEMP_WAV" --output-txt \
        -of ~/Documents/Voice-Memos-Organized/transcripts/"$BASENAME" \
        -t 8 --no-timestamps 2>/dev/null

    rm -f "$TEMP_WAV"
done
```

**Performance** with whisper-cpp `base.en` on Apple Silicon: ~60× realtime. 67 hours of memos ≈ 1 hour to transcribe.

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

### Step 7: Rename in iCloud (optional — syncs titles to iPhone)

The title your iPhone shows comes from `ZCUSTOMLABEL` in `CloudRecordings.db`, not the filename. Updating that column writes the new titles back through iCloud to every device.

> **Warning:** this edits a live, iCloud-synced SQLite database. Default to dry-run. Test on ONE memo before bulk-applying. Newer macOS versions store an encrypted title in `ZENCRYPTEDTITLE` that overrides `ZCUSTOMLABEL` — this script clears it so the plain label wins.
>
> **Verified 2026-05** on macOS 26: single-memo rename synced to iPhone within ~30s of reopening Voice Memos on Mac.

**Pre-flight (required):**

1. Quit Voice Memos on **every** device — Mac (⌘Q), iPhone, iPad (swipe up from app switcher).
2. Wait ~30 seconds for iCloud to settle.
3. Back up the database:

```bash
DB=~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db
cp "$DB" ~/Documents/Voice-Memos-Organized/CloudRecordings.db.backup-$(date +%Y%m%d-%H%M%S)
```

**Dry run** (prints proposed renames, no DB writes):

```bash
DB=~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db

for summary in ~/Documents/Voice-Memos-Organized/summaries/*.json; do
    BASENAME=$(basename "$summary" .json)
    NEW_TITLE=$(jq -r '.title // empty' "$summary")
    [ -z "$NEW_TITLE" ] && continue

    CURRENT=$(sqlite3 "$DB" "SELECT COALESCE(ZCUSTOMLABEL,'(untitled)') FROM ZCLOUDRECORDING WHERE ZPATH LIKE '%${BASENAME}%' LIMIT 1;")
    [ -z "$CURRENT" ] && { echo "SKIP: no DB row for $BASENAME"; continue; }
    echo "[DRY] $BASENAME: '$CURRENT' → '$NEW_TITLE'"
done
```

Review the output. If it looks right, run the apply step.

**Apply** (writes to DB — wrapped in a transaction):

```bash
DB=~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db

sqlite3 "$DB" "BEGIN;"
for summary in ~/Documents/Voice-Memos-Organized/summaries/*.json; do
    BASENAME=$(basename "$summary" .json)
    NEW_TITLE=$(jq -r '.title // empty' "$summary")
    [ -z "$NEW_TITLE" ] && continue

    # Escape single quotes for SQL
    SAFE_TITLE=$(printf "%s" "$NEW_TITLE" | sed "s/'/''/g")
    sqlite3 "$DB" "UPDATE ZCLOUDRECORDING
                   SET ZCUSTOMLABEL = '$SAFE_TITLE',
                       ZENCRYPTEDTITLE = NULL
                   WHERE ZPATH LIKE '%${BASENAME}%';"
    echo "RENAMED: $BASENAME → $NEW_TITLE"
done
sqlite3 "$DB" "COMMIT;"
```

Then reopen Voice Memos on Mac, wait for the sync indicator to clear (~30s), then open it on iPhone. New titles should appear.

**Recovery** — if anything looks wrong, quit Voice Memos everywhere and restore:

```bash
DB=~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db
ls -t ~/Documents/Voice-Memos-Organized/CloudRecordings.db.backup-* | head -1 | xargs -I {} cp {} "$DB"
```

**Test one memo first.** Replace the loop with a single `UPDATE … WHERE Z_PK = <id>;` against one recording, reopen the app, and confirm the new title shows up on iPhone before running the full batch.

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
