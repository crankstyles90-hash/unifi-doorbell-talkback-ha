# UniFi Doorbell Talkback for Home Assistant

Live two-way audio (talk to the visitor) for **UniFi Protect doorbells**, directly from a **Home Assistant dashboard** – including the Home Assistant iOS app on a wall tablet. No UniFi Protect app needed.

- Latency: **< 1 second**
- Works with stock **go2rtc** (no custom build) and the **WebRTC Camera** card
- Tested on **UniFi G6 Entry** (firmware 5.5.45), Home Assistant 2026.11, iPadOS 18.7

> 🇩🇪 Deutsche Anleitung: [README.de.md](README.de.md)

---

## How it works

```
iPad / browser mic ──WebRTC──▶ go2rtc ──stdin (PCMA 8 kHz)──▶ ffmpeg ──Opus 24 kHz / RTP / UDP──▶ doorbell:7004
doorbell video+audio ◀──WebRTC── go2rtc ◀──RTSPS── UniFi Protect (port 7441)
```

UniFi Protect cameras with a speaker accept talkback audio as a plain RTP/UDP stream on **port 7004 of the camera itself**. The expected format is published by Protect in each camera's `talkbackSettings`. The Home Assistant UniFi Protect integration already uses this path for its media player (TTS) – this project uses the same path for a **live microphone** via go2rtc's `exec` backchannel.

## Requirements

| What | Notes |
|---|---|
| Home Assistant with **HTTPS** | Browsers/apps only allow the microphone in a secure context |
| **go2rtc** add-on (AlexxIT) | v1.9.15 tested, `host_network` |
| **WebRTC Camera** (AlexxIT) | Install via *Settings → Marketplace* (HA ≥ 2026.11) or HACS on older versions |
| UniFi Protect doorbell with speaker | RTSP enabled for one channel |

## Setup

### 1. Find your doorbell's talkback settings

Log in to your UniFi console in a browser, then open:

```
https://<CONSOLE-IP>/proxy/protect/api/cameras
```

Find your doorbell and note:

- `host` → camera IP
- `talkbackSettings` → e.g.
  ```json
  {"typeFmt":"opus","typeIn":"serverudp-rtp","bindPort":7004,"channels":1,"samplingRate":24000}
  ```
- `channels[].rtspAlias` of an RTSP-enabled channel (enable RTSP in Protect → device → Settings → Advanced)

Quick check that the speaker path works (optional): in HA, call `tts.speak` on the doorbell's `media_player.*_speaker` entity.

### 2. go2rtc stream with backchannel

Add to `/config/go2rtc.yaml` (go2rtc add-on) – see [`examples/go2rtc.yaml`](examples/go2rtc.yaml):

```yaml
streams:
  doorbell_talk:
    - rtspx://<CONSOLE-IP>:7441/<RTSP-ALIAS>#backchannel=0
    - ffmpeg:doorbell_talk#audio=opus
    - "exec:ffmpeg -hide_banner -loglevel error -fflags nobuffer -f alaw -ar 8000 -ac 1 -i - -af aresample=24000 -c:a libopus -b:a 48k -frame_duration 20 -ar 24000 -ac 1 -f rtp udp://<CAMERA-IP>:7004#backchannel=1#audio=alaw/8000"
```

Adjust codec/sample rate to your `talkbackSettings` (see [Other models](#other-models)). Restart the go2rtc add-on.

Test: open the go2rtc web UI **from the Home Assistant sidebar** (so it runs over your HTTPS address – the browser blocks the microphone on plain `http://<HA-IP>:1984`), then append `webrtc.html?src=doorbell_talk&media=video+audio+microphone` to its URL and speak. You'll hear yourself back through the doorbell's own microphone.

### 3. Helper + safety automation

- Create a toggle helper `input_boolean.doorbell_talk`
- Import [`examples/automation.yaml`](examples/automation.yaml) – it turns the mic off after 3 minutes and when your doorbell popup closes

### 4. Dashboard card

See [`examples/card.yaml`](examples/card.yaml). The camera card is shown **twice**: without microphone when the toggle is off, with microphone when it's on. The microphone is only opened while the toggle is on.

## iOS app: microphone does not work

In the Home Assistant iOS/iPadOS app you may see **no microphone prompt at all**, and no "Microphone" entry under *iOS Settings → Home Assistant*. Two fixes were needed:

1. **Use HTTPS in the app** – *App settings → Companion App → your server*: set **Internal URL and External URL to `https://your-domain`**. With `http://` internally, `navigator.mediaDevices` is undefined.
2. **Enable Siri** on the iPad (*Settings → Siri*). In our test, `getUserMedia()` hung without any prompt and without an error while Siri was off (Dictation was already on – that alone was not enough). After enabling Siri, the microphone worked immediately.

## ⚠️ Privacy

Microphone permission is granted **per domain**. Any open page that shows a card with `media: …microphone` starts sending immediately – e.g. a forgotten browser tab on your PC.

- Only render the microphone card behind a toggle (as in the example)
- Keep the auto-off automation
- Close other dashboards when testing

## Known issues

- After switching off, audio may continue for ~2 s until the WebRTC connection closes.

## Other models

| Model | Firmware | talkbackSettings | Works |
|---|---|---|---|
| G6 Entry | 5.5.45 | opus / serverudp-rtp / 24000 Hz / port 7004 | ✅ |

Not tested with this setup yet:

- **G4 Doorbell Pro / Doorbell Lite** – according to [go2rtc PR #2340](https://github.com/AlexxIT/go2rtc/pull/2340), Protect requests Opus at about 22 kHz there, and that PR opens a talkback session through Protect first. Whether these models also accept audio directly on port 7004 (like the G6 Entry) is unknown.

Always check your own `talkbackSettings`. **Please add your model via issue or PR!**

## Pitfalls

- `exec` sources can't be added via go2rtc's `PUT /api/streams` – put them in `go2rtc.yaml`.
- `#audio=alaw/8000` on the exec line is required (otherwise "can't find consumer").
- Don't use `-re` in the backchannel ffmpeg.
- Testing with an audio file via `POST /api/streams?dst=…` is misleading (choppy or cut off) – test with a real microphone.

## Credits

- [go2rtc](https://github.com/AlexxIT/go2rtc) and [WebRTC Camera](https://github.com/AlexxIT/WebRTC) by AlexxIT
- [uiprotect](https://github.com/uilibs/uiprotect) for documenting the talkback stream
- Related: [go2rtc PR #2340](https://github.com/AlexxIT/go2rtc/pull/2340) "Add UniFi Protect Opus talkback source" (native `unifi-talkback:` source, uses a Protect talkback session), [go2rtc issue #1760](https://github.com/AlexxIT/go2rtc/issues/1760) "talkback RTP to Unifi cameras"

## License

MIT
