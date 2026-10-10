# jellyfin-server

This repo is a home-media stack for running Jellyfin through a VPN tunnel. It packages Jellyfin with Gluetun so outbound traffic is routed through NordVPN, while keeping the media server setup reproducible and easy to run on a self-hosted server.

In practice, this project defines the Docker-based configuration for a private Jellyfin instance and the VPN sidecar needed to secure or mask network access for streaming/media traffic.

## Run it

Prerequisites:
- Docker
- A NordVPN account and VPN credentials

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

If you run this on a remote server you can use SSH to forward traffic easily by running:

## Tips & Tricks

```bash
ssh -CN -L8096:<remote-host>:8096
# -CN enables compression and only forwarding traffic.
```

Then you should be able to setup the Jellyfin on your remote host through `localhost:8096` instead of having to be physically connected to the remote server or using remote desktop to configure it.
