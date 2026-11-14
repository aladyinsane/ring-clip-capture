# Ring Doorbell Clip Capture — Setup Notes

Reference doc for how this was built, and how to rebuild or troubleshoot it.
This is not a forward-looking plan — it describes the actual working setup.
For current run status, see `STATUS.md`.

## What this is
Self-hosted capture of Ring doorbell clips (ding + motion) without a Ring
Protect subscription, running on this Windows desktop via Docker Desktop.
Home Assistant records a short local clip whenever the doorbell rings or
detects motion; clips auto-purge after a few days. No cloud storage, no
subscription required.

## Architecture
```
Ring Cloud  <--->  ring-mqtt (bridge, RTSP relay)  <--->  Mosquitto (MQTT broker)
                                                              |
                                                        Home Assistant
                                                        (automation: on ding/motion -> record clip)
                                                              |
                                                     C:\ring-clips (local folder,
                                                     desktop's own drive)
                                                              |
                                                     cleanup service (deletes files > N days old)
```
Ring doorbells stream on-demand only (no 24/7 feed), so clips start a few
seconds *after* the trigger fires — this setup cannot retrieve Ring's own
pre-recorded clip without a subscription.

## Prerequisites
- Docker Desktop for Windows, WSL2 backend enabled
- Windows sleep/screen-off set to "Never" while plugged in; stay signed in
  (Docker Desktop only runs after sign-in)
- A Ring account with the doorbell already added in the Ring app (no
  subscription required)
- The Ring account's email/password on hand, plus phone/email access for a
  2FA code (needed once, manually, during initial auth)
- Tailscale installed and running on this desktop (for remote/mobile access)

## Repo structure
```
ring-clips/
├── docker-compose.yml
├── .env.example
├── .env                    # gitignored — copy of .env.example with real values
├── .gitignore
├── mosquitto/
│   └── mosquitto.conf
├── ring-mqtt/               # gitignored — stores Ring refresh token
└── homeassistant/           # gitignored — HA config
```

---

## Rebuilding from scratch

### 1. Base compose stack
**`docker-compose.yml`**
```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    restart: unless-stopped
    volumes:
      - ./mosquitto:/mosquitto/config
      - mosquitto-data:/mosquitto/data
    ports:
      - "1883:1883"

  ring-mqtt:
    image: tsightler/ring-mqtt:latest
    restart: unless-stopped
    volumes:
      - ./ring-mqtt:/data
    ports:
      - "8554:8554"
    depends_on:
      - mosquitto

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    volumes:
      - ./homeassistant:/config
      - C:/ring-clips:/media/ring-clips
    ports:
      - "8123:8123"
    depends_on:
      - mosquitto
      - ring-mqtt

  cleanup:
    image: alpine:latest
    restart: unless-stopped
    environment:
      - RETENTION_DAYS=${RETENTION_DAYS:-3}
    volumes:
      - C:/ring-clips:/media/ring-clips
    entrypoint: >
      sh -c "while true; do find /media/ring-clips -type f -mtime +$$RETENTION_DAYS -delete; sleep 86400; done"

volumes:
  mosquitto-data:
```
**Important:** don't use `network_mode: host` for `homeassistant` — host
networking is unreliable on Docker Desktop for Windows. Explicit port mapping
(`8123:8123`) is used instead; the only cost is mDNS auto-discovery, which
isn't needed since Ring is added manually via MQTT.

Clips live at `C:\ring-clips` on the desktop's own drive — no network share
or external drive involved. Since nothing is retained past `RETENTION_DAYS`,
total size stays in the low GB range regardless of where it lives.

