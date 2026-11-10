# Stream: handoff (9 Nov 2026)

Attach this file and `stream.html` to the first message of the new chat, then paste the starter message at the bottom.

## What exists

- `stream.html`: single file, zero dependencies, about 36 KB. Live tab (camera, camera flip, three quality levels, level meter, wake-lock), Game mode on a Mac (iPhone Mirroring window plus Mac camera in a chosen corner, optional second audio input such as BlackHole), local crash-safe recording to IndexedDB, Mark button / label chips / M key, Sessions tab (rename marks, Review with Jump, Save video plus marks file through the share sheet, Interrupted recovery), Setup tab (WHIP URL, Bearer token, clip window), and Go live over WHIP to a relay.
- Tested in headless Chromium with fake devices: 46 app checks plus recovery, contrast and style scans, all passing.
- Not tested: a real iPhone or Safari, a real relay, any real platform, iPhone Mirroring capture, BlackHole drift.
- `creators-delight-build-sheet.html`: older static build sheet, print-ready on one page. Low priority.

## Design rules (strict)

- Palette: cocoa `#53393a`, rose `#c1798d`, mauve `#835965`. Rose is the only accent. Page ink `#1a1112`, raised `#241819`, text `#f4e9ea`, dim `#c4a7ae`.
- Type: serif wordmark, sans-serif body, monospace for timecodes and labels. No web fonts.
- Forbidden: gradients, glass or blur, blobs, scroll animation, emojis, fake stats, placeholder footers, rounded-2xl styling. Flat only. Radius 3px max.

## Platform facts (checked 9 Nov 2026)

- Instagram: has RTMP ingest. Creator or Business accounts get a per-stream server URL and key from Live Producer on instagram.com (desktop). The key changes every stream.
- TikTok: 1,000+ followers and age 18+ for a stream key, and not every eligible account gets one. The key changes every stream.
- Twitch: WebRTC ingest is an unadvertised beta. Treat as unverified and use standard RTMP through the relay.
- MediaMTX: recent releases renamed `runOnReady` to `runOnAvailable` and `runOnNotReady` to `runOnUnavailable`. MediaMTX cannot push RTMP itself, so ffmpeg does the fan-out. It rejects unknown config keys, so run it once in the foreground to validate.
- Browsers: iOS needs HTTPS for the camera. Safari cannot capture the phone's own screen or another app's audio. Locking the iPhone stops the camera and recording.

## Measured in Chromium

- MP4 MediaRecorder ignores the 2-second timeslice. The first chunk (about 5 s) is only the file header, and the first media arrives at about 10 s. A crash in the first 10 s loses everything.
- Calling `requestData()` on Chrome MP4 returned a 36-byte chunk and then nothing, and the result would not open. Do not use it.
- WebM emitted a chunk every 2 s.
- Safari behaviour is unknown. Use `probe.html` (below) on the iPhone.

## Corrections still needed in stream.html

1. The Setup tab relay recipe uses `runOnReady` and a one-shot fan-out script. Replace it with the relay design below.
2. The crash-recovery copy says "up to about 10 s". Say instead that Chrome MP4 delivers its first media at about 10 s.
3. Optional: prefer WebM on Chromium for record-only, create the AudioContext at 48 kHz with unprocessed game-audio constraints, and add a game-audio delay field in ms using a DelayNode.

## Relay design (untested; derived from docs)

Serve `stream.html` from the relay box with Caddy so the page and WHIP share one origin. WHIP URL is `https://stream.example.com/live/whip`; the Bearer token is `WHIP_TOKEN`. Open ports 22, 80, 443 (TCP) and 8189 (UDP and TCP). Never open 8889 or 8554. Each platform is its own systemd service, so rotating a key restarts only that ffmpeg and leaves the ingest and recordings running.

`/opt/mediamtx/mediamtx.yml`

```yaml
logLevel: info
api: no
rtmp: no
hls: no
srt: no
rtspAddress: 127.0.0.1:8554
webrtc: yes
webrtcAddress: 127.0.0.1:8889
webrtcEncryption: no
webrtcAllowOrigins: ['https://stream.example.com']
webrtcLocalUDPAddress: :8189
webrtcLocalTCPAddress: :8189
webrtcAdditionalHosts: ['stream.example.com']
webrtcICEServers2:
  - url: stun:stun.cloudflare.com:3478
authInternalUsers:
  - user: any
    pass:
    ips: ['127.0.0.1', '::1']
    permissions:
      - action: publish
        path: live
      - action: read
        path: live
paths:
  live:
    record: yes
    recordPath: /var/lib/stream/recordings/%path/%Y-%m-%d_%H-%M-%S-%f
    recordFormat: fmp4
    recordPartDuration: 1s
    recordSegmentDuration: 1h
    recordDeleteAfter: 7d
```

