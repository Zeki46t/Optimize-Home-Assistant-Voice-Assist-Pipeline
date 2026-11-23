# Troubleshooting log

Problems hit while building this pipeline, in the order they came up.

## Satellite hangs at the STT stage, never replies

**Symptom:** pipeline trace stays at `stage: stt`, no speech events, satellite keeps listening. Browser tests work.

**Cause:** Whisper was started with `--vad-endpointing 0.7`. The trace changes from `requires_external_vad: true` to `false`, so Home Assistant stops deciding when you finished speaking and Whisper never ends the utterance for satellite audio.

**Fix:** remove `--vad-endpointing`, restart the service, reload the Wyoming entry in Home Assistant. Confirm `requires_external_vad: true` in the next trace.

## Whisper invents words on silence ("You", "Thank you")

**Cause:** the pipeline started with no speech (false wake word or silence) and Whisper hallucinates on empty audio.

**Fix:** `--vad-filter` on the Whisper server.

## `stt-no-text-recognized` on real speech

**Cause:** `tiny-int8` model is too weak for quiet or short speech.

**Fix:** `small.en`. On CPU it needs more RAM (see below) and runs float32.

## Whisper container "Killed"

**Cause:** the kernel OOM killer. `small.en` in float32 needs more than 2 GB, and the LXC had no swap.

**Fix:** more LXC memory plus swap, or a smaller model, or move Whisper to the GPU with int8.

## Recreating a Docker container that was not started by Compose

`docker inspect` labels were empty, so there was no compose file. Recreate with the same name, port, volume and restart policy from `docker inspect` (`Mounts`, `Cmd`, `RestartPolicy`). Be careful not to paste placeholder paths from examples; the placeholder `/path/to/whisper-data` would have created an empty data folder and re-downloaded models.

## Wyoming containers refuse connections after a Home Assistant restart

Piper and openWakeWord reconnected after restarting the Docker service on the Wyoming node. Cause not confirmed (suspected Docker port-forwarding state). After reboots, check that ports are listening:

```bash
ss -tlnp | grep -E '10200|10300|10400'
```

## LLM answers forecasts with made-up weather

**Cause:** prompt only required tools for device actions, and there was no forecast tool. See `home-assistant/conversation-agent-prompt.md` and `home-assistant/scripts/forecast_lookup.yaml`.

## Saying the script name runs it instead of asking the LLM

An exposed script called "Get weather forecast" matched locally ("Started" reply). Rename the alias to something people do not say.

## Template shows "variable is undefined" in Developer Tools

Script variables only exist while the script runs. Test scripts from Developer Tools, Actions, not the Template page.

## Echo Pyramid dropped off the network once at high volume

Seen once at 66% volume right as the speaker started. Did not repeat at 31%. Cause not confirmed; a better power supply is the first thing to try if it recurs.

## Slow first LLM call

About 12 s for the first request after a model load or context change, then about 2 s per call. Keep-alive forever avoids reloads.
