# Optimize Home Assistant Voice Assist Pipeline

A fully local Home Assistant Assist setup: fast commands handled by the built-in intent engine, open-ended conversation and tool use handled by a local LLM, GPU-accelerated speech-to-text, and ESPHome voice satellites.

## Architecture

| Stage | Component | Where it runs |
|---|---|---|
| Wake word | microWakeWord on the satellite (openWakeWord container available as an alternative) | Satellite / Wyoming LXC |
| Speech-to-text | `wyoming-faster-whisper`, `medium.en`, CUDA, `int8_float32` | GPU LXC (Pascal P102-100) |
| Conversation | Ollama, Gemma 4 12B, 12288 context, "Prefer handling commands locally" on | GPU node |
| Text-to-speech | Piper (`en_US-hfc_male-medium`) over Wyoming | Wyoming LXC (Docker) |
| Satellites | M5Stack Echo Pyramid + AtomS3R (ESPHome) | Room |

Home Assistant sits in the middle and calls each service in turn, so the services never talk to each other directly and can live on different Proxmox nodes.

## Results

| Change | Before | After |
|---|---|---|
| STT, `tiny-int8` on CPU | ~1.2 s, often empty transcripts | n/a (accuracy too low) |
| STT, `small.en` on CPU (float32) | ~9.4 s | n/a (too slow) |
| STT, `small.en` on GPU | n/a | ~0.1 to 0.2 s |
| STT, `medium.en` on GPU | n/a | (not yet measured) |
| Simple command, local intent match | n/a | ~12 ms intent time |
| Forecast question via LLM tool call | wrong answer (guessed) | correct, ~4.8 s end to end |

## Repo layout

- `whisper-gpu/wyoming-whisper-gpu.service`: systemd unit for GPU Whisper (no Docker).
- `whisper-gpu/setup.md`: install steps and CUDA library notes.
- `home-assistant/scripts/forecast_lookup.yaml`: forecast script with clothing suggestions, exposed to the LLM as a tool.
- `home-assistant/conversation-agent-prompt.md`: the conversation agent instructions.
- `docs/troubleshooting.md`: problems hit along the way and their fixes.
- `docs/media-control.md`: notes on voice control of Music Assistant and Sonos.

## Key lessons

1. **Do not use `--vad-endpointing` with satellites.** It makes Whisper report `requires_external_vad: false`, Home Assistant stops detecting end of speech, and satellite runs hang at the STT stage forever. Browser tests still work, which makes it confusing. Use `--vad-filter` instead.
2. **Let the local intent engine do the easy work.** Distinct entity names (no trailing numbers like "light 2") raise the local match rate and skip the LLM entirely.
3. **LLMs guess unless told not to.** The prompt must say to call tools for weather and state, and to say "I don't know" otherwise.
4. **Context size matters.** 6000 tokens was too small once the entity list, tool definitions and prompt were included. 12288 works, at the cost of splitting the 12B model across two 10 GB cards.
5. **Cold start.** The first request after a model (re)load took ~12 s. Later requests took ~2 s per LLM call.
6. **Whisper model size.** Started at `tiny-int8`, moved to `small.en`, now running `medium.en` on the GPU. Check `nvidia-smi` with the LLM loaded to make sure both fit (`small.en` used about 460 MB).

## Current status

Lights were renamed so none end in a number, the old CPU Whisper entry was removed from the Wyoming integration, and Whisper now runs `medium.en`. Media control is still untested (see `docs/media-control.md`).

## Notes on privacy

Network addresses, Wi-Fi details, device IDs and tokens are intentionally left out of this repo. Replace placeholders like `<HA_TOKEN>` with your own values and keep secrets out of version control.