`Caddyfile` (set `WHIP_TOKEN` in Caddy's service environment)

```
stream.example.com {
	@whip path /live/*
	handle @whip {
		@authed header Authorization "Bearer {$WHIP_TOKEN}"
		handle @authed {
			reverse_proxy 127.0.0.1:8889
		}
		respond 401
	}
	handle {
		root * /var/www/stream
		file_server
	}
}
```

`/opt/stream/push.sh`

```bash
#!/bin/bash
exec ffmpeg -hide_banner -loglevel warning -rtsp_transport tcp \
  -i rtsp://127.0.0.1:8554/live -r 30 \
  -c:v libx264 -preset veryfast -tune zerolatency -b:v 3000k -maxrate 3000k -bufsize 6000k \
  -g 60 -keyint_min 60 -sc_threshold 0 -pix_fmt yuv420p \
  -c:a aac -b:a 128k -ar 48000 -f flv "${URL%/}/${KEY}"
```

`/etc/systemd/system/dest@.service`

```ini
[Unit]
Description=Push to %i
After=mediamtx.service
StartLimitIntervalSec=0
[Service]
User=stream
EnvironmentFile=/etc/stream/%i.env
ExecStart=/opt/stream/push.sh
Restart=always
RestartSec=2
```

`/usr/local/bin/dest` (create `/etc/stream` first, then `chmod +x`)

```bash
#!/bin/bash
case "$1" in
 set) read -rp "Server URL: " u; read -rsp "Stream key: " k; echo
      printf 'URL="%s"\nKEY="%s"\n' "$u" "$k" > /etc/stream/$2.env
      chown root:stream /etc/stream/$2.env; chmod 640 /etc/stream/$2.env
      systemctl enable --now dest@$2; systemctl restart dest@$2 ;;
 off) systemctl disable --now dest@$2; rm -f /etc/stream/$2.env ;;
esac
```

Per stream: start the stream in the app, copy the fresh URL and key from the platform's producer page on a computer, run `ssh -t relay sudo dest set instagram` (or `tiktok`, `twitch`), press Go Live on the platform once its preview appears, and afterwards `sudo dest off instagram`.

Caveats: Hetzner raised cloud prices twice in 2026, so check the live price. A 2 vCPU server should handle re-encoding two destinations at 720p30 (untested). The $6 DigitalOcean plan is 1 vCPU and only suits `-c:v copy`. A home machine needs UDP and TCP 8189 plus 80 and 443 forwarded, no CGNAT, and DDNS. MediaMTX issue #5293 reported Opus errors with mono audio on 1.15.5; check the logs if audio is missing.

## Safari recorder probe (`probe.html`, host it next to stream.html)

```html
<pre id=o></pre><button onclick="go()">Probe</button>
<script>async function go(){const s=await navigator.mediaDevices.getUserMedia({video:true,audio:true}),t=performance.now(),r=new MediaRecorder(s,{mimeType:MediaRecorder.isTypeSupported('video/mp4')?'video/mp4':''});r.ondataavailable=e=>o.textContent+=((performance.now()-t)/1000).toFixed(1)+'s '+e.data.size+' B\n';r.start(2000);setTimeout(()=>{r.stop();s.getTracks().forEach(x=>x.stop())},30000)}</script>
```

## BlackHole sync notes (unverified over long streams)

- Set BlackHole, the Multi-Output device and the mic to 48,000 Hz in Audio MIDI Setup. In the Multi-Output device make the physical output the clock source and tick Drift Correction on BlackHole.
- In JS: `new AudioContext({sampleRate:48000, latencyHint:'interactive'})`; capture game audio with `echoCancellation:false, noiseSuppression:false, autoGainControl:false, channelCount:{ideal:2}, sampleRate:{ideal:48000}, latency:{ideal:0}`; route it through `createDelay(1)` with a user-set offset in ms.
- Calibrate with a tap test: record 10 s of tapping on the game screen, measure the audio-to-video offset in a frame-stepping player, enter it, and repeat after 30 minutes to detect drift.

## Open decisions

- Where to host: relay box with Caddy (recommended) or Cloudflare Pages Direct Upload (then add that origin to `webrtcAllowOrigins` and let OPTIONS through unauthenticated).
- Relay on a VPS or at home, and which domain to use.
- Safari recorder cadence (run the probe).

## Starter message

Continue the Stream project from the attached handoff and `stream.html`. Keep the design rules exactly. First apply the three corrections listed under "Corrections still needed", then re-run browser checks like before (record, marks, crash recovery, save, WHIP handshake, game mode, overflow, contrast, style scan) and report what passes and what I still need to test on a real iPhone.
