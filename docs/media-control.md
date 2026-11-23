# Voice control of media (Music Assistant and Sonos)

Status: planned, not yet tested.

## Plan

- Home Assistant 2025.6 and later has a built-in Search and Play intent that works with Music Assistant players, with no blueprints, custom sentences or LLM. Pause, resume, next, previous, volume and mute are built in too, so with "Prefer handling commands locally" they should be fast.
- Expose the Music Assistant player entities to Assist (Settings, Voice assistants, Expose) and assign each an area. With no player named, music plays on a player in the satellite's area.
- Dashboard cards (Sonos card, Music Assistant card) are display only; voice acts on the media player entities.

## Duplicate entities

Speakers exist as both native Sonos and Music Assistant entities. Spotify is played through the Sonos app because the Music Assistant Spotify setup did not work. Give the two entities different spoken names, for example "kitchen speaker" (Music Assistant) and "kitchen Spotify" (Sonos), so Assist can tell them apart and so duplicates do not confuse name matching.

## Known limits

- Playback commits to the first search result, so be specific ("play the album Rumours").
- Shuffle, repeat, clearing the queue and grouping need custom sentences.
- Spotify content played through the Sonos entity probably cannot be found by the built-in search; basic transport and volume should still work.

Reference: Music Assistant voice control documentation, https://www.music-assistant.io/integration/voice/
