# Aiona's Mobile Vision — CONFIRMED WORKING ✅

> Tested: May 9, 2026 at 3:37 PM ET
> iPhone Tailscale IP: `100.117.82.124`

---

## Connection Details

| Parameter | Value |
|-----------|-------|
| **HTTP URL** | `http://100.117.82.124:8081/` |
| **Auth** | admin / admin |
| **Stream format** | MJPEG (multipart/x-mixed-replace) |
| **Resolution** | 1440x1080 (dual camera: back telephoto + front) |
| **Video endpoint** | `/video` (also MJPEG) |
| **Snapshot endpoint** | None — extract frame from MJPEG stream |

---

## Aiona's Vision Pipeline (Working)

### Quick snapshot (one look)

```bash
# Pull one frame from MJPEG stream
curl -s -u "admin:admin" --max-time 5 "http://100.117.82.124:8081/" | \
  python3 -c "
import sys, re
data = sys.stdin.buffer.read()
match = re.search(rb'Content-Length:\s*\d+\r?\n\r?\n', data)
if match:
    jpg = data[match.end():]
    boundary = jpg.find(b'\r\n--')
    jpg = jpg[:boundary] if boundary > 0 else jpg
    with open('${WORKSPACE}/look-mobile.jpg', 'wb') as f: f.write(jpg)
"
```

Then analyze: `image(image="look-mobile.jpg", prompt="Describe what you see...")`

### Continuous observation (every 5 seconds)

```bash
while true; do
  curl -s -u "admin:admin" --max-time 4 "http://100.117.82.124:8081/" | \
    python3 ~/extract-mjpeg-frame.py workspace/live-mobile.jpg
  sleep 5
done
```

### Audio capture

The IP Camera Pro app supports bi-directional audio. Test the audio stream:

```bash
# Check audio endpoints on the iPhone (RTSP audio stream if available)
ffprobe rtsp://admin:admin@100.117.82.124:8554/live
```

Audio confirmed via: _pending test_

---

## Aiona's Vision Commands

| I say | What happens |
|-------|-------------|
| "Let me look" | Pulls one snapshot, I describe what I see |
| "Keep watching" | Poll every 10 seconds |
| "Let me listen" | Grab audio stream |
| "Show me where you are" | Snapshot + location description |
| "Watch for motion" | Polling loop, alert on changes |

---

## Notes

- **App must be open or recently active** — iOS background restrictions may pause stream
- **Battery overlay**: Stream includes timestamp, battery, camera info as text overlay
- **Dual camera**: Currently showing back telephoto + front camera simultaneously
- **Auth**: Consider changing from admin/admin in app settings for security
- **Port 8081**: Confirmed on both local WiFi and Tailscale

---

## Next Steps

1. **Change default password** — admin/admin is not secure for persistent use
2. **Test cellular only** — Turn off WiFi, verify Tailscale + stream still works
3. **Test audio** — Confirm RTSP audio stream endpoint and bi-directional capability
4. **Extract clean frames** — Remove text overlays (timestamp/battery) if possible in app settings
5. **Set up aiona@smfworks.com oauth** — So I can send emails from my own address
