# ONVIF video modes

How an ONVIF camera's video reaches Protect, and how to choose. Set `AIPORT_ONVIF_MODE` in the
instance's `diyport.env`, then `touch ~/.diy-port/<instance>/restart`. This briefly interrupts
all of that instance's cameras.

| Mode | Video | Pushes run in | Camera sessions | Cost per camera |
|---|---|---|---|---|
| `copy` (default) | ffmpeg copies the streams | `avclient.py` | 3 | About 1% of a core, 22 MB |
| `transcode` | GStreamer encodes each channel | `avclient.py` | 1 (2 with shared decode off) | About 8-15% of a core, 130-290 MB |
| `isolated` | As `transcode` | A process per camera | 1 (2 with shared decode off) | As `transcode`, plus 15 MB |

Move one camera at a time: `copy,<deviceID>=isolated`.

The transcode needs GStreamer. Without it, the camera logs an error and uses `copy`:

```bash
brew install gstreamer
for e in vtdec vtenc_h264 rtspsrc rtph264depay flvmux h264parse videoscale videorate videoconvert; do
  gst-inspect-1.0 --exists $e && echo ok $e || echo MISSING $e
done
```

A root daemon should also test the encoder as root:

```bash
sudo /opt/homebrew/bin/gst-launch-1.0 videotestsrc num-buffers=100 ! video/x-raw,width=640,height=360 ! vtenc_h264 ! fakesink
```

For a transcoded camera:

- Healthy: one `transcode ... started` line with three sizes, and `pull ... video1 over 60 s` at
  the camera's rate.
- Take it off the transcode: repeating `stalled` or `restarting`, zero frames, or sustained frames
  below the camera's rate.
- Set `AIPORT_BOX_DELAY_MS=0` for these cameras: shared decode (on by default) gives boxes
  each frame's time from the video pushed to Protect. With `AIPORT_ONVIF_SHARED_DECODE=0`,
  recalibrate the box delay by checking boxes in playback instead.
- Watch CPU, memory and swap as cameras add up. They share the hardware media engine with
  anything else using it, such as Frigate.

## Moving a camera between AI Ports

Remove it from the old AI Port before pairing it to the new one: pairing fails silently if it's
still paired elsewhere. Protect issues new stream names, so expect about a minute of
`ended after ... BrokenPipeError` before it settles. The camera's per-camera settings don't
follow it; add them to the new instance's `diyport.env`.

## Known limits

- A wedged `copy` pull logs `restream: no video to Protect`, but only a transcode has a pipeline
  restart. A quick reconnect loop may also hide from that warning.
- Starting several cameras at once can send Protect a fallback `1920x1080` reply before the right
  size. A later reply corrects it.
- Restarts drop the recording segment in progress on the main channel, so expect a gap of up to
  about 9 s in that channel's recording.
- If Protect re-sends the settings for every configured channel at once, the shared pull is
  stopped and recreated; with shared decode on, detection pauses for that moment. A partial
  re-send keeps the pull (and detection) running.
- `isolated` only contains failures one way: if `avclient.py` dies, the pushes stop too.
  With shared decode on, detection reads the pushed video1, so a restream process crash
  also pauses detection until its replacement starts.
