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

## Whisper Fallback (for non-European languages)

Parakeet v3 doesn't cover Mandarin, Japanese, Arabic, etc. For those, swap in whisper.cpp:

```bash
brew install whisper-cpp
mkdir -p ~/Documents/Voice-Memos-Organized/models
curl -L -o ~/Documents/Voice-Memos-Organized/models/ggml-base.bin \
  "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin"
```

Then in Step 4, replace the parakeet-mlx loop with whisper-cli (ffmpeg → 16kHz mono WAV → whisper-cli → txt). The Homebrew formula installs the binary as `whisper-cli` (not `whisper-cpp`).

## Common Issues

- **`parakeet-mlx` not found**: Run `pipx install parakeet-mlx`. If pipx isn't on `PATH`, run `pipx ensurepath` and restart the shell.
- **First run hangs**: First invocation downloads ~2.3 GB of model weights. Watch `~/.cache/huggingface/hub/` to confirm progress.
- **`.qta` files fail**: These are temporary in-progress recordings. The batch script continues past failed chunks. If a memo is critical, open Voice Memos on Mac and let it finalize the recording (the `.qta` should become a `.m4a`).
- **Out of memory on long batches**: The Step 4 script chunks by 50 files. Reduce the chunk size if you're on a Mac with less than 16 GB RAM.
- **Transcripts look like nonsense**: Confirm the audio is in a language Parakeet v3 supports. If not, use the whisper fallback above.
