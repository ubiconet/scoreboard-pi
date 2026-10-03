# Scoreboard Pi Streaming Stack

Raspberry Pi side of the baseball scoreboard system: captures camera + audio, renders the live GameChanger overlay, and pushes video+audio to Twitch/YouTube RTMP.

## Architecture (4-stage ffmpeg pipeline)

```
video ffmpeg ──► mpegts FIFO ─┐
                              ├─► merge ──► TS FIFO ──► pusher ──► RTMP
arecord ──► aac ffmpeg ──AAC──┘
```

Key design points (learned the hard way — see comments in code):
- **Video FIFO is mpegts, not raw h264**: raw H.264 has NOPTS timestamps; ffmpeg's interleaver then starves the audio input forever (the multi-day "audio freezes" bug)
- **Audio via `arecord` pipe**, not ffmpeg's `-f alsa` demuxer (stuck-state bug); `plughw:2,0` (camera mic) — the standalone USB PnP mic is dead hardware
- **Pusher is `-c copy`** with 1s probe budget; wchar is NOT a liveness metric for RTMP (librtmp buffers internally)

## Files

- `stream_scoreboard.py` — main streamer: camera capture, overlay compositor, control API on :8080 (`/stream/start`, `/stream/stop`, `/rtmp`, `/status`)
- `command_listener.py` — connects to the scoreboard backend to receive overlay data + start/stop commands (deploys as a flat root file on the Pi)
- `scoreboard-stream.service` — systemd unit
- `pin_diagnostic.py` — hardware diagnostics

## Deploy

```bash
scp stream_scoreboard.py command_listener.py skuzmak@<pi>:/home/skuzmak/projects/scoreboard/
```

Launch on Pi:
```bash
python3 stream_scoreboard.py --identifier TEST1 --api-port 8080 --camera /dev/video0 \
  --audio-device plughw:2,0 --audio-bitrate 64k --audio-sample-rate 44100 --audio-channels 1 \
  --width 640 --height 480 --fps 30 --url https://scoreboard.ubiconet.com
```

The web app lives in a sibling repo: [`baseball-scoreboard`](https://github.com/ubiconet/baseball-scoreboard).
