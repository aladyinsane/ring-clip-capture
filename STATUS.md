# Status

Current state only. What this is and the decisions behind it: [`README.md`](README.md).

## Waiting on Lauren

- The doorbell battery read 10% on 2026-11-19. Charge it (planned for the weekend of 2026-11-21).

## In progress

(none)

## Ready

(none)

## Done

- The stack runs from `ring-clips/docker-compose.yml`: `mosquitto`, `ring-mqtt`, `homeassistant`, `cleanup`, all `restart: unless-stopped`.
- Doorbell "Front Door" (doorbell_v4) is in Home Assistant through ring-mqtt and the MQTT integration. Camera entity `camera.front_door` streams from ring-mqtt's RTSP relay.
- Automation `automation.record_ring_doorbell_clip` records 20 seconds to `C:\ring-clips` on ding or motion. Motion was confirmed in August 2026, and a real ding on 2026-08-13 12:57 CT. Lauren found a clip from 2026-11-19 14:39 CT, after the stack was recreated for PR #2.
- MQTT (1883) and the RTSP relay (8554) are published on 127.0.0.1 only (PR #2, merged 2026-11-19). An independent review passed it.
- `cleanup` deletes clips older than `RETENTION_DAYS` (3).
- Survives a reboot. Confirmed by a reboot test once Docker Desktop's "Start Docker Desktop when you sign in" was turned on.
- Phone access through Tailscale and the Home Assistant app works on Lauren's phone. Steps for adding another phone are in `SETUP_NOTES.md`.