**`mosquitto/mosquitto.conf`**
```
listener 1883
allow_anonymous true
persistence true
persistence_location /mosquitto/data/
```
(Anonymous access is fine — the port isn't forwarded to the internet.)

**`.env`**
```
RETENTION_DAYS=3
```

**`.gitignore`**
```
.env
ring-mqtt/
homeassistant/
```

### 2. Ring authentication (one-time, manual — requires live 2FA code)
Run from the `ring-clips` folder:
```powershell
docker run -it --rm --mount type=bind,source="${PWD}/ring-mqtt",target=/data --entrypoint /app/ring-mqtt/init-ring-mqtt.js tsightler/ring-mqtt
```
**Note:** plain `docker run -v ./ring-mqtt:/data tsightler/ring-mqtt` (no
entrypoint override) does *not* prompt for login on ring-mqtt v5.x — it
starts the main service directly, finds no config, and exits. The
`--entrypoint /app/ring-mqtt/init-ring-mqtt.js` override is required to run
the actual token-acquisition CLI (source: [ring-mqtt wiki, Installation
(Docker)](https://github.com/tsightler/ring-mqtt/wiki/Installation-(Docker))).

Follow the prompts (email, password, 2FA code). At the "MQTT URL" prompt,
enter `mqtt://mosquitto:1883` (the mosquitto service name + port, matching
the docker-compose network). Once it confirms success, the refresh token is
saved to `ring-mqtt/ring-state.json` (with `ring-mqtt/config.json` alongside
it) — the regular compose-managed container reuses this on every future
start. This step is only needed once, unless the token is lost/revoked.

ring-mqtt v5.x reads the broker address from `config.json`'s `mqtt_url`
field — there's no `MQTTHOST`/`MQTTPORT` env var support anymore, don't add
those to the `ring-mqtt` service.

### 3. Bring up the stack
```powershell
docker compose up -d
docker compose logs -f
```
Verify: `mosquitto` and `ring-mqtt` show no errors, `ring-mqtt` logs show the
doorbell being discovered (device type + ID), `homeassistant` becomes
reachable at `http://localhost:8123`.

### 4. Home Assistant configuration
1. Complete HA's first-run onboarding (create admin account) at
   `http://localhost:8123`
2. Settings → Devices & Services → Add Integration → **MQTT** → broker
   `mosquitto`, port `1883`. Confirm the doorbell's entities appear
   automatically (motion sensor, ding sensor, battery, etc.)
3. Settings → Devices & Services → Add Integration → **Generic Camera** →
   stream URL: `rtsp://ring-mqtt:8554/<device-id>_live` (get `<device-id>`
   from the ring-mqtt logs)
4. Note the exact entity IDs assigned (rename if desired) — needed for the
   recording automation below

**Known cosmetic bug:** the Generic Camera setup wizard's live preview
thumbnail may never render (seen in both Firefox and Edge) even though the
stream is completely healthy. Don't take a stuck spinner as a sign of a
broken stream — confirm the RTSP feed directly instead:
```powershell
docker compose exec ring-mqtt sh -c "timeout 20 /app/ring-mqtt/node_modules/ffmpeg-for-homebridge/ffmpeg -rtsp_transport tcp -i rtsp://127.0.0.1:8554/<device-id>_live -t 8 -f null - 2>&1"
```
If that runs clean (valid H264/AAC output, no errors), the stream is fine —
submit the integration anyway.

### 5. Recording automation
Write directly to `homeassistant/automations.yaml` (or paste into the HA
automation editor). This is what's actually running, with real entity IDs:
```yaml
- alias: "Record Ring Doorbell Clip"
  trigger:
    - platform: state
      entity_id: binary_sensor.front_door_ding
      to: "on"
    - platform: state
      entity_id: binary_sensor.front_door_motion
      to: "on"
  action:
    - service: camera.record
      target:
        entity_id: camera.front_door
      data:
        filename: "/media/ring-clips/{{ now().strftime('%Y-%m-%d_%H-%M-%S') }}_{{ trigger.to_state.entity_id.split('.')[1] }}.mp4"
        duration: 20
```
**Gotcha:** `homeassistant/automations.yaml` must be a bare YAML list, with
**no** `automation:` wrapper key — `configuration.yaml` already does
`automation: !include automations.yaml`. Adding the wrapper here (as you
would if pasting straight into `configuration.yaml` instead) breaks it.

Restart HA after editing this file directly for changes to take effect
(`docker compose restart homeassistant`), since it isn't picked up by the
"reload automations" UI action when edited outside the editor in the same
way new entities are.

### 6. Retention
Already handled by the `cleanup` service from step 1 — checks daily, deletes
files older than `RETENTION_DAYS`. No separate cron/Task Scheduler setup
needed.

---

## Mobile access via Tailscale

Home Assistant is only published to the Windows host (`8123:8123`), not to
the internet — Tailscale is the way to reach it remotely instead of
forwarding a router port.

### For yourself
1. Confirm Tailscale is running on this desktop and note its Tailscale IP:
   ```powershell
   tailscale ip -4
   ```
2. Install Tailscale on your phone, sign into the same Tailscale account/tailnet.
3. Install the official **Home Assistant** mobile app.
4. On first launch, when it asks for a server address, enter
   `http://<tailscale-ip>:8123` (use the IP from step 1, not `localhost`).
5. If it won't connect, check Windows Defender Firewall → allow inbound TCP
   8123 for the network profile Tailscale's interface is on (Docker Desktop's
   port mapping is sometimes blocked on the "Public" profile Windows assigns
   to new interfaces, including Tailscale's).

### Setting up someone else's phone
Same idea, from their side:
1. They install the **Tailscale** app on their phone and sign in — you need
   to have already invited them to your tailnet (Tailscale admin console →
   add a user, or share the machine), since only devices on the same tailnet
   can reach this desktop's Tailscale IP.
2. They install the official **Home Assistant** mobile app.
3. When the app asks for a server address, they enter
   `http://<this-desktop's-tailscale-ip>:8123` (same IP as above — get it
   fresh with `tailscale ip -4` in case it changed).
4. They'll log in with the same HA admin account credentials (or you can
   create them a separate HA user under Settings → People, if you want
   them to have their own login rather than sharing yours).
5. Same firewall caveat applies if the connection fails to establish.

---

## Operational notes
- `ring-mqtt` uses Ring's unofficial API (the same one the mobile app uses)
  — not officially supported by Ring/Amazon, could break if Ring changes
  their API. Low risk for personal use, but not guaranteed long-term.
- Don't point a continuous NVR tool (e.g. Frigate) at these cameras — the
  ring-mqtt maintainer explicitly warns this interferes with ding/motion
  event delivery, since Ring cameras aren't designed for always-on streaming.
- Clips are stored locally at `C:\ring-clips` rather than on separate/network
  storage — fine here since nothing is retained past `RETENTION_DAYS` (a few
  days of 20-second clips is at most a few GB).

## Troubleshooting quick reference
| Symptom | Cause / fix |
|---|---|
| `ring-mqtt` crash-loops with "No configuration file found at /data/config.json" | Expected before Phase 2 (auth) has been run once. If it happens *after* auth previously worked, the token file may have been lost — re-run the Phase 2 auth command. |
| Generic Camera setup wizard preview spinner never stops | Cosmetic bug, not a real failure — verify the stream directly with the `ffmpeg` probe command above, then submit anyway. |
| Automation doesn't seem to exist / doesn't show in Settings → Automations | If it was added directly to `automations.yaml` without an explicit `id:` field, it may not show reliably in the UI list depending on HA version. Verify it's loaded via Developer Tools → States, searching for `automation.<slug-of-alias>`. |
| Containers don't come back after a reboot | Docker Desktop's "Start Docker Desktop when you sign in" setting (Settings → General) must be on — confirmed via an actual reboot test that this is off by default and the stack stays down without it. Also confirm Docker Desktop wasn't fully quit (vs. just closing its window) before the reboot. All services use `restart: unless-stopped`, so once the engine is up they self-start on their own. |
