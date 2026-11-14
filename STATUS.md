# Status

**This project is done and running.** For how it's built, how to rebuild it
from scratch, or how to troubleshoot it, see `SETUP_NOTES.md`.

## Current state
- Stack is up via `docker compose` in `C:\dev\ring\ring-clips`: `mosquitto`,
  `ring-mqtt`, `homeassistant`, `cleanup` — all `restart: unless-stopped`, so
  they survive a Docker Desktop restart or a full machine reboot without
  re-running Ring auth.
- Ring doorbell "Front Door" (doorbell_v4) integrated via `ring-mqtt` + HA's
  MQTT integration.
- Camera entity `camera.front_door`, RTSP relay via `ring-mqtt`'s embedded
  go2rtc.
- Recording automation `automation.record_ring_doorbell_clip` is active:
  records a 20-second clip to `C:\ring-clips` whenever the ding or motion
  sensor fires.
- Retention: `cleanup` service purges clips older than `RETENTION_DAYS`
  (currently `3`) from `C:\ring-clips`. No cron/Task Scheduler needed.
- Remote/mobile access: Tailscale + the official Home Assistant mobile app,
  confirmed working on Lauren's phone. Setup steps for adding another
  person's phone are in `SETUP_NOTES.md`.
- n8n integration was considered and explicitly decided against — not
  implemented, no MQTT trigger workflow exists.

## Validation
- [x] Motion-triggered recording confirmed — clip landed in `C:\ring-clips`
- [x] Doorbell-press trigger confirmed via a real ring on 2026-08-13 12:57 CT
      — `binary_sensor.front_door_ding` fired (confirmed in the HA recorder
      DB), one second after `binary_sensor.front_door_motion`. Because the
      automation defaults to `mode: single` and motion fired first, the ding
      trigger was skipped rather than starting its own recording — the
      resulting clip is filename-tagged `front_door_motion`, not `_ding`,
      even though a real doorbell press is what happened. The 20s clip
      window still covers the ding (which landed 1s into it), so the event
      itself isn't missed, just mislabeled when motion and ding land close
      together. Not fixing unless it becomes a real problem — noting here so
      it's not mistaken for a bug later.
- [x] Full reboot test confirmed. First attempt failed — Docker Desktop
      didn't auto-start after reboot, so the stack stayed down. Fixed by
      toggling Docker Desktop's "Start Docker Desktop when you sign in"
      setting on (Settings → General, in the Docker Desktop GUI). Retested
      and confirmed the stack comes back on its own after a reboot.

All open validation items are now closed.

## Standing instructions
- Never attempt Ring 2FA login on Lauren's behalf.
- Never attempt Home Assistant UI steps or physical doorbell testing on
  Lauren's behalf.
- If Docker Desktop/WSL2 issues come up, don't work around them with guessed
  Windows-specific fixes — report the blocker.
