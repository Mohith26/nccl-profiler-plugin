# NCCL Profiler Plugin / Mosaic with RCCL Tests

Detailed instructions for using the NCCL profiler plugin and Mosaic with AMD GPUs and rccl_tests.

## Build otel, rccl, rccl-tests, and nccl-profiler-plugin

### Set key environent variables

```bash
export WORKDIR=$HOME/projects # Change accordingly
export NCCL_PATH=$WORKDIR/rccl
export OTEL_PATH=$WORKDIR/opentelemetry-install
```

### Build

```bash
mkdir -p $WORKDIR && cd $WORKDIR

# Install OTEL locally
cd $WORKDIR
git clone https://github.com/open-telemetry/opentelemetry-cpp.git
mkdir -p $OTEL_PATH
cd opentelemetry-cpp
mkdir -p build && cd build
cmake -DCMAKE_INSTALL_PREFIX=$OTEL_PATH \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
  -DBUILD_SHARED_LIBS=ON -DWITH_OTLP_GRPC=ON \
  -DWITH_OTLP_HTTP=ON \
  -DBUILD_TESTING=OFF \
  -DOPENTELEMETRY_INSTALL=ON \
  -DWITH_PROMETHEUS=ON ..
cmake --build . --parallel 64
cmake --install .

# Prepare rccl
cd $WORKDIR
git clone --recursive https://github.com/ROCm/rccl.git
cd rccl
mkdir -p build && cd build
cmake ..

# Build rccl-test
cd $WORKDIR
git clone https://github.com/ROCm/rccl-tests.git
cd rccl-tests
make -j MPI=1 MPI_HOME=/usr/lib/x86_64-linux-gnu/openmpi

# Build the Profiler Plugin
cd $WORKDIR
# Use SSH to avoid password prompts
git clone git@gitub/delosdata/nccl-profiler-plugin.git
cd nccl-profiler-plugin
make
```

## Run otel-lgtm stack

The otel-lgtm stack will display the nccl-profiler-plugin metrics as-is. To get more detailed information about GPU health, network health, and ROCm processes the prometheus configuration must be modified.

### Generate modified promethus configuration

A utility script is provided at `otel/prometheus-config-generate.sh` that will produce a new prometheus configuration file that includes the additional scrape configurations necessary to get all metrics into Mosaic.

Copy the `otel/prometheus-config-generate.yaml.example` to `otel/prometheus-config-generate.yaml` and modify to include all the host machines that will be part of the RCCL tests.

```bash
cd $WORKDIR
./nccl-profiler-plugin/otel/prometheus-config-generate.sh \
  --generate-config ./nccl-profiler-plugin/otel/prometheus-config-generate.yaml \
  --output ./nccl-profiler-plugin/otel/prometheus-amd.yaml \
  --base-config ./nccl-profiler-plugin/otel/prometheus.yaml
```

### Launch otel-lgtm stack (on head node)

```bash
cd $WORKDIR
docker compose -f ./nccl-profiler-plugin/otel/docker-compose.yml -f ./nccl-profiler-plugin/otel/docker-compose.amd.yml up -d
```

Mosaic will now be available on the head node at: http://localhost:3000

### Launch monitoring stack (on each node)

The monitoring processes must be running on each workstation that will be part of the RCCL tests.  The following must be run on each machine.

```bash
cd $WORKDIR
docker compose -f ./nccl-profiler-plugin/otel/amd-gpu-monitoring/docker-compose.yml --profile=node-exporter --profile=amd-device-exporter up -d
```

## Run `rccl-tests`

Use mpi to run a RCCL test across all workstations. Create a shell script to run via mpi to set appropriate environment vars and use the nccl profiler plugin and send telemetry data to the otel-lgtm collector.

### Create script to run tests

Create a file `$WORKDIR/rccl_test_script.sh` with the following.

```bash
#!/bin/bash

export WORKDIR=$HOME/projects # Change this accordingly
export NCCL_DEBUG=INFO
export NCCL_IB_HCA="^=mlx5_0,mlx5_1" # Change as necessary to exclude specific devices
export NCCL_IB_GID_INDEX=3
export NCCL_DEBUG_FILE=rccl_%h_%p.log
export NCCL_PROFILER_PLUGIN=$WORKDIR/nccl-profiler-plugin/build/libnccl-profiler-otel.so
export NCCL_PROFILER_OTEL_TELEMETRY_ENDPOINT=http://{head_node}:4318 # Head node will collect OTEL metrics
export LD_LIBRARY_PATH="$WORKDIR/opentelemetry-install/lib:$LD_LIBRARY_PATH"

# Example of setting backend interface based based on hostname.
# Adjust as necessary.
if [ "$HOSTNAME" = "head_node" ]; then
    export NCCL_SOCKET_IFNAME=ens51f1np1
elif [ "$HOSTNAME" = "worker_node_a" ]; then
    export NCCL_SOCKET_IFNAME=ens61f1np1
else
    echo "Unknown hostname: $HOSTNAME"
    exit 1
fi

$WORKDIR/rccl-tests/build/all_reduce_perf -b 1M -e 256M -f 2 -g 8 -c 0 -N 0
```

### Use mpi to run the test script

```bash
CD $WORKDIR
TORCH_NCCL_HIGH_PRIORITY=1 \
RCCL_MSCCL_ENABLE=0 \
mpirun -H head_node,worker_node_a \
  --allow-run-as-root \
  --mca btl_tcp_if_include 10.100.0.0/16 \
  -np 2 \
  ./rccl_test_script.sh
```
