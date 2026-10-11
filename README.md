# jellyfin-server

This repo is a home-media stack for running Jellyfin through a VPN tunnel. It packages Jellyfin with Gluetun so outbound traffic is routed through NordVPN, while keeping the media server setup reproducible and easy to run on a self-hosted server.

In practice, this project defines the Docker-based configuration for a private Jellyfin instance and the VPN sidecar needed to secure or mask network access for streaming/media traffic.

## Run it

Prerequisites:
- Docker
- A NordVPN account and VPN credentials

Before starting, follow [Transcoding Setup](#transcoding-setup) to adapt the GPU configuration to your host, or remove it for CPU-only transcoding.

From the project root, copy the example environment file to a real `.env` file and fill in your VPN credentials before starting the stack:

```bash
cp .env.example .env
# or rename the sample file if your project uses a different example name

docker compose up -d
```

Then open the Jellyfin web UI in your browser (default port usually `8096` unless you change the compose configuration). If you need to inspect logs or stop the services:

```bash
docker compose logs -f
docker compose down
```

This will bring up Jellyfin and the Gluetun VPN container together so Jellyfin traffic is routed through the tunnel.

## Transcoding Setup

The current [docker-compose.yaml](docker-compose.yaml) exposes `/dev/dri/renderD128` to Jellyfin and adds supplementary group `991` for an Intel GPU setup. These values are host-specific; passing a device into Docker does not automatically enable hardware acceleration in Jellyfin.

The instructions below target Docker Engine on a Linux host using the official `jellyfin/jellyfin` image. Keep `network_mode: "service:gluetun"`, volumes, and Gluetun configuration unchanged when editing the Jellyfin service.

| Hardware | Jellyfin hardware acceleration | Container setup |
| --- | --- | --- |
| Supported Intel integrated GPU or Arc GPU | Intel QuickSync (QSV), or VA-API where QSV is unsupported | Render device and its host group ID |
| Supported AMD integrated or discrete GPU | VA-API on Linux | Render device and its host group ID |
| Supported NVIDIA GPU | NVIDIA NVENC | NVIDIA driver, Container Toolkit, and GPU reservation |
| No supported GPU | None | Remove GPU device and group settings |

Codec support depends on the exact GPU, driver, and Jellyfin version. Not every GPU has a video encoder. Consult Jellyfin's [Intel](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/), [AMD](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/amd/), or [NVIDIA](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/nvidia/) guide for hardware requirements, supported codecs, and advanced options such as HDR tone mapping.

### Intel and AMD on Linux

Ensure the GPU is enabled in the host BIOS/UEFI and the appropriate kernel driver and firmware are installed. The official Jellyfin image includes the Intel and AMD user-space media drivers; the host must still expose a working GPU.

Inspect the available render devices and the numeric group ID owning the selected device:

```bash
ls -l /dev/dri/
ls -l /dev/dri/by-path/
stat -c '%g' /dev/dri/renderD128
```

On multi-GPU hosts, use the `by-path` links to identify the intended GPU rather than assuming it is `renderD128`. Replace both sides of the Jellyfin `devices` mapping with the selected path, and replace `"991"` in `group_add` with the numeric ID reported by `stat`. The owning group is commonly `render`, but its name and ID vary by distribution. If your setup also requires access to another GPU device, add that device's owning group ID too. Do not work around permission errors by making GPU devices world-writable or enabling privileged mode.

After starting Jellyfin, open **Dashboard > Playback > Transcoding**:

- For supported Intel GPUs, select **Intel QuickSync (QSV)**. Use **VA-API** for older or otherwise unsupported QSV hardware, following the Intel guide.
- For AMD GPUs, select **VA-API**, not AMF (the recommended Linux method differs from Windows).
- Set the render-device path, where offered, to the path inside the container, such as `/dev/dri/renderD128`.
- Enable hardware encoding and select only the hardware decoding codecs your GPU supports, then save.

Check device access and VA-API capabilities from inside the container, adjusting the path if needed:

```bash
docker compose exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
```

If the device is missing or initialization fails, check the host driver, mapped device, and group permissions before enabling more transcoding options.

### NVIDIA on Linux

Install a supported NVIDIA driver on the host and confirm that `nvidia-smi` works there. Install and configure the [NVIDIA Container Toolkit for Docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html), including its Docker runtime configuration steps. Installing the toolkit alone is not enough.

Remove the existing Intel/AMD `devices` and `group_add` entries from the Jellyfin service. Add the following fields under that existing service, preserving its other settings. This is a partial configuration, not a replacement compose file:

```yaml
services:
  jellyfin:
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

This requests one NVIDIA GPU. On a multi-GPU host, replace `count` with `device_ids: ["0"]` (using the intended NVIDIA GPU ID); do not specify both. The official Jellyfin image already sets the NVIDIA environment variables needed for transcoding.

After recreating the container, verify that it can see the GPU:

```bash
docker compose exec jellyfin nvidia-smi
```

In **Dashboard > Playback > Transcoding**, select **NVIDIA NVENC**, enable hardware encoding, select only supported hardware decoding codecs, and save.

### CPU-only and Other Platforms

For CPU-only transcoding, remove the Jellyfin service's GPU `devices` and `group_add` entries, plus any NVIDIA `runtime` and GPU reservation settings you added. Set hardware acceleration to **None** in Jellyfin. Leave Gluetun's `/dev/net/tun` device mapping intact. Software transcoding uses more CPU, especially for high-resolution video.

Do not assume these Linux GPU mappings work on Docker Desktop for macOS or Windows. For native installations or other hardware, consult Jellyfin's [hardware acceleration documentation](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/), including its Apple and Rockchip guides. Hardware acceleration on ARM devices and NAS appliances depends on the specific device and container environment; use CPU-only mode if it is unsupported.

### Apply and Verify

Validate and apply compose changes from the project root:

```bash
docker compose config --quiet
docker compose up -d
```

Play a video and lower its quality or bitrate enough to force **video transcoding**. Direct Play, Direct Stream, or audio-only transcoding does not test the video encoder. Inspect the FFmpeg transcoding log under **Dashboard > Logs** for hardware decoder/encoder usage (such as `qsv`, `vaapi`, or `nvenc`) and errors. For Intel, `intel_gpu_top` on the host can show video-engine activity; for NVIDIA, `nvidia-smi` can show the FFmpeg process using the GPU. A successful device check alone does not prove playback is hardware-transcoded.

If you run this on a remote server you can use SSH to forward traffic easily by running:

## Tips & Tricks

```bash
ssh -CN -L8096:<remote-host>:8096
# -CN enables compression and only forwarding traffic.
```

Then you should be able to setup the Jellyfin on your remote host through `localhost:8096` instead of having to be physically connected to the remote server or using remote desktop to configure it.
