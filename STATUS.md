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
- [ ] Doorbell-press trigger itself not separately confirmed (uses the same
      automation/action as motion, so expected to work, just not yet
      explicitly tested by pressing the physical button)
- [ ] Full reboot test (restart the desktop, confirm everything comes back
      without manual steps) not yet explicitly confirmed

Neither open item is blocking — flagging them here in case they come up
later, not as outstanding work.

## Standing instructions
- Never attempt Ring 2FA login on Lauren's behalf.
- Never attempt Home Assistant UI steps or physical doorbell testing on
  Lauren's behalf.
- If Docker Desktop/WSL2 issues come up, don't work around them with guessed
  Windows-specific fixes — report the blocker.
