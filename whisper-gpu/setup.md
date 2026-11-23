# GPU Whisper setup (no Docker)

Runs `wyoming-faster-whisper` directly on an LXC that already has a working NVIDIA driver and CUDA.

## Why not the stock image

`rhasspy/wyoming-whisper` is CPU only. On CPU, `small.en` loaded as float32 took ~9 s per command and needed more than 2 GB of RAM (it was OOM-killed in a 2 GB LXC with no swap).

## Install

```bash
git clone https://github.com/OHF-Voice/wyoming-faster-whisper.git
cd wyoming-faster-whisper
script/setup

# CTranslate2 needs CUDA 12 libraries. Pascal cards (P102-100) are not supported by the CUDA 13 toolkit,
# so avoid anything built for cu13 even if the driver reports CUDA 13.
.venv/bin/pip install nvidia-cublas-cu12 "nvidia-cudnn-cu12==9.*"
```

## Try it by hand first

```bash
export LD_LIBRARY_PATH=/root/wyoming-faster-whisper/.venv/lib/python3.12/site-packages/nvidia/cublas/lib:/root/wyoming-faster-whisper/.venv/lib/python3.12/site-packages/nvidia/cudnn/lib:$LD_LIBRARY_PATH
script/run --model small.en --language en --device cuda --compute-type int8_float32 \
  --uri 'tcp://0.0.0.0:10300' --data-dir /var/lib/wyoming-whisper --download-dir /var/lib/wyoming-whisper
```

Say a command and check `nvidia-smi` for a Python process using VRAM (about 460 MB for `small.en`).

## Install as a service

```bash
cp wyoming-whisper-gpu.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now wyoming-whisper-gpu
journalctl -u wyoming-whisper-gpu -f
```

Reboot once to confirm it starts on boot.

## Home Assistant side

1. Settings, Devices & services, add a Wyoming Protocol entry pointing at this host, port `10300`.
2. Set it as the speech-to-text engine in the pipeline.
3. Remove any old CPU Whisper entry.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `Library libcublas.so.12 is not found` | `LD_LIBRARY_PATH` not set for the process | Set it in the unit file (`Environment=`) |
| `stt-stream-failed` right after you stop speaking | Same, crash on first transcription | Same |
| `Killed` with no Python traceback | Out of memory | More RAM or swap, or a smaller model |
| Float16 compute warning | CPU or Pascal GPU cannot do efficient float16 | Use `int8_float32` (GPU) or `int8` (CPU) |
