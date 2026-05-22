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
  "SELECT Z_PK as id, COALESCE(ZCUSTOMLABELFORSORTING, ZCUSTOMLABEL, '') as title, ZPATH as filename, ROUND(ZDURATION,1) as duration_secs, datetime(ZDATE + 978307200, 'unixepoch', 'localtime') as recorded_date FROM ZCLOUDRECORDING WHERE ZPATH IS NOT NULL AND ZPATH != '' ORDER BY ZDATE ASC;" \
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
brew install ffmpeg pipx
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

> **macOS 26 (Tahoe): direct SQLite writes to `CloudRecordings.db` cannot sync renamed titles to iPhone.** This is a hard limitation of how Apple's NSPersistentCloudKit framework tracks changes, not something you can work around with a different title column or flag. The Mac UI will show a SQLite-edited title locally, but iPhone will keep showing the old title indefinitely. **Use UI automation through the Voice Memos app.**

#### Findings from investigation (macOS 26.5, May 2026)

The original version of this skill said to `UPDATE ZCUSTOMLABEL` and `SET ZENCRYPTEDTITLE = NULL`. That guidance is wrong on multiple counts. Here is the actual schema behavior:

| Column                   | What it actually is                                                                                                                                                     | What to do                                                                                                                                                                      |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ZCUSTOMLABEL`           | ISO timestamp like `2026-05-16T05:02:02Z` for untitled memos. **Not** the display title.                                                                                | Leave alone.                                                                                                                                                                    |
| `ZCUSTOMLABELFORSORTING` | The display title shown in both Mac and iPhone apps. For unnamed memos: `"New Recording 309"`.                                                                          | Must be updated.                                                                                                                                                                |
| `ZENCRYPTEDTITLE`        | **Misnamed — it's plain UTF-8 text**, not encrypted. Confirmed by reading hex: `544553542052454E414D4520E28094204D6179203230` = `"TEST RENAME — May 20"` byte-for-byte. | Must be updated to the same plain text as `ZCUSTOMLABELFORSORTING`. **Never NULL it** — that crashes Voice Memos on next launch with an `NSFetchedResultsController` `SIGABRT`. |

Even with both title columns set correctly via SQLite:

- **Mac UI**: shows the new title after relaunching Voice Memos ✓
- **iPhone**: does **not** receive the rename — iCloud never pushes it

#### Why iPhone sync fails

`CloudRecordings.db` is a Core Data store with NSPersistentCloudKit. CloudKit sync is driven by these tables:

- `ATRANSACTION` / `ACHANGE` — Core Data persistent history. Records which entities changed and when.
- `ANSCKEXPORTEDOBJECT` / `ANSCKRECORDMETADATA` / `ANSCKEXPORTOPERATION` — CloudKit export queue and mirror state.

When you UPDATE a row via raw SQLite, **nothing is written to those tables**. From `voicememod`'s perspective (the iCloud sync daemon), the record hasn't changed and there's nothing to push. The Voice Memos app on Mac reads the underlying SQLite row directly, so the local UI updates — but CloudKit never finds out.

To do this "properly" via direct DB writes, you'd need to also insert correct rows into `ATRANSACTION`, `ACHANGE`, and the `ANSCK*` export tables — replicating what Core Data does internally. That's fragile, undocumented, and one schema bump from Apple will break it.

#### Verified path: UI automation through Voice Memos

The verified sync-safe method is to drive the Voice Memos UI programmatically, the same way a person would rename a memo. This goes through the app's Core Data stack, which records persistent history and queues the CloudKit export.

Prerequisites:

- Voice Memos must open cleanly.
- The automation host must have Accessibility permission: System Settings → Privacy & Security → Accessibility. Enable the terminal app, Codex host, or script runner that will drive the UI.
- Do not touch the iPhone directly. The test is whether iCloud sync carries the Mac-side rename to iPhone.

High-level automation flow:

1. Open Voice Memos.
2. Select one target memo by current visible title/date/duration, or by using the Voice Memos search field.
3. Click the title field in the details pane.
4. Select all text in the title field, type the new title, and press Return.
5. Verify the Mac UI shows the new title.
6. Verify Core Data/CloudKit metadata moved for that one memo.
7. Ask the user to confirm the iPhone shows the new title, usually within 60 seconds.
8. Only after that one-memo gate passes, bulk-rename the remaining memos with the same UI path. Do not require iPhone confirmation after every batch unless CloudKit metadata stops advancing or pending upload/export rows remain.

Codex Computer Use worked on macOS 26.5 once Voice Memos was stable. The detail title field was accessible as a settable text field after selecting a memo. A reliable interaction was:

1. Select memo in the Voice Memos list.
2. Click the settable title text field in the detail pane.
3. Press Command-A.
4. Type the new title.
5. Press Return.

AppleScript/System Events can use the same accessibility path, but Voice Memos has no useful AppleScript scripting dictionary. Treat it as UI scripting, not app scripting.

#### Bulk rename workflow

For large libraries, do not try to rename everything in one pass. Use small batches that can be recovered safely.

1. Rebuild a title-candidate manifest from the current `CloudRecordings.db`.
2. Select only generic `New Recording...` rows that are 10 seconds or longer.
3. Skip rows that already have real titles, rows under 10 seconds, failed/corrupt audio, empty/missing transcripts, and transcripts too thin or sensitive to title safely.
4. Generate concise descriptive titles from transcript excerpts.
5. Save a proposal manifest before renaming.
6. Back up `CloudRecordings.db` with SQLite `.backup` before each rename batch.
7. Rename only through the Mac Voice Memos UI.
8. Verify read-only in SQLite before moving to the next batch.
9. Keep a running report with batch names, backups, verification results, and skipped-row reasons.

Recommended batch size: 10-15 memos. Ten is slower but easier to recover if Voice Memos focus or search behaves unexpectedly.

Useful candidate-query shape:

```bash
DB=~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db

