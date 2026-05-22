# Contributor Notes

This is a Claude Code skill for organizing Apple Voice Memos. End-user docs live in `SKILL.md` (loaded by Claude on every invocation) and `README.md` (install guide).

## Repo Layout

- `SKILL.md` — the skill definition. Read by Claude when the skill is invoked. Keep concise; users don't see it directly.
- `README.md` — install + usage for humans browsing the repo.
- `TROUBLESHOOTING.md` — linked from `SKILL.md` for edge cases.
- `CLAUDE.md` — this file. Notes for contributors editing the skill.

## Voice Memos Database

Voice Memos titles live in `CloudRecordings.db` at
`~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/`.

Relevant columns in `ZCLOUDRECORDING`:

- `ZCUSTOMLABEL` — usually an ISO timestamp for generic/untitled memos. It is not the display title on current macOS.
- `ZCUSTOMLABELFORSORTING` — display title shown in Voice Memos, e.g. `New Recording 309` or a user-provided title.
- `ZENCRYPTEDTITLE` — misnamed; current Voice Memos stores the display title as plain text here too. Keep it equal to `ZCUSTOMLABELFORSORTING`. Do not clear it.
- `ZPATH` — the audio filename in the same folder.
- `ZDATE` — Apple epoch (seconds since 2001-01-01). Add `978307200` for Unix epoch.

## Verifying the Rename Approach (Step 7)

`CloudRecordings.db` is iCloud-synced via CloudKit. Raw SQLite title updates are not sync-safe on current macOS because they bypass Core Data persistent history and CloudKit export queues. The only verified production rename path is Mac Voice Memos UI automation.

Production rules:

- Do not rename by direct SQLite updates.
- Do not set `ZENCRYPTEDTITLE = NULL`; this can crash Voice Memos on next launch.
- Back up `CloudRecordings.db` with `sqlite .backup` before each UI rename batch.
- Rename through the Mac Voice Memos app only.
- Verify read-only in SQLite after each batch:
  - `ZCUSTOMLABELFORSORTING` equals the proposed title.
  - `ZENCRYPTEDTITLE` equals the proposed title.
  - `ACHANGE` rows exist for the renamed `ZCLOUDRECORDING` primary keys.
  - `ZNEEDSUPLOAD = 0`.
  - No pending metadata or export operation rows remain.
  - `ANSCKRECORDMETADATA.ZLASTEXPORTEDTRANSACTIONNUMBER` advances when available. Some older rows may have legacy-null export markers; if so, explicitly document the caveat and rely on title columns plus no pending upload/export rows.

If changing rename instructions, re-run a single-memo UI rename and confirm Mac SQLite metadata plus iPhone sync before claiming the public skill is verified.

**Last verified:** 2026-05 on macOS 26 using Mac Voice Memos UI automation. UI renames updated local title columns, wrote Core Data change rows, and synced to iPhone after the initial user-confirmed gate.

## Transcription Engine: Parakeet MLX

The skill uses [`parakeet-mlx`](https://github.com/senstella/parakeet-mlx) — an Apple Silicon native port of NVIDIA's Parakeet TDT model. Defaults to `mlx-community/parakeet-tdt-0.6b-v3` (multilingual, ~2.3 GB).

**Why not whisper.cpp:** Parakeet matches whisper's speed in batch mode but produces properly punctuated and capitalized output, lowering downstream LLM-summarization noise. WER is also competitive (1.69% on LibriSpeech test-clean).

**Why not the NeMo upstream:** NeMo requires CUDA; only the mlx-community port runs natively on M-series GPUs.

**Verified 2026-05 on M3 Ultra:** ~60× realtime in batch mode (8 min of audio in ~8s wall time). Whisper.cpp remains the recommended fallback for non-European languages — see `TROUBLESHOOTING.md`.

## Pull Request Guidelines

- Keep examples generic — no personal data, no business names, no real recording titles.
- All processing must stay local (no API keys required). The skill's whole point is no third-party services touching your audio.
- If you change the rename logic in Step 7, re-run the single-memo UI verification above and update the "Last verified" line.
- If you change the transcription engine or model, re-run the batch benchmark and update the performance line in `SKILL.md` Step 4.
- Test on a backup DB before iterating on live data.
