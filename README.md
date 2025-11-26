# cupsane

[![Docker Hub](https://img.shields.io/docker/v/bartekmp/cupsane?label=Docker%20Hub&logo=docker)](https://hub.docker.com/r/bartekmp/cupsane)
[![GitHub Container Registry](https://img.shields.io/badge/ghcr.io-bartekmp%2Fcupsane-blue?logo=github)](https://ghcr.io/bartekmp/cupsane)
[![Build](https://github.com/bartekmp/cupsane/actions/workflows/build-and-push.yml/badge.svg)](https://github.com/bartekmp/cupsane/actions/workflows/build-and-push.yml)

A containerized CUPS print server with SANE scanner support and web-based scanning interface (scanservjs), designed for network printing and scanning with HP printers/scanners.

## Features

- **CUPS** - Network print server with web interface
- **SANE** - Scanner backend with network support
- **scanservjs** - Web-based scanning interface
- **HP printer/scanner support** via HPLIP
- **Persistent configuration** - Printer and scanner settings survive container restarts
- **Network accessible** - All services available on your LAN

## Services & Ports

- **8631** - CUPS web interface (IPP printing)
- **6566** - SANE network scanner daemon (for external SANE clients)
- **8632** - scanservjs web scanner interface (container listens on 8081)

## Architecture

The container runs three services:
- **saned** — owns the USB scanner, exposes it over the network
- **scanservjs** — web UI that connects to saned via SANE net backend
- **cupsd** — print server

This architecture avoids "Device busy" conflicts by having a single process (saned) access the USB device.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SANE_FORCE_HPAIO_ONLY` | `1` | Use minimal SANE config (hpaio only) for fast scanner detection |
| `SANED_PORT` | `6566` | Port for saned network scanner daemon |
| `TZ` | — | Timezone (e.g., `Europe/Warsaw`) |

## Deployment Methods

### 1. Docker Run

#### Quick Start (no persistence)
```bash
docker run --rm -d \
  --name cupsane \
  --privileged \
  -v /var/run/dbus:/var/run/dbus \
  -v /dev/bus/usb:/dev/bus/usb \
  -p 8631:631 \
  -p 6566:6566 \
  -p 8632:8081 \
  bartekmp/cupsane:latest
```

#### With Persistence
```bash
# Create data directories
mkdir -p ./storage/cups-config ./storage/cups-cache ./storage/cups-spool ./storage/scans

# Run container
docker run --rm -d \
  --name cupsane \
  --privileged \
  -v /var/run/dbus:/var/run/dbus \
  -v /dev/bus/usb:/dev/bus/usb \
  -v ./storage/cups-config:/etc/cups \
  -v ./storage/cups-cache:/var/cache/cups \
  -v ./storage/cups-spool:/var/spool/cups \
  -v ./storage/scans:/var/lib/scanservjs/output \
  -e TZ=Europe/Warsaw \
  -p 8631:631 \
  -p 6566:6566 \
  -p 8632:8081 \
  bartekmp/cupsane:latest
```

### 2. Docker Compose

Perfect for TrueNAS SCALE and other Docker environments.

```bash
# Create project directory
mkdir cupsane && cd cupsane

# Copy docker-compose.yml to this directory

# Create data directories
mkdir -p cups-config cups-cache cups-spool scans

# Start services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

### 3. Kubernetes (k3s/k8s)

#### Prerequisites
```bash
# Create persistent storage directories on the workstation node
sudo mkdir -p /opt/cupsd-data/{config,cache,spool}
sudo chmod 755 /opt/cupsd-data/{config,cache,spool}
```

#### Deploy
```bash
# Apply the deployment
kubectl apply -f cupsd-deployment.yaml

# Check status
kubectl get pods -l app=cupsane
kubectl logs -f deployment/cupsane

# Access services (replace <node-ip> with your workstation IP)
# CUPS: http://<node-ip>:8631
# Scanner web: http://<node-ip>:8632
```

#### Node Selector
The deployment includes a node selector to pin the pod to the `workstation` node where the USB printer is connected. Modify `nodeSelector` in `cupsd-deployment.yaml` if your node has a different hostname.

## Configuration

### CUPS Web Interface
Access at `http://<your-ip>:8631`

- Add printers via the web interface
- Configure print queues and sharing
- Monitor print jobs

### Scanner Web Interface
Access at `http://<your-ip>:8632`

- Web-based scanning interface
- Preview and adjust scan settings
- Save scans as PDF, PNG, JPG, or TIF
- Download scans directly from browser

### Network Configuration
The CUPS server is configured to allow access from:
- `192.168.0.0/24` (IPv4)
- `fd21:37::/64` (IPv6)

To modify network access, edit `cupsd.conf` and rebuild the image.

## Persistent Data

The following directories are persisted:
- `/etc/cups` - CUPS configuration and printer definitions
- `/var/cache/cups` - CUPS cache files
- `/var/spool/cups` - Print job queue
- `/var/lib/scanservjs/output` - Scanned documents

## Troubleshooting

### Check if printer is detected
```bash
# Docker
docker exec cupsane scanimage -L

# Kubernetes
kubectl exec deployment/cupsane -- scanimage -L
```

### View logs
```bash
# Docker
docker logs cupsane

# Docker Compose
docker-compose logs -f

# Kubernetes
kubectl logs deployment/cupsane
```

### Restart services
```bash
# Docker
docker restart cupsane

# Docker Compose
docker-compose restart

# Kubernetes
kubectl rollout restart deployment/cupsane
```

## Building from Source

```bash
# Build image
docker build -t cupsane .

# Test locally
docker run --rm --privileged -p 8631:631 -p 8632:8081 cupsane

# Note: to avoid slow scanner discovery, the image uses an hpaio-only SANE config by default.
# You can disable this with: -e SANE_FORCE_HPAIO_ONLY=0

# Push to registry
docker tag cupsane bartekmp/cupsane:latest
docker push bartekmp/cupsane:latest
```

## Requirements

- USB printer/scanner connected to the host
- Privileged container access (for USB device access)
- D-Bus support on host system

## Supported Printers

Primarily designed for HP printers/scanners via HPLIP, but also supports:
- Generic PostScript printers
- PDF printing (cups-pdf)
- Network printers via IPP
- AirScan-compatible scanners

## License

See LICENSE file for details.