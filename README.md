# Rpi-RTSP-Webcam

Stream a USB webcam from a Raspberry Pi 4 over RTSP, compatible with [Frigate NVR](https://frigate.video) and [Home Assistant](https://home-assistant.io).

**Stack:** MediaMTX (RTSP server) + FFmpeg + Balena OS

**Stream URL:** `rtsp://<user>:<password>@<device-ip>:8554/cam`

---

## Setup

### 1. Set credentials

Copy `.env.example` to `.env` and set a password:
```bash
cp .env.example .env
```

For Balena deployments, set `RTSP_PASSWORD` as a fleet or device variable in the Balena dashboard instead of using a `.env` file.

### 2. Deploy via Balena

```bash
balena push <your-fleet-name>
```

Or deploy in one click:

[![deploy button](https://balena.io/deploy.svg)](https://dashboard.balena-cloud.com/deploy?repoUrl=https://github.com/iayanpahwa/Rpi-RTSP-Webcam)

### 3. Frigate integration

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

## Notes

- Webcam is captured via V4L2 in MJPEG format at 1280×720 30fps
- Encoded to H.264 (libx264, ultrafast) at 900kbps with timestamp overlay
- RTSP served on port 8554 over TCP/UDP/multicast
