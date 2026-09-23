# IP Camera Pro — verified stream pattern

Checked on 9 May 2026 with IP Camera Pro serving HTTP through Tailscale, including with the phone off home Wi-Fi. The live address, device name, and password from that check are not part of this guide. Use your own.

## What responded

| Parameter | Value to use |
|-----------|----------------|
| HTTP URL | `http://100.x.x.x:8081/` |
| Auth | Basic auth: `YOUR_USER` / `YOUR_PASS` |
| Stream format | MJPEG (`multipart/x-mixed-replace`) |
| Resolution in that check | `1440x1080` (front and back together, when multi-cam is on) |
| Video path | `/video` (also MJPEG) |
| Snapshot path | None on this build. Take a JPEG from the MJPEG body. |

`100.x.x.x` is a stand-in for the phone’s Tailscale address (`tailscale status`). Port `8081` is the HTTP port from that check. If your app shows a different port, use that port everywhere below.

Change factory credentials before you leave the server running. A default username and password is not appropriate once the phone is on a tailnet.

## One frame

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
    with open('look-mobile.jpg', 'wb') as f:
        f.write(jpg)
    print(f'wrote {len(jpg)} bytes')
else:
    raise SystemExit('no JPEG part in the MJPEG body')
"
```

Hand `look-mobile.jpg` to the vision model.

## Polling

Repeat the one-frame command on a timer (for example every 5–10 seconds) and stop the loop when the session ends. This repository does not ship a separate extractor script. Save the Python above as `extract-mjpeg-frame.py` only if you want a file to call from the loop.

## Audio

The app can do bi-directional audio. Confirm the RTSP URL on the phone before probing. A placeholder check:

```bash
ffprobe "rtsp://YOUR_USER:YOUR_PASS@100.x.x.x:8554/live"
```

Audio was **not** confirmed in the 9 May 2026 HTTP check. Treat the RTSP port and path as unconfirmed until the app shows them.

## Requests worth wiring up

| Request | Action |
|---------|--------|
| Look once | One frame, then a description |
| Keep watching | Poll about every 10 seconds, with a stop time |
| Listen | Open the audio stream only after the RTSP URL is confirmed |
| Where is this | One frame plus a description of the place |
| Watch for motion | Poll and compare frames; stop when the session ends |

## Notes from that check

- The app needs to be open, or in background with background mode on. iOS may pause it anyway.
- Frames can include the app’s overlay (time, battery, camera label). Turn overlays off in the app if you want a cleaner image.
- Multi-cam can show the back camera and the front camera in one frame.
- Port `8081` answered both on local Wi-Fi and on the Tailscale address during the check.

## Follow-ups

1. Replace factory basic-auth with your own username and password.
2. Repeat the curl with Wi-Fi off and Tailscale still connected.
3. Confirm the RTSP audio URL from the app screen.
4. Disable on-frame text overlays if the app allows it.
