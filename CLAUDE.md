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

- `ZCUSTOMLABEL` — the title users see (plain text).
- `ZCUSTOMLABELFORSORTING` — sort key; usually same string as `ZCUSTOMLABEL`.
- `ZENCRYPTEDTITLE` — present on newer macOS versions. **If non-null, it overrides `ZCUSTOMLABEL` on iPhone.** Clearing it lets the plain label win.
- `ZPATH` — the audio filename in the same folder.
- `ZDATE` — Apple epoch (seconds since 2001-01-01). Add `978307200` for Unix epoch.

## Verifying the Rename Approach (Step 7)

`CloudRecordings.db` is iCloud-synced via CloudKit. Apple has changed Voice Memos' sync model in past releases, so re-verify before shipping schema changes:

1. Quit Voice Memos on Mac **and** iPhone.
2. Back up `CloudRecordings.db` to a safe location.
3. Pick one row (`Z_PK = <id>`), `UPDATE ZCUSTOMLABEL` to something distinctive, set `ZENCRYPTEDTITLE = NULL`.
4. Reopen Voice Memos on Mac. Wait for the sync indicator to clear (~30s).
5. Open Voice Memos on iPhone. Confirm the new title appears.
6. If both update, the approach still works. If not, dump the schema and look for new title fields.

**Last verified:** 2026-05 on macOS 26 (Darwin 25). Single-memo rename propagated to iPhone within ~30s.

## Pull Request Guidelines

- Keep examples generic — no personal data, no business names, no real recording titles.
- All processing must stay local (no API keys required). The skill's whole point is no third-party services touching your audio.
- If you change the rename logic in Step 7, re-run the single-memo verification above and update the "Last verified" line.
- Test on a backup DB before iterating on live data.
