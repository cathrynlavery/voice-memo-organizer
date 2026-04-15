# Troubleshooting

## Full Disk Access

Voice Memos are stored in a protected directory. If `ls` returns "Operation not permitted":

1. Open **System Settings > Privacy & Security > Full Disk Access**
2. Enable the toggle for the terminal app (Terminal, iTerm2, Warp, etc.)
3. Restart the terminal session

## iCloud Sync

If the Recordings directory is empty or missing:

1. Open the **Voice Memos** app on the Mac
2. Wait for iCloud to finish syncing (check the progress indicator)
3. Recordings from iPhone will appear once sync completes

## Whisper Model Selection

| Model | File | Size | Speed | Use Case |
|-------|------|------|-------|----------|
| base.en | `ggml-base.en.bin` | ~150MB | ~1 min/sec | English-only (recommended default) |
| base | `ggml-base.bin` | ~150MB | ~1 min/sec | Multilingual memos |
| medium.en | `ggml-medium.en.bin` | ~500MB | ~3 min/sec | Higher accuracy for important recordings |

## Common Issues

- **`whisper-cli` not found**: Run `brew install whisper-cpp` — the binary is `whisper-cli`, not `whisper-cpp`
- **ffmpeg conversion fails**: Some `.qta` files may be corrupted or DRM-protected; the batch script skips these automatically
- **Transcripts are empty**: Check that the whisper model file downloaded completely (`ls -lh` should match expected size)
