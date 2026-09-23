# IP Camera Pro setup notes

Community setup notes for the iPhone camera path in [SMF OpenClaw Vision](../README.md).

IP Camera Pro runs an RTSP/HTTP server on the phone. Tailscale gives that phone a stable mesh address, so the same endpoint works on Wi-Fi and on cellular. The earlier ipCam app was only useful on the local network.

These notes use placeholders. Substitute your own values:

| Placeholder | Meaning |
|-------------|---------|
| `100.x.x.x` | The iPhone’s address from `tailscale status` |
| `YOUR_USER` / `YOUR_PASS` | Basic auth you set in the app |
| `<port>` and `<path>` | Whatever the app prints on its connection screen |

If the app offers a factory username and password, change them before the server is reachable from other tailnet devices. Do not publish real credentials.

## 1. Install IP Camera Pro

Install **IP Camera Pro** (publisher 沈垚 / ShenYao) from the App Store. The purchase used for this guide was $2.99, one time.

## 2. Read the URLs on the phone

Launch the app and note the URLs it shows.

| What to look for | Typical shape |
|------------------|---------------|
| RTSP URL | `rtsp://<ip>:<port>/<path>` |
| HTTP URL | `http://<ip>:<port>/` |
| Audio | A bi-directional audio toggle, if you want the microphone |

The IP on that screen is usually the LAN address (`192.168.x.x`). That is fine for a first test on the same Wi-Fi. The agent should use the Tailscale address, not the LAN address, once you leave that network.

## 3. Configure the app

| Setting | Suggestion | Why |
|---------|------------|-----|
| Resolution | `640x480` or `720p` to start | Enough detail without a large cellular upload |
| Frame rate | 15–20 FPS, or 10 FPS on battery | Lower rates cost less battery and data |
| Audio | Off until you need it | The microphone is part of the stream when this is on |
| Authentication | `YOUR_USER` / `YOUR_PASS` | Replace any factory default first |
| RTSP port | App default (often `554` or `8554`) | Change only if something else already uses it |
| HTTP port | App default, or `8081` if you set it | The verified HTTP pattern later in these docs used `8081` |
| Background mode | On | Keeps the server up when you switch apps, within iOS limits |

Grant the microphone permission only if audio is enabled.

## 4. Test on the local network

From a computer on the same Wi-Fi, using the LAN address shown in the app:

```bash
ffprobe "rtsp://YOUR_USER:YOUR_PASS@<local-ip>:<port>/<path>"

curl -v -u "YOUR_USER:YOUR_PASS" \
  "http://<local-ip>:<port>/" -o test-local.bin

curl -v -u "YOUR_USER:YOUR_PASS" \
  "http://<local-ip>:<port>/snapshot.jpg" -o test-local.jpg
```

A missing snapshot URL is normal for the build checked in the [verified stream notes](./ip-camera-pro-mobile-working.md). If `snapshot.jpg` 404s, extract a frame from the MJPEG stream instead (see the README).

## 5. Reach it through Tailscale

On the agent host:

```bash
tailscale status
```

Find the phone’s `100.` address. The app binds to interfaces on the phone. Two outcomes showed up while this was first tested:

**The server is reachable on the Tailscale address.** No extra tunnel:

```bash
curl -v -u "YOUR_USER:YOUR_PASS" --max-time 10 \
  "http://100.x.x.x:<port>/" -o test-tailscale.bin
```

**The server answers only on Wi-Fi or cellular, not on the Tailscale interface.** Tailscale is still connected, but the camera process is not listening on it. Options that stay on the tailnet:

1. Check the app for a “listen on all interfaces” (or similar) setting and turn it on.
2. Use Tailscale’s own sharing tools only if you understand who else on the tailnet can open the port. Prefer an ACL that limits which devices can reach the phone.

Do not put the camera on the public internet to work around a bind issue.

## 6. Test on cellular

Turn Wi-Fi off. Leave cellular data on. Confirm the Tailscale app still shows connected, and that iOS allows cellular data for Tailscale and for IP Camera Pro.

```bash
curl -v -u "YOUR_USER:YOUR_PASS" --max-time 10 \
  "http://100.x.x.x:<port>/" -o test-cellular.bin
```

If that returns the stream, the mesh path works away from home Wi-Fi.

## 7. Commands the agent can run

### One frame from MJPEG

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
    open('look-mobile.jpg', 'wb').write(jpg)
    print('wrote', len(jpg), 'bytes')
"
```

Pass `look-mobile.jpg` to the vision model. Port `8081` matches the [verified notes](./ip-camera-pro-mobile-working.md). Use the port from your app if it differs.

### A still URL, when the app has one

```bash
curl -s -u "YOUR_USER:YOUR_PASS" --max-time 5 \
  "http://100.x.x.x:<port>/snapshot.jpg" -o look-mobile.jpg
```

### An RTSP frame every few seconds

```bash
while true; do
  ffmpeg -y -rtsp_transport tcp \
    -i "rtsp://YOUR_USER:YOUR_PASS@100.x.x.x:<port>/<path>" \
    -vframes 1 -q:v 2 live-mobile.jpg
  sleep 5
done
```

### A short audio clip

Only if the app’s RTSP URL includes audio and you have turned audio on:

```bash
ffmpeg -y -rtsp_transport tcp \
  -i "rtsp://YOUR_USER:YOUR_PASS@100.x.x.x:<port>/<path>" \
  -t 10 -acodec pcm_s16le room-audio.wav
```

### Lower bandwidth

Drop resolution in the app, lengthen the poll interval, or both. A query string such as `?res=low` only works if that build documents it. Do not assume it.

## Example requests an agent might map to tools

| Request | Action |
|---------|--------|
| Look once | Pull one frame and describe it |
| Watch for a few minutes | Poll every 10 seconds, then stop |
| Listen | Record a short clip only when audio is enabled |
| Where is the camera pointed | One frame plus a location description from the image |

## Troubleshooting

| Issue | What to try |
|-------|-------------|
| No route on cellular | Tailscale connected, cellular data allowed for Tailscale and the camera app |
| Stream stops in the background | Background mode in the app; iOS may still suspend it |
| Battery use is high | `640x480` at about 10 FPS, or power the phone |
| No audio | Microphone permission and the in-app audio toggle |
| LAN URL works, mesh URL does not | Re-check which interface the app bound, and the port |

## Later improvements

These are optional follow-ups, not extra products:

1. Confirm bi-directional audio on the RTSP URL the app prints.
2. Poll for motion only while a session is active.
3. Use the phone’s existing low-light camera behavior through the app.
4. Try the app’s multi-camera mode if you want front and back together.
