# Stream

A single-file stream control room (`stream.html`, no dependencies, no build step).

- **Live**: camera or Game + camera (a Mac window such as iPhone Mirroring with the camera in a corner), quality presets, level meter, Mark / label chips / `M` key, Record only or Go live over WHIP.
- **Sessions**: every recording is stored in the browser (IndexedDB) as it goes, so a crash keeps what was written. Rename, review, jump to marks, save the video plus a marks file.
- **Setup**: WHIP URL and token, clip window, the relay recipe (MediaMTX + Caddy + ffmpeg) and the Game-mode steps.

`probe.html` measures when MediaRecorder delivers its chunks on a device (useful on iPhone Safari).
`HANDOFF.md` is the design brief and platform notes this was built from.

## Run it

The camera needs HTTPS or localhost:

```bash
cd ~/Documents/stream && python3 -m http.server 8898
```

Then open <http://localhost:8898/stream.html>. On an iPhone, host it on an HTTPS site (ideally the relay itself, see Setup).

## Where it came from

Copied from `~/Downloads/stream.html` (the newest version, 9 Nov). The originals in Downloads are untouched.
