# Ring doorbell clip capture

Saves a short local clip whenever the Ring doorbell rings or sees motion, without a Ring Protect subscription. It runs in Docker Desktop on Lauren's home desktop, and clips go to `C:\ring-clips`. For Lauren's household only.

How it's built, how to rebuild it and how to troubleshoot it: [`SETUP_NOTES.md`](SETUP_NOTES.md). Current state: [`STATUS.md`](STATUS.md).

## What "done" means

- A ding or motion event records a 20-second clip to `C:\ring-clips`.
- Clips older than `RETENTION_DAYS` (3) are deleted automatically.
- The stack comes back on its own after a reboot.
- Lauren can see the doorbell from her phone through the Home Assistant app over Tailscale.

All four have been confirmed (details in STATUS).

## How it fits together

`ring-mqtt` talks to Ring's cloud and relays the camera as RTSP. It publishes events to Mosquitto (MQTT). Home Assistant records a clip when the ding or motion sensor fires. A small `cleanup` container deletes old clips. All four are services in `ring-clips/docker-compose.yml`.

## Constraints

- Never attempt the Ring 2FA login on Lauren's behalf.
- Never use the Home Assistant UI or test the physical doorbell on Lauren's behalf.
- If Docker Desktop or WSL2 causes trouble, report the blocker. Don't work around it with guessed Windows fixes.
- Don't point an always-on recorder (such as Frigate) at the camera. The ring-mqtt maintainer warns it breaks ding and motion events.
- Don't open live streams just to test. Each one wakes the doorbell and drains its battery.

## Key decisions

- **No Ring subscription.** Ring streams on demand only, so clips start a few seconds after the event. Ring's own pre-recorded clip isn't available.
- **ring-mqtt uses Ring's unofficial API**, the same one the Ring app uses. It could break if Ring changes that API. Accepted for personal use.
- **Clips stay on the desktop's own drive**, not a network share. With 3-day retention they stay at a few GB at most.
- **Remote access is Tailscale, not a forwarded router port.**
- **No host networking for Home Assistant.** It's unreliable on Docker Desktop for Windows, so ports are mapped explicitly. The only cost is mDNS discovery, which isn't needed.
- **No n8n integration.** Considered and decided against.
- **Mislabeled clips are accepted.** The automation runs in `mode: single`. When motion fires a second before a ding, the ding doesn't start its own clip, so the clip is named `_motion`. The 20-second clip still covers the ding. Not fixing unless it becomes a real problem.
