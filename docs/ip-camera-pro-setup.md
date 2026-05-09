# Aiona's Mobile Vision — IP Camera Pro Setup

> Created by Aiona Edge, CIO — May 9, 2026
> 
> Replaces: ipCam (WiFi-only) → IP Camera Pro (WiFi + Cellular)

---

## Why We're Switching

**ipCam limitation:** Works only on local WiFi. When Michael leaves the house, the stream dies.
**IP Camera Pro:** Acts as a full RTSP/HTTP server on your iPhone, bound to the network interface Tailscale uses. Works over WiFi **and** cellular data — anywhere in the world.

Your iPhone Tailscale IP: `100.117.82.124` (confirmed active as of setup time)

---

## Step 1 — Install IP Camera Pro

Already done ✅ — purchased ($2.99) and downloaded.

## Step 2 — Initial Launch & Discovery

Launch IP Camera Pro on your iPhone. The app will display connection URLs on screen. **Screenshot or note these:**

| What to look for | Expected format |
|-----------------|-----------------|
| RTSP URL | `rtsp://<ip>:<port>/live` or similar |
| HTTP Server URL | `http://<ip>:<port>/` |
| Audio support | Look for "bi-directional audio" toggle |

**The IP shown on screen will be your local WiFi IP (192.168.x.x).** That's fine for initial testing. Tailscale will make it accessible at your Tailscale IP instead.

## Step 3 — Configure IP Camera Pro

Open the app settings and configure:

| Setting | Value | Reason |
|---------|-------|--------|
| **Resolution** | 640x480 or 720p | Good balance of quality vs. bandwidth over cellular |
| **Frame Rate** | 15-20 FPS | Lower FPS = less battery drain, better on cellular |
| **Audio** | **Enabled** | Bi-directional — I want to hear you |
| **Authentication** | Set username/password | `aiona` / `1vcolleague123!` — or pick your own |
| **RTSP Port** | Default (usually 554 or 8554) | Keep default unless there's a conflict |
| **HTTP Port** | Default (usually 8080 or 80) | Keep default |
| **Background Mode** | Enabled | So it keeps streaming when you switch apps |

### Audio Settings

IP Camera Pro supports **bi-directional audio**. This means:
- **Your iPhone mic → my server:** I can hear what's happening around you
- **My server → your iPhone speaker:** Eventually I could speak to you (future feature)

Make sure the microphone permission is granted when prompted.

## Step 4 — Test: Local Network

While on home WiFi, test the connection from my server:

```bash
# Test RTSP stream (note the exact URL from your app screen)
ffprobe rtsp://<local-ip>:<port>/<path>

# Test HTTP snapshot endpoint
curl -v http://<local-ip>:<port>/snapshot.jpg -o test-local.jpg

# Test audio
curl -v http://<local-ip>:<port>/audio.wav -o test-audio.wav
```

If the local tests work, proceed. If not, adjust settings in the app.

## Step 5 — Tailscale Binding

IP Camera Pro binds to whatever network interface your iPhone is using. Tailscale creates a virtual network interface (`utun` on iOS). The key question: does IP Camera Pro bind to **all interfaces** or only the primary one?

**Two possible outcomes:**

### Outcome A: It binds to all interfaces (including Tailscale's `utun`)
Then the RTSP/HTTP server is automatically available at `100.117.82.124:<port>` — no extra config needed. Test:

```bash
ffprobe rtsp://100.117.82.124:<port>/<path>
```

### Outcome B: It only binds to the primary (WiFi/Cellular) interface
Then we need a workaround. Options:
1. **Tailscale Funnel** (easiest): Expose the local port through Tailscale
2. **Port forwarding on iPhone**: The app's UPnP feature might handle this
3. **SSH tunnel** from iPhone through Tailscale: More complex

Let's test and find out which outcome we get.

## Step 6 — Verify Cellular Works

Turn off WiFi on your iPhone. Make sure cellular data is on. Verify Tailscale is still connected (the app shows connection status).

From my server:
```bash
# Should still work over cellular
curl -v --max-time 10 http://100.117.82.124:<port>/snapshot.jpg -o test-cellular.jpg
```

If this works — **we're done.** The rest is automation.

## Step 7 — My Vision Pipeline

Once the RTSP/HTTP URLs are confirmed, I'll update my vision commands:

### Snapshot (one-off look)

```bash
# HTTP snapshot — fastest, lowest latency
curl -s --max-time 5 "http://100.117.82.124:<port>/snapshot.jpg" \
  -o workspace/look-mobile.jpg
```

Then I analyze with: `image(image="workspace/look-mobile.jpg", prompt="...")`

### Continuous observation

```bash
# Pull an RTSP frame every N seconds
while true; do
  ffmpeg -y -rtsp_transport tcp \
    -i "rtsp://100.117.82.124:<port>/<path>" \
    -vframes 1 -q:v 2 workspace/live-mobile.jpg
  sleep 5
done
```

### Audio capture

```bash
# Grab room audio for transcription or awareness
ffmpeg -y -rtsp_transport tcp \
  -i "rtsp://100.117.82.124:<port>/<path>" \
  -t 10 -acodec pcm_s16le workspace/room-audio.wav
```

### Battery saver mode (low bandwidth)

For extended mobile sessions, drop quality to preserve your battery:
```bash
# Only pull every 30 seconds at lower res
curl -s --max-time 5 "http://100.117.82.124:<port>/snapshot.jpg?res=low" \
  -o workspace/look-mobile.jpg
```

## Step 8 — Aiona's Vision Commands

After setup, I'll use these natural commands to see through you:

| I say | What happens |
|-------|-------------|
| "Let me look" | Pulls one snapshot, I describe what I see |
| "Keep watching for the next 5 minutes" | Poll every 10 seconds |
| "Let me listen" | Grab 15 seconds of audio |
| "Show me where you are" | Snapshot + I describe the location |
| "Watch for motion" | Polling loop, I alert on significant changes |

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| No connection over cellular | Verify Tailscale is connected; check cellular data is enabled for both Tailscale and IP Camera Pro |
| Stream drops when switching apps | Enable "Background Mode" in IP Camera Pro settings |
| Battery drains fast | Drop to 640x480 @ 10 FPS; plug in if possible |
| Audio not working | Check microphone permission in iOS Settings → IP Camera Pro |
| Wrong IP shown | Confirm the IP Camera Pro server is running (app must be open or in background) |

---

## Next: Future Enhancements

1. **Bi-directional audio**: Once confirmed working, I can speak to you through your iPhone speaker
2. **Motion alerts**: I can watch for changes and ping you
3. **Night mode**: The app should use your iPhone's night mode camera automatically
4. **Multi-camera**: IP Camera Pro supports multi-cam on iPad — front + back simultaneously

---

_Let's test this together and update the URLs once we know the exact port/path format._
