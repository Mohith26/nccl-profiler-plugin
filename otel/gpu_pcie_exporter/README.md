# GPU PCIe Exporter

Prometheus exporter for GPU to PCIe port mapping. Discovers GPUs using `nvidia-smi` (NVIDIA) or `rocm-smi` (AMD) and exposes metrics with labels for host, gpu_id, pcie_port, and vendor.

## Building the Docker Image

From the `gpu_pcie_exporter` directory:

```bash
docker build -t gpu-pcie-exporter .
```

Or from the parent directory using docker-compose:

```bash
docker compose build gpu-pcie-exporter
```

**Note:** If you encounter import errors after updating dependencies, rebuild without cache:

```bash
docker compose build --no-cache gpu-pcie-exporter
```

## Running

### Using Docker Compose

The service is configured in `amd-gpu-monitoring/docker-compose.yml`. Start it with:

```bash
docker compose up -d gpu-pcie-exporter
```

### Using Docker Directly

```bash
docker run -d \
  --name gpu-pcie-exporter \
  -p 8000:8000 \
  -v /opt/rocm:/opt/rocm:ro \
  -v /usr/bin:/host/usr/bin:ro \
  -e PATH=/opt/rocm/bin:/usr/bin:/usr/local/bin:/usr/sbin:/bin:/sbin \
  -e LD_LIBRARY_PATH=/opt/rocm/lib:/usr/lib:/usr/local/lib \
  gpu-pcie-exporter
```

## Accessing Metrics

Once running, metrics are available at:

```
http://localhost:8000/metrics
```

## Configuration

The exporter listens on port 8000 by default. You can customize this by:

1. Setting the `--port` argument in the Dockerfile CMD
2. Using environment variables (if you modify the script to support them)
3. Overriding the command in docker-compose.yml

## Requirements

The container needs access to:
- `/opt/rocm/bin/rocm-smi` (for AMD GPUs)
- `/usr/bin/nvidia-smi` (for NVIDIA GPUs)

These are mounted from the host system via volumes in docker-compose.yml.
