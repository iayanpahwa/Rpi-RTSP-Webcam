# Rpi-RTSP-Webcam

Stream a USB webcam from a Raspberry Pi 4 over RTSP, compatible with [Frigate NVR](https://frigate.video) and [Home Assistant](https://home-assistant.io).

**Stack:** MediaMTX (RTSP server) + FFmpeg + Balena OS

**Stream URL:** `rtsp://<user>:<password>@<device-ip>:8554/cam`

---

## Setup

### 1. Enable hardware acceleration (Balena Dashboard)

Before deploying, set these **Device Configuration** variables in the [balenaCloud Dashboard](https://dashboard.balena-cloud.com/) for your device or fleet:

| Variable | Value | Purpose |
|----------|-------|---------|
| `RESIN_HOST_CONFIG_gpu_mem` | `128` | Allocates GPU memory for the VPU hardware codec |
| `RESIN_HOST_CONFIG_start_x` | `1` | Loads full firmware blob required for HW video encode/decode |
| `RESIN_HOST_CONFIG_dtoverlay` | `vc4-kms-v3d` | Enables the GPU driver |

After saving, the device will reboot and expose `/dev/video11` (H.264 encoder) on the host.

### 2. Set credentials

Edit credentials directly in `media/mediamtx.yml`, or set as Balena fleet/device variables for a more secure setup.

### 3. Deploy via Balena

```bash
balena push <your-fleet-name>
```

Or deploy in one click:

[![deploy button](https://balena.io/deploy.svg)](https://dashboard.balena-cloud.com/deploy?repoUrl=https://github.com/iayanpahwa/Rpi-RTSP-Webcam)

### 4. Frigate integration

Add to your `go2rtc` streams in `frigate-config.yml`:
```yaml
go2rtc:
  streams:
    outdoor:
      - rtsp://admin:<password>@<device-ip>:8554/cam#rtsp_transport=tcp
```

---

## Hardware

- Raspberry Pi 4
- Any UVC-compatible USB webcam (tested with Lenovo FHD Webcam)

---

## Pipeline

```
[USB Webcam] --MJPEG--> [V4L2 capture]
  --> [FFmpeg: MJPEG decode + drawtext + yuv420p conversion]  (CPU)
  --> [h264_v4l2m2m HW encode via bcm2835-codec VPU]          (VPU)
  --> [MediaMTX RTSP server :8554/cam]
  --> [Frigate NVR]
```

H.264 encoding is offloaded to the RPi4's VideoCore VI VPU via `h264_v4l2m2m` (V4L2 M2M API), keeping CPU usage low (~20-40% vs ~80-100% with software encoding).

### Why `format=yuv420p` is required

The webcam outputs MJPEG which decodes to `yuvj422p` (4:2:2 chroma subsampling). The RPi4 VPU only accepts `yuv420p` (4:2:0). A CPU-side colorspace conversion filter bridges the two — it is computationally cheap (simple row averaging, SIMD-optimized).

---

## Notes

- Webcam captured via V4L2 in MJPEG format at 1280×720 30fps
- Encoded to H.264 at 900kbps with HH:MM timestamp overlay
- RTSP served on port 8554 over TCP/UDP/multicast
- Keyframe every ~1s (`-g 30` at 30fps) for reliable Frigate detection