sqlite3 -header -csv "$DB" "
SELECT Z_PK,
       COALESCE(ZCUSTOMLABELFORSORTING, '') AS current_title,
       ZPATH,
       ROUND(ZDURATION, 1) AS duration_secs,
       datetime(ZDATE + 978307200, 'unixepoch', 'localtime') AS recorded_at
FROM ZCLOUDRECORDING
WHERE ZPATH IS NOT NULL
  AND ZPATH != ''
  AND ZDURATION >= 10
  AND ZCUSTOMLABELFORSORTING LIKE 'New Recording%'
ORDER BY ZDATE DESC;
"
```

Back up before each UI batch:

```bash
sqlite3 "$DB" ".backup '$HOME/Documents/Voice-Memos-Organized/CloudRecordings.db.pre-ui-batch-$(date +%Y%m%d-%H%M%S)'"
```

Verified result from the May 2026 production run:

- The first UI rename was confirmed on iPhone.
- Follow-on batches were completed through Mac Voice Memos UI automation only.
- Each batch was verified with Mac title columns, Core Data change rows, no pending upload/export rows, and CloudKit export metadata.
- Some older recordings had legacy-null per-row export markers; those were logged explicitly and only continued when `ZNEEDSUPLOAD = 0` and no pending upload/export rows remained.
- Final verified state: 410 total rows, 338 descriptive titles, 72 generic titles remaining, 38 tiny generic rows under 10 seconds, and 34 useful generic rows skipped because their transcripts were empty, missing, failed/corrupt, too thin, or too sensitive/ambiguous to title safely.

#### Validate one rename before bulk work

After renaming one memo through the UI, verify the DB and CloudKit mirror state. Substitute the audio basename and the row primary key:

```bash
DB=~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db
BASENAME="20260512 084706-68FCCD72"

sqlite3 "$DB" "
SELECT Z_PK, ZCUSTOMLABELFORSORTING, quote(ZENCRYPTEDTITLE), ZPATH
FROM ZCLOUDRECORDING
WHERE ZPATH LIKE '%' || '$BASENAME' || '%';

SELECT COUNT(*) AS transaction_count, MAX(Z_PK) AS max_transaction
FROM ATRANSACTION;

