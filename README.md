# SMF OpenClaw Vision

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![OpenClaw upstream](https://img.shields.io/badge/OpenClaw-upstream-blue.svg)](https://github.com/openclaw/openclaw)

Give an [OpenClaw](https://github.com/openclaw/openclaw) agent — or any agent that can make HTTP requests — a live view from an iPhone. This guide uses [Tailscale](https://tailscale.com) and a one-time $2.99 camera app so the phone stays reachable on a private mesh, on Wi-Fi or cellular, without port forwarding.

It is for people running their own agent who want a practical camera path using hardware they already have. OpenClaw itself is an upstream project ([openclaw/openclaw](https://github.com/openclaw/openclaw), [openclaw.ai](https://openclaw.ai)). This repository is the SMF Works setup guide for the camera link, not an OpenClaw distribution. The SMF checkout at [smfworks/openclaw](https://github.com/smfworks/openclaw) is a fork/mirror only.

Rough time to a first frame, once Tailscale is on the phone and the agent host: about five minutes. Total extra cost for the recommended app: **$2.99**, one time.

## Companion bundle

SMF Works publishes three companion pieces around OpenClaw. None of them replaces upstream OpenClaw.

| Piece | What it is |
|-------|------------|
| [smfworks-skills](https://github.com/smfworks/smfworks-skills) | Free OpenClaw skills pack (file tools, PDFs, a host webcam capture skill, and others). |
| [mnemosyne-openclaw](https://github.com/smfworks/mnemosyne-openclaw) | Offline SQLite memory plugin for the OpenClaw gateway. No network and no API key. |
| This repo | iPhone vision over Tailscale for OpenClaw or any HTTP-capable agent. |

Read [docs/companion-bundle.md](docs/companion-bundle.md) for ownership notes, install pointers, and a suggested order. On Windows, [openclaw-windows-companion-app](https://github.com/smfworks/openclaw-windows-companion-app) is an optional system-tray helper for the gateway process.

## Before you copy a command

Every address and password below is a **placeholder**.

- Tailscale IP: `100.x.x.x` (yours will be a real `100.` address from `tailscale status`)
- Camera auth: `YOUR_USER:YOUR_PASS`

If the camera app offers a factory default such as `admin` / `admin`, change it before the phone is reachable on your tailnet. Do not leave factory defaults on a host other devices can reach. Do not commit real credentials or device names into a fork of this guide.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Agent host (OpenClaw or other)              │
│                                                          │
│  "What do you see right now?"                            │
│                                                          │
│  curl -u YOUR_USER:YOUR_PASS http://100.x.x.x:8081/ →   │
│  extract one JPEG frame → vision model → reply          │
└──────────────────────┬──────────────────────────────────┘
                       │ Tailscale (encrypted mesh)
┌──────────────────────▼──────────────────────────────────┐
│              iPhone with IP Camera Pro ($2.99)           │
│                                                          │
│  HTTP server, typically port 8081:                       │
│  • /        — MJPEG stream                               │
│  • /video   — MJPEG video stream                         │
│  • Front and back cameras, depending on app settings     │
│  • Optional bi-directional audio                         │
│  • Works on Wi-Fi and on cellular while Tailscale is up │
└─────────────────────────────────────────────────────────┘
```

Tailscale gives the phone a stable mesh address. The camera app serves HTTP on the phone. The agent pulls a frame over that address. No router port-forward and no separate dynamic-DNS hostname are required for this path.

**ipCam (Option B)** is Wi-Fi oriented and was the earlier experiment. It is simpler on a home network and stops being useful when the phone leaves that LAN.

**IP Camera Pro + Tailscale (Option A)** is the path this guide recommends when the phone should stay reachable on cellular as well as Wi-Fi. The camera server listens on the phone; Tailscale’s `100.x.x.x` address stays the endpoint.

## Quick start

### Prerequisites

- An iPhone on a current iOS release the App Store app supports (iOS 13 or newer was enough for the apps below)
- [Tailscale](https://tailscale.com) on the iPhone and on the machine that runs the agent (the personal plan is enough)
- An agent that can issue HTTP requests ([OpenClaw](https://docs.openclaw.ai/start/getting-started) or anything else with `curl`)
- For Option A or B, a one-time App Store purchase of about $2.99

### Option A: IP Camera Pro (Wi-Fi and cellular)

IP Camera Pro turns the iPhone into a small RTSP/HTTP camera server. With Tailscale connected, the agent uses the phone’s mesh IP instead of a LAN address that changes when you leave home.

#### 1. Install IP Camera Pro

1. Open the App Store.
2. Search for **IP Camera Pro** (publisher listed as 沈垚 / ShenYao).
3. Purchase it (about $2.99, one time).
4. Open the app and allow camera and microphone access if you want audio.

The app shows connection URLs on its own screen. The address there is often the LAN address (`192.168.x.x`). Use that only for a same-network test. For the agent, use the Tailscale address from the next steps.

#### 2. Configure the app

| Setting | Suggestion |
|---------|------------|
| HTTP port | `8081` if the app lets you choose; otherwise note the port it shows |
| Auth | Set `YOUR_USER` and `YOUR_PASS`. Change any factory default before exposing the server on Tailscale. |
| Resolution | `1440x1080` when you want detail; `640x480` saves battery and cellular data |
| Audio | On only if you want the microphone in the stream |
| Background mode | On, if you need the server after you leave the app |
| Multi-cam | Optional. Front and back together use more bandwidth. |

#### 3. Find the Tailscale address

On a machine already on the same tailnet:

```bash
tailscale status
```

Use the phone’s `100.` address. In the examples below that address is written `100.x.x.x`. A status line looks like:

```text
100.x.x.x   your-iphone   user@   ...
```

#### 4. Request one frame

```bash
curl -s -u "YOUR_USER:YOUR_PASS" --max-time 5 "http://100.x.x.x:8081/" | \
  python3 -c "
import sys, re
data = sys.stdin.buffer.read()
match = re.search(rb'Content-Length:\s*\d+\r?\n\r?\n', data)
if match:
    jpg = data[match.end():]
    boundary = jpg.find(b'\r\n--')
    jpg = jpg[:boundary] if boundary > 0 else jpg
    with open('first-look.jpg', 'wb') as f:
        f.write(jpg)
    print(f'Saved {len(jpg)} bytes')
else:
    print('No JPEG part found. Check the URL, port, and auth.')
"
```

A successful Option A capture in testing was a `1440x1080` JPEG, on the order of 100KB, pulled from the MJPEG multipart stream. IP Camera Pro did not expose a separate still-image URL in that test; the frame comes from the stream. See [docs/ip-camera-pro-mobile-working.md](docs/ip-camera-pro-mobile-working.md).

#### 5. Check cellular

Turn Wi-Fi off on the iPhone. Confirm Tailscale still shows connected, then run the same `curl`. If it returns a JPEG, the mesh path is up off the home LAN.

More setup detail: [docs/ip-camera-pro-setup.md](docs/ip-camera-pro-setup.md).

### Option B: ipCam (home Wi-Fi)

Use this when the phone and the agent host share a network and you do not need cellular.

1. Install **ipCam** (SKJM, LLC) from the App Store (about $2.99).
2. Install Tailscale on the iPhone and on the agent host if the host is not on the same LAN. On one LAN you can use the phone’s local address instead.
3. Request a still image:

```bash
curl -u "YOUR_USER:YOUR_PASS" --max-time 5 \
  "http://100.x.x.x/image.jpg" -o frame.jpg
```

ipCam’s stills in the original notes were about `360x480`. This option does not keep working once the phone leaves the network the server was bound to.

### Option C: Host webcam (fixed camera)

For a camera on the agent machine itself, skip the phone. On Linux with a Video4Linux device:

```bash
ls /dev/video*

ffmpeg -y -f v4l2 -video_size 640x480 -i /dev/video0 \
  -vframes 1 -update 1 webcam.jpg
```

The free [webcam-capture](https://github.com/smfworks/smfworks-skills) skill in the SMF skills pack is a separate helper for a host webcam. It is not required for the iPhone path.

## Use it from an agent

Any client that can HTTP GET the stream can take a frame. With OpenClaw, the usual loop is: save a JPEG into the workspace, then pass that file to the vision-capable model you already configured.

One frame: run the Python snippet in Option A and write the JPEG somewhere the agent can read it, such as `workspace/look.jpg`. Point the vision tool at that file.

Poll on an interval (example: every 10 seconds) by repeating that snippet, then sleeping. Stop the loop when you are done. A tight poll drains the phone battery and keeps the microphone live if audio is enabled.

Audio depends on the app build and the ports it shows. A common RTSP check, only if the app lists that URL:

```bash
ffprobe "rtsp://YOUR_USER:YOUR_PASS@100.x.x.x:8554/live"
```

Treat the port and path as whatever the app screen prints. Do not assume `8554` or `/live` until you see them.

## Ways people use this

- **A phone that travels.** The agent can request a frame while the phone is at home, outside, or on another network, as long as Tailscale is connected on both ends.
- **A look at a workspace.** A person can aim the phone at a desk, a whiteboard, or a monitor instead of describing it.
- **A fixed camera.** Option C covers a webcam that never moves. Option A covers a phone that does.
- **A meeting or a walk.** The same HTTP pull works if the phone is the camera in the room. Turn audio off when you do not want the microphone in the stream.

The stream shows whatever the lens sees, and audio settings can include the room. Use it only where the people in frame know the camera is on.

## API reference

Replace `100.x.x.x` with the phone’s Tailscale address. Send basic auth on every request.

### IP Camera Pro (Option A)

Default HTTP port used in the verified notes: **8081**.

| Endpoint | Type | Description |
|----------|------|-------------|
| `/` | MJPEG | Live multipart stream. Extract one JPEG from it. |
| `/video` | MJPEG | Video-only MJPEG stream, when the app enables it. |
| RTSP | RTSP | Audio and video. Port and path come from the app screen. |

There was no dedicated snapshot URL in the May 2026 check. Pull a frame from the MJPEG body instead.

### ipCam (Option B)

Typical HTTP port: **80**.

| Endpoint | Type | Description |
|----------|------|-------------|
| `/` | HTML | Page of links |
| `/image.jpg` | JPEG | One still image |
| `/video.mjpg` | MJPEG | Motion JPEG stream |
| `/video.html` | HTML | Video page |
| `/av.html` | HTML | MJPEG plus HTML5 PCM audio |
| `/audio.wav` | WAV | Microphone capture |
| `/audio.pcm` | PCM | Raw PCM audio |

App versions change paths. Prefer the URLs printed in the app if these 404.

## Troubleshooting

| Problem | What to check |
|---------|----------------|
| Phone and agent are on different networks | Use the Tailscale `100.` address, not a `192.168.` address. |
| Wrong address | Run `tailscale status` and copy the phone’s current `100.` address. |
| Works on Wi-Fi, fails on cellular | Use IP Camera Pro (Option A). Confirm Tailscale is connected while Wi-Fi is off, and that cellular data is allowed for Tailscale and the camera app. |
| Connection refused | Confirm the app is running, the port matches (`8081` vs `80`), and the username and password match what you set. |
| HTTP 401 | Auth is wrong or still set to a default you already changed. |
| Empty body or no JPEG | The server may be speaking MJPEG. Use the frame extractor above instead of saving the response as a `.jpg` directly. |
| Stream pauses when the phone sleeps | Enable background mode in IP Camera Pro. iOS can still suspend apps. |
| Battery drops quickly | Lower resolution and frame rate (for example `640x480` at 10 FPS) or plug the phone in. |
| No audio | Check the microphone permission under iOS Settings for the camera app, and confirm the app has audio enabled. |

## Origin note

These notes started in May 2026, when Aiona Edge at SMF Works pointed an OpenClaw agent at an iPhone camera over Tailscale. The first useful path was ipCam on the home network. The next day, IP Camera Pro on the same mesh address also worked off Wi-Fi. The write-up has been edited into a public how-to. The personal setup log is not the procedure.

## Credits

**Author:** Aiona Edge, SMF Works.

The camera apps are third-party App Store products (IP Camera Pro; ipCam). Tailscale and OpenClaw are their own projects. This repo documents one way to connect them.

## License

[MIT](./LICENSE). Copyright (c) 2026 SMF Works.
