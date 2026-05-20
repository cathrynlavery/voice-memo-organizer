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

## Parakeet Model Notes

`parakeet-mlx` defaults to `mlx-community/parakeet-tdt-0.6b-v3` (~2.3 GB), cached at `~/.cache/huggingface/hub/`. The v3 model handles English plus 24 European languages.

**Faster downloads:** Set a Hugging Face token to avoid anonymous rate limits on the first model fetch.

```bash
export HF_TOKEN=<your-hf-token>
```

**Pinning a different model:** Pass `--model <hf-repo>` or set `PARAKEET_MODEL`. The English-only v2 (`mlx-community/parakeet-tdt-0.6b-v2`) trades language coverage for slightly lower English WER on some benchmarks.

## Whisper Model Selection

Whisper.cpp is documented as a first-class lighter alternative in `SKILL.md` Step 3 and Step 4. Pick whichever model fits your needs:

| Model     | File                 | Size    | Speed         | Use case                                        |
| --------- | -------------------- | ------- | ------------- | ----------------------------------------------- |
| base.en   | `ggml-base.en.bin`   | ~150 MB | ~60× realtime | English only (recommended default)              |
| base      | `ggml-base.bin`      | ~150 MB | ~60× realtime | Multilingual — Mandarin, Japanese, Arabic, etc. |
| medium.en | `ggml-medium.en.bin` | ~500 MB | ~20× realtime | Higher accuracy on important English recordings |

Download whichever you want into `~/Documents/Voice-Memos-Organized/models/`:

```bash
mkdir -p ~/Documents/Voice-Memos-Organized/models
curl -L -o ~/Documents/Voice-Memos-Organized/models/ggml-base.bin \
  "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin"
```

Then in the Step 4 whisper batch loop, update the `MODEL=` line to point at the file you downloaded.

## Common Issues

- **`parakeet-mlx` not found**: Run `pipx install parakeet-mlx`. If pipx isn't on `PATH`, run `pipx ensurepath` and restart the shell.
- **First parakeet run hangs**: First invocation downloads ~2.3 GB of model weights. Watch `~/.cache/huggingface/hub/` to confirm progress.
- **`whisper-cli` not found**: Run `brew install whisper-cpp` — the binary is `whisper-cli`, not `whisper-cpp`.
- **`.qta` files fail**: These are temporary in-progress recordings. The batch script continues past failed chunks. If a memo is critical, open Voice Memos on Mac and let it finalize the recording (the `.qta` should become a `.m4a`).
- **Out of memory with parakeet**: The Step 4 parakeet script chunks by 50 files; reduce the chunk size, or switch to the whisper batch loop (~500 MB RAM vs 3-4 GB).
- **Parakeet transcripts look like nonsense**: Confirm the audio is in a language Parakeet v3 supports (English + 24 European). Otherwise switch to whisper's multilingual `ggml-base.bin`.