SELECT Z_PK, ZENTITYPK, ZTRANSACTIONID, hex(ZCOLUMNS)
FROM ACHANGE
ORDER BY Z_PK DESC
LIMIT 10;
"
```

Expected result:

- `ZCUSTOMLABELFORSORTING` matches the new title.
- `ZENCRYPTEDTITLE` contains the same title as plain UTF-8 bytes. Do not set it to `NULL`.
- A recent `ACHANGE` row exists for the renamed `ZCLOUDRECORDING.Z_PK`.
- `ATRANSACTION` advanced after the UI rename.

Then check the CloudKit metadata for the renamed row:

```bash
PK=<ZCLOUDRECORDING_Z_PK_FROM_QUERY_ABOVE>
sqlite3 "$DB" "
SELECT ZENTITYPK, ZLASTEXPORTEDTRANSACTIONNUMBER, ZNEEDSUPLOAD
FROM ANSCKRECORDMETADATA
WHERE ZENTITYPK = $PK;
"
```

Expected result: `ZLASTEXPORTEDTRANSACTIONNUMBER` reaches the new transaction number. In the verified May 2026 test, this was enough for the iPhone title to update after the user reopened Voice Memos.

For every batch, also check there is no pending CloudKit work:

```bash
sqlite3 "$DB" "
SELECT COUNT(*) AS pending_record_metadata
FROM ANSCKRECORDMETADATA
WHERE COALESCE(ZNEEDSUPLOAD,0) != 0
   OR COALESCE(ZPENDINGEXPORTCHANGETYPENUMBER,0) != 0
   OR COALESCE(ZPENDINGEXPORTTRANSACTIONNUMBER,0) != 0;

SELECT COUNT(*) AS export_operations
FROM ANSCKEXPORTOPERATION;

SELECT MAX(ZLASTEXPORTEDTRANSACTIONNUMBER) AS max_exported_transaction
FROM ANSCKRECORDMETADATA;
"
```

If older rows have `NULL` `ZLASTEXPORTEDTRANSACTIONNUMBER` before and after the UI rename, document this as a legacy-null metadata caveat. Continue only if title columns match, `ACHANGE` rows exist, `ZNEEDSUPLOAD = 0`, and pending metadata/export-operation counts are zero.

#### Crash recovery after DB restore or experimentation

Voice Memos can crash on launch if `voicememod` keeps stale Core Data/SQLite state after the DB file has been replaced. The crash may surface as an `NSFetchedResultsController` abort in `~/Library/Logs/DiagnosticReports/VoiceMemos-*.ips`, even when `ZCLOUDRECORDING` itself is clean.

Before touching the DB, quit Voice Memos and verify:

```bash
pgrep -lf "VoiceMemos.app/Contents/MacOS/VoiceMemos" && echo "still running" || echo "ok"
```

If restoring or replacing the DB:

1. Back up the current DB first.
2. Copy `CloudRecordings.db` and matching `CloudRecordings.db-wal` / `CloudRecordings.db-shm` sidecars together if those sidecars exist for the backup being restored.
3. Restart `voicememod` after the file replacement.
4. Launch Voice Memos and verify no new crash report appears.

Commands:

```bash
DB=~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db
cp "$DB" ~/Documents/Voice-Memos-Organized/CloudRecordings.db.backup-$(date +%Y%m%d-%H%M%S)

# Quit Voice Memos first, then:
killall voicememod 2>/dev/null || true
open -a VoiceMemos
```

Do not rely on `launchctl kickstart -k gui/$(id -u)/com.apple.voicememod`; on macOS 26.5 it can fail with SIP restrictions. `killall voicememod` is enough; launchd restarts it on demand.

Useful log query while diagnosing launch crashes:

```bash
/usr/bin/log show --debug --info --last 5m --style compact \
  --predicate 'process == "VoiceMemos" OR process == "voicememod"'
```

Look for `voicememod` persistent-store errors, SQLite `disk I/O error`, or XPC store failures. If those appear immediately after DB replacement, restart `voicememod` before changing title rows again.

#### Direct SQLite rename attempts

Do not provide a direct-SQLite rename workflow in this skill. Raw SQLite title edits can make the Mac app appear renamed locally, but they bypass Core Data persistent history and do not reliably sync to iPhone. For public use, treat direct database updates as unsupported except for read-only inspection and `.backup` snapshots.

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
