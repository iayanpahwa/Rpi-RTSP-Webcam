#   
To access the Raspberry Pi 4's hardware H.264 decoder inside Docker, you must pass the specific hardware device files from the host into the container and ensure the user inside the container has permission to use them.   
  
## 1. Identify the Correct Device Nodes  
On the Raspberry Pi 4, hardware video acceleration is typically exposed through **V4L2 (Video for Linux 2)** device nodes:  
  
* **/dev/video10**: Hardware H.264 **Decoder**.  
* **/dev/video11**: Hardware H.264 **Encoder**.  
* **/dev/video12**: Hardware Image Sensor Pipeline (ISP) for resizing/format conversion.   
  
## 2. Configure Docker to Access Hardware   
You can grant access using either the Docker CLI or Docker Compose.   
  
**Using Docker CLI:**  
```
docker run -it \
  --device /dev/video10 \
  --device /dev/video11 \
  --device /dev/video12 \
  --group-add video \
  your-image-name

```
  
* --device: Maps the host's hardware decoder into the container.  
* --group-add video: Gives the container user the necessary permissions to access these devices.   
  
**Using Docker Compose:**  
```
services:
  app:
    image: your-image-name
    devices:
      - "/dev/video10:/dev/video10"
      - "/dev/video11:/dev/video11"
      - "/dev/video12:/dev/video12"
    group_add:
      - video

```
##   
## 3. Verification Inside the Container  
Once inside the container, verify that FFmpeg can see the hardware codecs:   
```
ffmpeg -codecs | grep v4l2m2m

```
You should see h264_v4l2m2m in the output. To test decoding specifically, run: [8]   
```
ffmpeg -c:v h264_v4l2m2m -i input.mp4 -f null -

```
## Important Requirements  
  
* **GPU Memory:** Ensure you have allocated enough memory to the GPU in /boot/config.txt (e.g., gpu_mem=128 or higher).  
* **64-bit OS:** If you are on a 64-bit OS, **V4L2 M2M** is the primary supported API for hardware decoding; older APIs like OpenMAX (OMX) are largely deprecated for the Pi 4.  
* **Base Image:** Ensure your container's base image (like Debian) includes the v4l-utils and a version of FFmpeg compiled with v4l2m2m support.   
  
Do you need help with a **specific application** like Jellyfin or Frigate, or are you writing a **custom script**?  
  
** balenaOS fully supports the Raspberry Pi 4's hardware H.264 decoder**. Because balenaOS is designed specifically for containerized workloads, it includes the necessary drivers and kernel support to pass these hardware capabilities through to your Docker containers.   
  
## How to Enable it in Balena  
To get it working, you need to configure both the **Host OS** (via the balenaCloud dashboard or config.txt) and your **Docker configuration**.   
  
## 1. Host Configuration (balenaCloud Dashboard)  
Hardware acceleration on the Pi 4 requires specific GPU memory and firmware settings. You can set these via **Device Configuration** variables in the ++[balenaCloud Dashboard](https://dashboard.balena-cloud.com/)++:   
  
* **RESIN_HOST_CONFIG_gpu_mem**: Set to at least 128 (recommended for 1080p decoding).  
* **RESIN_HOST_CONFIG_start_x**: Set to 1 to enable the full firmware with hardware video decoding support.  
* **RESIN_HOST_CONFIG_dtoverlay**: Ensure "vc4-kms-v3d" or "vc4-fkms-v3d" is present to enable the GPU driver.   
## 2. Container Configuration (docker-compose.yml)   
In a balena multicontainer setup, hardware devices are not exposed by default. You must explicitly map them or use privileged mode:   
```
services:
  video-processor:
    image: balenalib/raspberrypi4-64-debian:latest
    privileged: true  # Simplest way to grant full hardware access
    environment:
      - UDEV=1        # Automatically populates /dev/video* nodes in the container

```
  
* **UDEV=1**: This is a balena-specific feature for their base images (e.g., ++[balenalib/raspberrypi4-64](https://hub.docker.com/r/balenalib/raspberrypi4-64)++) that dynamically handles device plugging and permissions.   
  
## Verification  
Once your container is running, you can SSH into it via the balena CLI or dashboard and check for the decoder:   
```
# List available hardware video devices
ls /dev/video* 

# Check for H.264 decoder support (usually /dev/video10)
v4l2-ctl -d /dev/video10 --list-formats-out

```
