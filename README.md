# Schmoose Live

**Schmoose Live** — the voice & video companion for
[Schmoose](https://github.com/DerDast3/schmoose). One click on the video
button in any chat opens a call room that belongs to that chat: same room
for the same people, no second sign-in, no subscription, no telemetry.

This repository contains the **deployment files** for running the Schmoose
Live SFU on your own server. The application ships as a prebuilt container
image from ghcr.io — pull, configure, run.

**Contents:** [What it is](#what-it-is) · [Floor control](#floor-control--one-mic-rules-the-room) ·
[Requirements](#requirements) · [Quick start](#quick-start) ·
[Connect to Schmoose](#connect-to-schmoose) · [Testing without Schmoose](#testing-without-schmoose) ·
[Firewall](#firewall) · [Troubleshooting](#troubleshooting)

## What it is

A **Selective Forwarding Unit** (SFU): every participant sends their media
once to this server, and the server forwards the streams to everyone else.
No media transcoding, no mixing, no storage — the SFU holds nothing on
disk, keeps no history, and forgets everything when the last person leaves
or the process restarts.

What you get:

- **Real WebRTC** via [mediasoup](https://mediasoup.org/) — audio + video
  with TCP fallback when UDP is blocked.
- **Floor control, built in** — see below; this is the feature that makes
  500-person chats possible on small hardware.
- **Schmoose identity, carried over** — name, initials and color theme ride
  in a short-lived signed ticket (Ed25519, ≤ 120 s). The ticket travels in
  the URL fragment and never touches a server log.
- **IPv6-native** — the SFU announces its global v6 address (plus LAN v4)
  for ICE; phones on cellular connect directly without STUN/TURN.
- **No persistence by design** — nothing to back up, nothing to leak.

## Floor control — one mic rules the room

Classic conference calls descend into overlap, echo and "you're muted!"
chaos. Schmoose Live replaces that with a **talking stick**:

- Exactly **one person broadcasts** at a time. Everyone else's camera and
  microphone are closed — not muted, *closed* (LED off, zero upload).
- Anyone can press the button: **Speak** when the floor is free,
  **Raise hand** when someone else holds it. The current speaker then has
  **3 seconds to keep the floor** — press again or hand it over.
- Handovers are instant (the media transports stay warm), so "you're on
  in 3… 2… 1…" actually feels like it.
- The server enforces this: only the floor holder's media is accepted
  (produce gate), no client can cheat around it.

This is what keeps a 500-member chat usable: one sender, everyone else
listening at ~zero cost.

## Requirements

- Linux host with Docker (the container is Debian-based; mediasoup needs
  glibc — on an Alpine host run it inside Docker, don't build natively)
- A hostname with a TLS certificate (e.g. from Let's Encrypt) —
  `fullchain.pem` + `privkey.pem`
- Open ports: **TCP 9443** (web page + WebSocket) and **UDP 40000–40100**
  (media)
- IPv6 strongly recommended (direct paths for mobile clients); IPv4-only
  hosts work on the LAN but phones may fall back to TCP

## Quick start

```sh
mkdir -p /srv/schmoose-live/certs          # put fullchain.pem + privkey.pem here
cd /srv/schmoose-live
curl -O https://raw.githubusercontent.com/DerDast3/schmoose-live/main/docker-compose.yml
curl -O https://raw.githubusercontent.com/DerDast3/schmoose-live/main/.env.sample
mv .env.sample .env                        # edit: PUBLIC_HOST, CERT_DIR, AUTH_MODE
docker login ghcr.io -u <your-github-user> # token with read:packages
docker compose up -d
```

Then open `https://<your-host>:9443` — you should see the room page.

### Layout

The runtime directory (e.g. `/srv/schmoose-live`) holds
`docker-compose.yml`, `.env` and `certs/`. The compose file is
self-contained; the image comes from ghcr.

## Connect to Schmoose

Point Schmoose at this SFU and share one secret:

| Where | Variable | Value |
|---|---|---|
| Schmoose `.env` | `SFU_URL` | `https://<your-sfu-host>:9443` |
| Schmoose `.env` | `SFU_API_KEY` | the shared secret |
| SFU `.env` | `SCHMOOSE_URL` | `https://<your-schmoose-host>:7443` |
| SFU `.env` | `API_KEY` | **the same secret** |
| SFU `.env` | `AUTH_MODE` | `ticket` |

In one breath: the browser asks the Schmoose backend (membership is
checked there), the backend asks this SFU for the chat's room
(`POST /api/rooms`), signs a short-lived ticket (Ed25519, ≤ 120 s, contains
name, initials, theme and the room) and hands it to the new tab via the
URL fragment — the fragment never reaches any server log. The SFU verifies
the ticket against Schmoose's public key and issues its own session token
(reconnects survive network hiccups without a new ticket).

Key detail: **the SFU deliberately holds no Schmoose-side secret.** The
public-key fetch is unauthenticated (public keys are public), and key
rotation on the Schmoose side needs no SFU change — new `kid` at the
public-key endpoint, the SFU picks it up automatically.

## Testing without Schmoose

Set `AUTH_MODE=standalone` and the room ID itself becomes the ticket:
press **Create a room**, get a code like `7QK-M2X-9TB`, share the URL (or
scan the QR code shown on the page) and everyone lands in the same room.
Handy for trying the stack — and for talking to yourself to check your
microphone before a real meeting.

## Firewall

Open **TCP 9443** and **UDP 40000–40100**. The container runs with host
networking (mediasoup requirement — see the compose file), so the traffic
passes through your normal host firewall: what you see in `ufw status`
is what applies. Restricting sources per network is possible via your
firewall; the ticket check is the application-level door.

## Troubleshooting

### Cameras work on localhost but not from outside

`getUserMedia` requires a secure context: as soon as the page is served
from anything other than `localhost`, the URL must be **https** with a
certificate your browser trusts. A self-signed cert works on desktop with
a warning but not on iOS.

### Video freezes after a few seconds

Classic bridge-network + UDP symptom. The compose file uses host
networking for a reason — keep it. Also check the UDP range is actually
open (`RTC_MIN_PORT`–`RTC_MAX_PORT`) and matches your firewall.

### Countdown timers look wrong on some devices

Clock skew between server and clients confuses any server-timestamp-based
countdown. Schmoose Live computes grace-period countdowns client-side
(starting when the message arrives) — if you see drift anyway, sync the
**server** clock (NTP), not the clients.

### Alpine host: build fails

mediasoup compiles a native C++ worker — supported on glibc (Debian),
which is exactly what this image is. Don't try to build it natively on an
Alpine host; run the container.
