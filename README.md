# Performance Analysis of Virtual Machines and Containers


To experimentally compare the performance, resource utilization, application performance, and
scalability of Virtual Machines (VMs) and Containers under identical workloads.

---

## Table of Contents

1. [Abstract](#abstract)
2. [Objectives](#objectives)
3. [Architecture](#architecture)
4. [Experimental Environment](#experimental-environment)
5. [Hardware Configuration](#hardware-configuration)
6. [Software Configuration](#software-configuration)
8. [Setup Instructions](#setup-instructions)
9. [Experiments](#experiments)
    - [Baseline](#baseline-measurement)
    - [CPU](#experiment-1--cpu-performance)
    - [Memory](#experiment-2--memory-performance)
    - [Disk I/O](#experiment-3--disk-io-performance)
    - [Network](#experiment-4--network-performance)
    - [FastAPI Application](#experiment-5--fastapi-application-performance)
    - [Startup Time](#experiment-6--startup-time)
    - [Scalability](#experiment-7--scalability)
11. [Automation Scripts](#automation-scripts)
12. [Result Collection (CSV)](#result-collection-csv)
13. [Results](#results)
14. [Statistical Analysis](#statistical-analysis)
15. [VM vs Container Comparison](#vm-vs-container-comparison)
16. [Discussion](#discussion)
17. [Limitations](#limitations)
19. [Reproduction Instructions](#reproduction-instructions)
21. [Conclusion](#conclusion)
22. [Final Project Structure](#final-project-structure)

---

## Abstract

Virtual Machines and containers are the two dominant ways of isolating and deploying workloads. VMs virtualize hardware and run a full guest operating system on top of a hypervisor; containers share the host kernel and isolate processes using namespaces and cgroups. This project measures how these differences show up in practice.

The same workloads are executed in both environments: CPU (prime computation), memory (block transfers), disk I/O (sequential and random read/write), network throughput, a FastAPI web application, startup time, and scalability under increasing load. Every experiment is repeated multiple times, raw outputs are preserved, results are processed into CSV files, analysed statistically with Python, and published with scripts and documentation so that the work is fully reproducible.


## Objectives

- Prepare a consistent and documented benchmarking environment for both VM and container.
- Measure CPU, memory, disk, and network performance with industry-standard tools (Sysbench, fio, iperf3).
- Evaluate real application behaviour using a FastAPI service under load (Apache Benchmark / wrk).
- Compare environment and application startup times.
- Study how each environment scales as workload increases.
- Automate benchmark execution and process results with Pandas / Matplotlib.
- Publish code, raw data, graphs, and documentation on GitHub.

## Architecture

```
                 PERFORMANCE ANALYSIS
                        │
          ┌─────────────┴─────────────┐
          │                           │
    VIRTUAL MACHINE               CONTAINER
 (VMware + Ubuntu guest)            (Docker)
          │                           │
          └─────────────┬─────────────┘
                        │
                  SAME WORKLOADS
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
         CPU         Memory     Disk / Network
                        │
                        ▼
                APPLICATION TEST
                        │
                        ▼
                  FINAL ANALYSIS
```



## Experimental Environment

| Component | Choice |
|---|---|
| Host OS | Windows |
| Hypervisor | VMware Workstation |
| Guest OS | Ubuntu (24.04 LTS recommended) |
| Container runtime | Docker |
| Programming language | Python |
| CPU / Memory benchmark | Sysbench |
| Disk benchmark | fio |
| Network benchmark | iperf3 |
| API framework | FastAPI (served by Uvicorn) |
| HTTP load tools | Apache Benchmark (`ab`), `wrk` (optional) |
| Analysis | Pandas, Matplotlib, NumPy, Jupyter |

## Hardware Configuration

| Resource | VM | Container |
|---|---|---|
| CPU | 4 vCPU | `--cpus=4` |
| Memory | 8 GB | `--memory=6g` |
| Disk | 60 GB virtual disk | Dedicated benchmark directory mounted via `-v` |
| Base OS | Ubuntu 24.04 LTS | `ubuntu:24.04` image |
| Network | NAT or Bridged (fixed) | Documented Docker network mode |

## Software Configuration

| Software | Version |
|---|---|
| Ubuntu | _to be filled_ |
| Kernel (`uname -a`) | _to be filled_ |
| Docker | _to be filled_ (`docker --version`) |
| Sysbench | _to be filled_ |
| fio | _to be filled_ |
| iperf3 | _to be filled_ |
| Python | _to be filled_ |

---

**Terminal rule**

- Linux commands: run in the Ubuntu terminal inside VMware Workstation.
- Docker commands: run in the terminal where Docker is installed.
- Always start from the project root before using relative paths:

```bash
cd ~/vm-vs-container-performance
pwd    # /home/<your-ubuntu-user>/vm-vs-container-performance
```

**Performance difference formulas**

For execution time (lower is better):

```python
difference = ((vm_time - container_time) / vm_time) * 100
print(f"Performance difference: {difference:.2f}%")
```

For throughput (higher is better):

```python
difference = ((container_throughput - vm_throughput) / vm_throughput) * 100
print(f"Throughput difference: {difference:.2f}%")
```

---

## Setup Instructions

### 1. Install benchmark tools (Ubuntu)

```bash
sudo apt update
sudo apt install -y sysbench fio iperf3 htop iotop sysstat python3 python3-pip git
```

Verify:

```bash
sysbench --version
fio --version
iperf3 --version
python3 --version
git --version
```

### 2. Create the project directory

```bash
mkdir -p ~/vm-vs-container-performance
cd ~/vm-vs-container-performance
mkdir -p docs results/raw results/processed results/figures scripts workloads docker api analysis
```

### 3. Record hardware and software configuration

```bash
lscpu     > docs/cpu-info.txt
free -h   > docs/memory-info.txt
lsblk     > docs/storage-info.txt
uname -a  > docs/kernel-info.txt
docker --version
docker info
```

### 4. Create the VM environment

In VMware Workstation: open the Ubuntu VM → *Edit virtual machine settings* → set **Processors = 4**, **Memory = 8 GB**, **Hard Disk = 60 GB**, **Network Adapter = NAT or Bridged** → Apply → Power on.

Inside the VM:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y sysbench fio iperf3 htop sysstat python3 python3-pip git

nproc        # confirm vCPUs
free -h      # confirm memory
lsblk
df -h
```

Record the final settings:

```bash
cd ~/vm-vs-container-performance
nano docs/vm-configuration.txt
```

### 5. Create the Docker environment

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker

docker --version
sudo docker run --rm hello-world

# run Docker without sudo (log out and back in afterwards)
sudo usermod -aG docker $USER
docker run --rm hello-world
```

**Controlled container run**
```bash
docker run --rm --cpus=4 --memory=8g vm-container-benchmark
```

### 6. Build the benchmark Docker image

Create `docker/Dockerfile`:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update && \
    apt-get install -y \
        sysbench \
        fio \
        iperf3 \
        python3 \
        python3-pip \
        procps \
        sysstat && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /benchmark
```

Build from the project root (`-f` selects the Dockerfile, the final `.` is the build context):

```bash
cd ~/vm-vs-container-performance
docker build -t vm-container-benchmark -f docker/Dockerfile .
docker images
docker run --rm -it vm-container-benchmark
# inside the container:
sysbench --version && fio --version && python3 --version
```

---

## Experiments

All commands below are run from the project root unless stated otherwise. Parameters are identical between VM and container.

### Baseline Measurement

Reference CPU measurement before VM-vs-container comparison.

```bash
mkdir -p results/raw/baseline
sysbench cpu --cpu-max-prime=20000 --threads=4 --time=30 run > results/raw/baseline/cpu.txt
```

---

### Experiment 1 — CPU Performance

| Item | Value |
|---|---|
| Tool | Sysbench |
| Workload | Prime number calculation |
| Threads | 1 / 2 / 4 / 8 |
| Duration | 30 seconds |
| Metrics | Events/sec, execution time |

**VM (10 repetitions):**

```bash
mkdir -p results/raw/cpu/vm
for i in {1..10}; do
  sysbench cpu --cpu-max-prime=20000 --threads=4 --time=30 run \
    > results/raw/cpu/vm/run$i.txt
done
```

**Container (10 repetitions, with controlled limits):**

```bash
mkdir -p results/raw/cpu/container
for i in {1..10}; do
  docker run --rm --cpus=4 --memory=8g \
    vm-container-benchmark \
    sysbench cpu --cpu-max-prime=20000 --threads=4 --time=30 run \
    > results/raw/cpu/container/run$i.txt
done
```

**CPU scalability:**

```bash
for threads in 1 2 4 8; do
  sysbench cpu --cpu-max-prime=20000 --threads=$threads --time=30 run
done
```

---

### Experiment 2 — Memory Performance

| Item | Value |
|---|---|
| Tool | Sysbench |
| Workload | Memory operations (1M blocks, 10G total) |
| Metrics | Operations/sec, throughput, latency |

**VM (10 repetitions):**

```bash
mkdir -p results/raw/memory/vm
for i in {1..10}; do
  sysbench memory --memory-block-size=1M --memory-total-size=10G --threads=4 run \
    > results/raw/memory/vm/run$i.txt
done
```

**Container (10 repetitions):**

```bash
mkdir -p results/raw/memory/container
for i in {1..10}; do
  docker run --rm --cpus=4 --memory=8g \
    vm-container-benchmark \
    sysbench memory --memory-block-size=1M --memory-total-size=10G --threads=4 run \
    > results/raw/memory/container/run$i.txt
done
```

**Resource monitoring** (run in a second terminal while benchmarks execute):

```bash
htop            # or: vmstat 1
docker stats    # for containers: CPU, memory, network I/O, block I/O
```

---

### Experiment 3 — Disk I/O Performance

| Item | Value |
|---|---|
| Tool | fio |
| Operations | Read / Write |
| Workloads | Sequential (1M blocks) / Random (4k blocks) |
| Metrics | MB/s, IOPS, latency |

Common parameters: `--size=2G --direct=1 --iodepth=16 --runtime=30 --time_based`

**VM:**

```bash
mkdir -p ~/fio-test results/raw/disk

# Sequential write
fio --name=seq-write --filename=$HOME/fio-test/testfile --size=2G --bs=1M \
    --rw=write --direct=1 --iodepth=16 --runtime=30 --time_based \
    > results/raw/disk/seq-write.txt

# Sequential read
fio --name=seq-read --filename=$HOME/fio-test/testfile --size=2G --bs=1M \
    --rw=read --direct=1 --iodepth=16 --runtime=30 --time_based \
    > results/raw/disk/seq-read.txt

# Random read
fio --name=random-read --filename=$HOME/fio-test/testfile --size=2G --bs=4k \
    --rw=randread --direct=1 --iodepth=16 --runtime=30 --time_based \
    > results/raw/disk/random-read.txt

# Random write
fio --name=random-write --filename=$HOME/fio-test/testfile --size=2G --bs=4k \
    --rw=randwrite --direct=1 --iodepth=16 --runtime=30 --time_based \
    > results/raw/disk/random-write.txt
```

**Container** (host directory mounted into the container; document the storage path used):

```bash
mkdir -p ~/fio-test results/raw/disk-container

# Sequential write
docker run --rm -v $HOME/fio-test:/fio-test vm-container-benchmark \
  fio --name=seq-write --filename=/fio-test/testfile --size=2G --bs=1M \
      --rw=write --direct=1 --iodepth=16 --runtime=30 --time_based \
  > results/raw/disk-container/seq-write.txt

# Random read
docker run --rm -v $HOME/fio-test:/fio-test vm-container-benchmark \
  fio --name=random-read --filename=/fio-test/testfile --size=2G --bs=4k \
      --rw=randread --direct=1 --iodepth=16 --runtime=30 --time_based \
  > results/raw/disk-container/random-read.txt
```

Repeat the same pattern for `seq-read` and `random-write`.

---

### Experiment 4 — Network Performance

| Item | Value |
|---|---|
| Tool | iperf3 |
| Metrics | Throughput, retransmissions |
| Test | Client ↔ Server |

```bash
mkdir -p results/raw/network

# On the server
iperf3 -s
ip addr                                   # find the server IP

# On the client
iperf3 -c <SERVER-IP> -t 30 > results/raw/network/iperf3.txt
iperf3 -c <SERVER-IP> -t 30 -P 4 > results/raw/network/iperf3-parallel4.txt
```

> Use the same client/server arrangement for VM and container tests, and document the network mode (NAT, bridged, host networking) because it changes results.

---

### Experiment 5 — FastAPI Application Performance

A realistic application workload with three endpoints: `/health` (lightweight), `/compute` (CPU-bound), `/memory` (memory-bound).

**`api/main.py`:**

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "healthy"}

@app.get("/compute")
def compute():
    total = 0
    for i in range(1_000_000):
        total += i * i
    return {"result": total}

@app.get("/memory")
def memory():
    data = [i for i in range(1_000_000)]
    return {"elements": len(data)}
```

**Run in the VM:**

```bash
cd ~/vm-vs-container-performance/api
python3 -m pip install fastapi uvicorn
uvicorn main:app --host 0.0.0.0 --port 8000

# second terminal
curl http://localhost:8000/health        # {"status":"healthy"}
```

**`api/requirements.txt`:**

```
fastapi
uvicorn
```

**`api/Dockerfile`:**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Build and run the container:**

```bash
cd ~/vm-vs-container-performance
docker build -t performance-api -f api/Dockerfile api
docker run --rm --cpus=4 --memory=8g -p 8000:8000 performance-api

# second terminal
curl http://127.0.0.1:8000/health
```

**Load testing:**

```bash
sudo apt install -y apache2-utils
ab -n 10000 -c 100 http://127.0.0.1:8000/health
ab -n 1000  -c 10  http://127.0.0.1:8000/compute

# optional
sudo apt install -y wrk
wrk -t4 -c100 -d30s http://127.0.0.1:8000/health
```

Metrics: requests per second, time per request, failed requests, connection times. Use identical request counts and concurrency for VM and container.

---

### Experiment 6 — Startup Time

```bash
# Container startup
time docker run --rm performance-api

# More controlled test
time docker run --rm -d --name startup-test -p 8000:8000 performance-api
docker stop startup-test
```

Record for both environments (multiple repetitions):

| | Environment started | Application ready |
|---|---|---|
| VM | _to be filled_ | _to be filled_ |
| Container | _to be filled_ | _to be filled_ |

---

### Experiment 7 — Scalability

Workload is increased in steps: **1 → 2 → 4 → 8**.

**CPU scalability:**

```bash
for threads in 1 2 4 8; do
  sysbench cpu --cpu-max-prime=20000 --threads=$threads --time=30 run
done
```

**API scalability:**

```bash
wrk -t1 -c10  -d30s http://127.0.0.1:8000/health
wrk -t2 -c50  -d30s http://127.0.0.1:8000/health
wrk -t4 -c100 -d30s http://127.0.0.1:8000/health
wrk -t4 -c200 -d30s http://127.0.0.1:8000/health
```

Record throughput, latency, CPU usage, and memory usage at each level.

---

## Automation Scripts

Scripts keep benchmark parameters consistent and reduce manual errors. Example `scripts/run_cpu.sh`:

```bash
#!/bin/bash
# Resolve paths relative to the project root, regardless of where the script is called from
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
OUTPUT_DIR="$SCRIPT_DIR/../results/raw/cpu"
mkdir -p "$OUTPUT_DIR"

for threads in 1 2 4 8
do
  echo "Running CPU test with $threads threads"
  sysbench cpu \
    --cpu-max-prime=20000 \
    --threads=$threads \
    --time=30 \
    run > "$OUTPUT_DIR/cpu_${threads}_threads.txt"
done

echo "CPU benchmark completed."
```

```bash
chmod +x scripts/run_cpu.sh
./scripts/run_cpu.sh
```

Equivalent scripts: `run_memory.sh`, `run_disk.sh`, `run_network.sh`, and `collect_metrics.py` for parsing raw output into CSV.

## Result Collection (CSV)


## Results


### CPU Results

| Threads | VM events/sec | Container events/sec | VM time (s) | Container time (s) |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 4 | | | | |
| 8 | | | | |

_Figure: `results/figures/cpu_performance.png`, `results/figures/cpu_scalability.png`_

### Memory Results

| Metric | VM | Container |
|---|---|---|
| Operations/sec | | |
| Throughput (MiB/s) | | |
| Latency (ms) | | |

### Disk I/O Results

| Workload | Metric | VM | Container |
|---|---|---|---|
| Sequential read | MB/s | | |
| Sequential write | MB/s | | |
| Random read | IOPS | | |
| Random write | IOPS | | |
| All | Latency (µs/ms) | | |

### Network Results

| Metric | VM | Container |
|---|---|---|
| Throughput (Mbps), single stream | | |
| Throughput (Mbps), 4 parallel streams | | |
| Retransmissions | | |

### Application (FastAPI) Results

| Endpoint | Metric | VM | Container |
|---|---|---|---|
| /health | Requests/sec | | |
| /health | Time per request (ms) | | |
| /health | Failed requests | | |
| /compute | Requests/sec | | |
| /compute | Time per request (ms) | | |
| /compute | Failed requests | | |

### Startup-Time Results

| Metric | VM | Container |
|---|---|---|
| Environment start (s) | | |
| Application ready (s) | | |

### Scalability Results

| Workload level | VM | Container |
|---|---|---|
| CPU 1 / 2 / 4 / 8 threads | | |
| API wrk 10 / 50 / 100 / 200 connections | | |

---

## Statistical Analysis

Computed with Pandas from `results/processed/cpu_results.csv`:

```python
import pandas as pd

df = pd.read_csv("results/processed/cpu_results.csv")

summary = df.groupby("environment")["events_per_second"].agg(
    ["mean", "median", "min", "max", "std"]
)
print(summary)
```

- **Mean**: average performance across repeated runs.
- **Median**: reduces the influence of outliers.
- **Standard deviation**: variability between runs.

| Environment | Mean | Median | Min | Max | Std Dev |
|---|---|---|---|---|---|
| VM | | | | | |
| Container | | | | | |

## Graphs

Generated by `scripts/analyze_results.py` and `scripts/generate_plots.py`, run from the project root:

```bash
pip install pandas matplotlib numpy jupyter
python3 scripts/analyze_results.py
python3 scripts/generate_plots.py
```


## VM vs Container Comparison

| Metric | VM | Container | Difference |
|---|---|---|---|
| CPU Performance (events/sec) | | | |
| Memory Usage / Throughput | | | |
| Sequential Read (MB/s) | | | |
| Sequential Write (MB/s) | | | |
| Random Read (IOPS) | | | |
| Random Write (IOPS) | | | |
| Network Throughput (Mbps) | | | |
| Startup Time (s) | | | |
| API Requests/sec | | | |
| API Latency (ms) | | | |

## Discussion



## Limitations



## Reproduction Instructions

1. Create the Ubuntu VM in VMware Workstation with fixed resources (4 vCPU, 8 GB RAM, 60 GB disk) and a fixed network mode.
2. Install the tools: `sysbench fio iperf3 htop iotop sysstat python3 python3-pip git`, then Docker.
3. Clone the repository:
   ```bash
   git clone <GITHUB-REPOSITORY-URL> ~/vm-vs-container-performance
   cd ~/vm-vs-container-performance
   ```
4. Record the environment into `docs/` (`lscpu`, `free -h`, `lsblk`, `uname -a`, `docker info`).
5. Build the images:
   ```bash
   docker build -t vm-container-benchmark -f docker/Dockerfile .
   docker build -t performance-api -f api/Dockerfile api
   ```
6. Run the experiments in order: baseline → CPU → memory → disk → network → FastAPI → startup → scalability.
7. Save raw output to `results/raw/`, build CSVs in `results/processed/`.
8. Run the analysis scripts from the project root to produce tables and graphs in `results/figures/`.
9. Fill in the Results, Discussion, and Conclusion sections.


## Conclusion

_To be written after all measurements and analysis are complete. The conclusion should be based strictly on the collected data, repeated runs, statistical analysis, and documented configuration, not on assumptions about which environment should perform better._

## Final Project Structure

```
vm-vs-container-performance/
│
├── README.md
├── .gitignore
│
├── docs/
│   ├── architecture.png
│   ├── cpu-info.txt
│   ├── memory-info.txt
│   ├── storage-info.txt
│   └── methodology.md
│
├── vm/
│   ├── setup.sh
│   └── benchmark.sh
│
├── docker/
│   ├── Dockerfile
│   └── benchmark.sh
│
├── api/
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── workloads/
│   ├── cpu/
│   ├── memory/
│   ├── disk/
│   └── network/
│
├── scripts/
│   ├── run_cpu.sh
│   ├── run_memory.sh
│   ├── run_disk.sh
│   ├── run_network.sh
│   ├── collect_metrics.py
│   ├── analyze_results.py
│   └── generate_plots.py
│
├── results/
│   ├── raw/
│   ├── processed/
│   └── figures/
│
└── analysis/
    └── analysis.ipynb
```
