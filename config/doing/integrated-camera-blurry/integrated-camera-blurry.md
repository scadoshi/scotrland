# Integrated Camera — Blurry Image

## Status
In progress — WirePlumber approach exhausted; next: v4l2loopback bridge

## Symptoms
- Camera appears blurry in all apps (Kamoso, Teams, browser)
- Affects all apps equally, not app-specific

## What We Know
- Camera: Integrated Camera (USB ID 30c9:005f)
- Device: /dev/video0
- **Fixed focus** — no autofocus controls, so blur is not a focus issue
- Camera supports two formats:
  - MJPEG: up to 1920x1080
  - YUYV raw: max 640x480 only
- Default negotiated format: **YUYV at 640x360** (confirmed via ffmpeg and v4l2-ctl)
- Upscaling 640x360 → display resolution = blurry
- PipeWire version: 1.4.10
- WirePlumber running as user service
- PipeWire node names for camera:
  - `v4l2_input.pci-0000_00_14.0-usb-0_9_1.2` (default source, wpctl ID 73)
  - `v4l2_input.pci-0000_00_14.0-usb-0_9_1.0` (wpctl ID 72)
- libcamera also detects the camera (wpctl devices 71, 91, 92) but exposes no PipeWire Sources

## Root Cause (confirmed)
PipeWire's spa-v4l2 plugin independently negotiates the capture format via `VIDIOC_S_FMT`
when an app connects. It defaults to YUYV. **There is no simple config knob to override this.**

- WirePlumber `update-props` (video.format, video.width, video.height) only sets PipeWire node
  metadata — spa-v4l2 ignores these during actual format negotiation. Not a real constraint.
- udev `ACTION=="add"` sets format too early — PipeWire calls `VIDIOC_S_FMT` itself on client
  connect and overrides it back to YUYV.

## What Was Tried

### 1. WirePlumber rule — attempt 1
- File: `~/.config/wireplumber/wireplumber.conf.d/50-camera-resolution.conf`
- Set bare `video.width = 1280`, `video.height = 720` with no surrounding structure
- Result: **no effect** — silently ignored, wrong syntax entirely

### 2. WirePlumber rule — attempt 2 (correct syntax, wrong mechanism)
- Replaced with proper `monitor.v4l2.rules` block:
  ```
  monitor.v4l2.rules = [
    {
      matches = [
        {
          node.name = "~v4l2_input*"
        }
      ]
      actions = {
        update-props = {
          video.format = "MJPG"
          video.width  = 1280
          video.height = 720
        }
      }
    }
  ]
  ```
- Result: **no effect** — rule is parsed correctly and node glob matches, but `update-props`
  does not constrain spa-v4l2 capture format. `v4l2-ctl --get-fmt-video` still shows YUYV 640x360.
- Lesson: WirePlumber `update-props` is metadata only, not a format lock for v4l2 sources.

### 3. udev rule — staged but abandoned
- File staged at `scotrland/99-camera-mjpeg.rules`
- Would run `v4l2-ctl --set-fmt-video` on device connect
- Abandoned: PipeWire overrides the format on client connect anyway, making this ineffective.

## Next Steps (ranked)

### Option A — v4l2loopback bridge (recommended, reliable)
Create a virtual `/dev/video10` that only offers MJPEG 1280x720, sourced from the real camera.
Apps use the virtual device and always get a sharp image.

```bash
# Install
sudo dnf install v4l2loopback kmod-v4l2loopback gstreamer1-plugins-good

# Load module
sudo modprobe v4l2loopback video_nr=10 card_label="HD Camera" exclusive_caps=1

# Test bridge
gst-launch-1.0 v4l2src device=/dev/video0 \
  ! image/jpeg,width=1280,height=720,framerate=30/1 \
  ! v4l2sink device=/dev/video10
```

If bridge works: make persistent via:
- `/etc/modprobe.d/v4l2loopback.conf` for module autoload options
- `/etc/modules-load.d/v4l2loopback.conf` for autoloading on boot
- Systemd user service to run the GStreamer bridge on login

### Option B — libcamera source (quick test, uncertain outcome)
libcamera detects the camera but no PipeWire Sources are exposed.
Worth checking if they can be enabled — libcamera may negotiate MJPEG by default.

```bash
pw-cli list-objects | grep -A8 "libcamera\|media.class.*Video"
```
