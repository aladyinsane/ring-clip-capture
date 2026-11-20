# Status

Current state only. What this is and the decisions behind it: [`README.md`](README.md).

## Waiting on Lauren

- Review and merge PR #2 (MQTT and RTSP published on localhost only). It's already applied to the running stack, so merging just makes the repo match.
- The doorbell battery read 10% on 2026-11-19. Charge it.

## In progress

(none)

## Ready

(none)

## Done

- The stack runs from `ring-clips/docker-compose.yml`: `mosquitto`, `ring-mqtt`, `homeassistant`, `cleanup`, all `restart: unless-stopped`.
- Doorbell "Front Door" (doorbell_v4) is in Home Assistant through ring-mqtt and the MQTT integration. Camera entity `camera.front_door` streams from ring-mqtt's RTSP relay.
- Automation `automation.record_ring_doorbell_clip` records 20 seconds to `C:\ring-clips` on ding or motion. Motion was confirmed in August 2026, and a real ding on 2026-08-13 12:57 CT. Lauren found a clip from 2026-11-19 14:39 CT, after the stack was recreated for PR #2.
- `cleanup` deletes clips older than `RETENTION_DAYS` (3).
- Survives a reboot. Confirmed by a reboot test once Docker Desktop's "Start Docker Desktop when you sign in" was turned on.
- Phone access through Tailscale and the Home Assistant app works on Lauren's phone. Steps for adding another phone are in `SETUP_NOTES.md`.
