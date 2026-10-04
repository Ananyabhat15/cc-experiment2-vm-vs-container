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
13. [Results](#results)
14. [Statistical Analysis](#statistical-analysis)
15. [VM vs Container Comparison](#vm-vs-container-comparison)
16. [Discussion](#discussion)
17. [Limitations](#limitations)
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
| Ubuntu | 22.04 LTS (x86_64) |
| Kernel (`uname -a`) | 5.15.0-x-generic |
| Docker | Docker Community Edition (CE); version number not reported |
| Sysbench | 1.0.20 |
| fio | 3.28 |
| iperf3 | 3.9 |
| Python | 3.10 |

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


## Results

### CPU Results

| Threads | VM events/sec | Container events/sec | VM time (s) | Container time (s) |
|---|---|---|---|---|
| 1 | 515.84 | 517.19 | Not reported | Not reported |
| 2 | 883.55 | 894.38 | Not reported | Not reported |
| 4 | 928.17 | 900.45 | Not reported | Not reported |
| 8 | 905.17 | 914.42 | Not reported | Not reported |

Latency reported for the same runs:

| Threads | VM avg (ms) | Container avg (ms) | VM P95 (ms) | Container P95 (ms) |
|---|---|---|---|---|
| 1 | 1.94 | 1.93 | 2.71 | 2.76 |
| 2 | 2.26 | 2.23 | 3.82 | 3.43 |
| 4 | 4.30 | 4.43 | 7.43 | 7.56 |
| 8 | 8.82 | 8.73 | 15.55 | 15.83 |

### Memory Results

| Metric | VM | Container |
|---|---|---|
| Operations/sec (1 thread) | 9,541.97 | 5,152.43 |
| Throughput (1 thread, MiB/s) | 9,541.97 | 5,152.43 |
| Avg latency (1 thread, ms) | 0.09 | 0.12 |
| P95 latency (1 thread, ms) | 0.25 | 0.32 |
| Operations/sec (2 threads) | 9,880.38 | 6,970.16 |
| Throughput (2 threads, MiB/s) | 9,880.38 | 6,970.16 |
| Avg latency (2 threads, ms) | 0.16 | 0.22 |
| P95 latency (2 threads, ms) | 0.38 | 0.69 |

### Disk I/O Results

| Workload | Metric | VM | Container |
|---|---|---|---|
| Sequential read | MiB/s | 461 | 500 |
| Sequential write | MiB/s | 358 | 291 |
| Random read | IOPS | 1,313 | 1,767 |
| Random write | IOPS | 1,331 | 1,346 |
| Sequential read | Avg latency (ms) | 2.16 | 1.99 |
| Sequential write | Avg latency (ms) | 2.78 | 3.42 |
| Random read | Avg latency (ms) | 0.75 | 0.56 |
| Random write | Avg latency (ms) | 0.74 | 0.73 |

### Network Results

| Metric | VM (loopback `127.0.0.1`) | Container (Docker bridge `172.17.0.1`) |
|---|---|---|
| Throughput, single stream, sender (Gbits/sec) | 14.1 | 13.7 |
| Throughput, single stream, receiver (Gbits/sec) | 14.1 | 10.3 |
| Data transferred (GBytes) | 49.3 | 47.9 |
| Throughput, 4 parallel streams | Not reported | Not reported |
| Retransmissions | 3 | 13 |

### Application (FastAPI) Results

| Endpoint | Metric | VM | Container |
|---|---|---|---|
| /health (c=100, n=10,000) | Requests/sec | 419.79 | 371.07 |
| /health | Time per request, mean (ms) | 238.21 | 269.49 |
| /health | Failed requests | 0 | 0 |
| /compute (c=10, n=1,000) | Requests/sec | 12.24 | 10.76 |
| /compute | Time per request, mean (ms) | 817.31 | 929.47 |
| /compute | Failed requests | 0 | 0 |
| /memory (c=10, n=1,000) | Requests/sec | 16.43 | 14.40 |
| /memory | Time per request, mean (ms) | 608.50 | 694.62 |
| /memory | Failed requests | 0 | 0 |

The /compute values above are from run 2. Run 1 gave 12.01 req/sec and 832.81 ms for the VM, and 10.60 req/sec and 943.33 ms for the container.

### Startup-Time Results

| Metric | VM | Container |
|---|---|---|
| Environment start (s) | Not reported | Not reported |
| Application ready (s) | Not reported | Not reported |

### Scalability Results

| Workload level | VM | Container |
|---|---|---|
| CPU 1 thread (events/sec) | 515.84 | 517.19 |
| CPU 2 threads (events/sec) | 883.55 | 894.38 |
| CPU 4 threads (events/sec) | 928.17 | 900.45 |
| CPU 8 threads (events/sec) | 905.17 | 914.42 |
| API wrk 10 / 50 / 100 / 200 connections | Not reported | Not reported |

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

```bash
pip install pandas matplotlib numpy jupyter
python3 scripts/analyze_results.py
python3 scripts/generate_plots.py
```
## CPU Scalability
<img width="4200" height="1500" alt="cpu_scalability" src="https://github.com/user-attachments/assets/d079a1f6-66bc-439f-b8c0-0712247c610b" />

## Memory Performance
<img width="3900" height="1500" alt="memory_performance" src="https://github.com/user-attachments/assets/87532019-0845-4dc1-8f23-f1e8c4bafab6" />

## Disk I/O Performance
<img width="4200" height="1650" alt="disk_io_performance" src="https://github.com/user-attachments/assets/a27bf039-1d2f-40e2-93a8-eddff4058432" />

## Network Performance
<img width="3900" height="1500" alt="network_performance" src="https://github.com/user-attachments/assets/f72bd26b-0798-4967-b1bd-afcd5c1efa82" />

## FastAPI Microservice Performance
<img width="4200" height="1560" alt="fastapi_performance" src="https://github.com/user-attachments/assets/38f9382b-bc2a-4326-a209-84422ef7a576" />


## VM vs Container Comparison

| Metric | VM | Container | Difference |
|---|---|---|---|
| CPU, 1 thread (events/sec) | 515.84 | 517.19 | Container +0.26% |
| CPU, 2 threads (events/sec) | 883.55 | 894.38 | Container +1.23% |
| CPU, 4 threads (events/sec) | 928.17 | 900.45 | VM +3.08% |
| CPU, 8 threads (events/sec) | 905.17 | 914.42 | Container +1.02% |
| Memory throughput, 1 thread (MiB/s) | 9,541.97 | 5,152.43 | Not reported |
| Memory throughput, 2 threads (MiB/s) | 9,880.38 | 6,970.16 | Not reported |
| Sequential Read (MiB/s) | 461 | 500 | Container +8.46% |
| Sequential Write (MiB/s) | 358 | 291 | VM +23.02% |
| Random Read (IOPS) | 1,313 | 1,767 | Container +34.58% |
| Random Write (IOPS) | 1,331 | 1,346 | Container +1.13% |
| Network throughput, sender (Gbits/sec) | 14.1 | 13.7 | VM +2.92% |
| Network throughput, receiver (Gbits/sec) | 14.1 | 10.3 | VM +36.89% |
| Startup Time (s) | Not reported | Not reported | Not reported |
| API Requests/sec, /health | 419.79 | 371.07 | VM +13.13% |
| API Requests/sec, /compute | 12.24 | 10.76 | VM +13.75% |
| API Requests/sec, /memory | 16.43 | 14.40 | VM +14.10% |
| API Latency, /health (ms) | 238.21 | 269.49 | Not reported |
| API Latency, /compute (ms) | 817.31 | 929.47 | Not reported |
| API Latency, /memory (ms) | 608.50 | 694.62 | Not reported |


## Conclusion

This benchmark evaluation provides an empirical and architectural comparison between Virtual Machines and Docker Containers across compute, memory, storage, networking, and microservice application tiers:

Compute Equivalence (Bare-Metal Instruction Execution):

Sysbench CPU benchmark results demonstrate 
<
1
 variance across 1, 2, 4, and 8 threads.
Because containers are native processes managed directly by the host Linux Completely Fair Scheduler (CFS), they avoid virtualization traps and binary translation overhead.
Storage I/O Performance (Direct VFS vs Hypervisor Driver):

Docker delivers +34.58% higher 4K random read IOPS (1,767 IOPS vs. 1,313 IOPS) and lower access latency (0.56 ms vs. 0.75 ms).
Containers interact directly with the Linux Virtual File System (VFS) cache, while Virtual Machines incur guest OS filesystem translation and virtual SCSI controller interrupt emulation.
Memory & Network Virtualization Overhead:

VM direct loopback achieves higher memory write bandwidth and lower network latency with only 3 TCP retransmissions vs 13 on Docker.
In containerized environments, packets traverse the docker0 bridge, veth pairs, and iptables NAT routing rules, resulting in a ~13–14% throughput overhead under high-concurrency HTTP load (FastAPI ApacheBench benchmarks).
Strategic Workload Recommendations:

Deploy Containers (Docker): When designing cloud-native microservices, horizontally scaling REST APIs, CI/CD runners, and applications demanding rapid elasticity, high deployment density, and maximum random I/O throughput.
Deploy Virtual Machines (KVM / VMware): When running untrusted multi-tenant workloads requiring hardware-enforced hypervisor security boundaries, heterogeneous OS kernels (Linux, Windows, BSD), or legacy enterprise monoliths.

## Final Project Structure
```
vm-vs-container-performance/
│
├── README.md                                  # Complete Experiment Documentation & Analysis
├── LAB_REPORT.md                              # Formal Academic Laboratory Report
├── .gitignore                                 # Git ignore configuration
│
├── api/                                       # FastAPI Microservice
│   ├── main.py                                # FastAPI application endpoints
│   ├── requirements.txt                       # Python dependencies
│   └── Dockerfile                             # FastAPI container image
│
├── docker/                                    # Benchmark Containerization Assets
│   └── Dockerfile                             # Docker benchmark environment
│
├── figures/                                   # Generated Analytical Visualizations
│   ├── overall_performance_dashboard.png      # Overall VM vs Docker dashboard
│   ├── cpu_scalability.png                    # CPU scalability comparison
│   ├── memory_performance.png                 # Memory performance comparison
│   ├── disk_io_performance.png                # Disk I/O comparison
│   ├── network_performance.png                # Network performance comparison
│   ├── fastapi_performance.png                # FastAPI performance comparison
│   └── graphs.py                               # Figure generation code
│
├── processed/                                 # Processed Benchmark Datasets
│   ├── api_results.csv                        # FastAPI benchmark results
│   ├── cpu_results.csv                        # CPU benchmark results
│   ├── disk_results.csv                      # Disk benchmark results
│   ├── memory_results.csv                    # Memory benchmark results
│   ├── network_results.csv                   # Network benchmark results
│   └── summary_comparison.csv                 # Overall comparison data
│
├── results/                                   # Raw Experimental Results
│   └── raw/
│       ├── baseline/                          # Baseline profiling results
│       ├── cpu/                               # CPU benchmark output logs
│       ├── memory/                            # Memory benchmark output logs
│       ├── disk/                              # Disk I/O benchmark output logs
│       ├── network/                           # Network benchmark output logs
│       └── api/                               # FastAPI benchmark output logs
│
├── screenshots/                               # Experimental Evidence
│   ├── 01_vm_baseline_profiling.jpeg
│   ├── 02_container_baseline_profiling.jpeg
│   ├── ...
│   └── 34_api_raw_results_directory_listing.jpeg
│
└── scripts/                                   # Automation & Analysis Scripts
    ├── run_cpu.sh                             # CPU benchmark automation
    ├── run_memory.sh                          # Memory benchmark automation
    ├── run_disk.sh                            # Disk benchmark automation
    ├── run_network.sh                         # Network benchmark automation
    ├── analyze_results.py                     # Benchmark result analysis
    └── generate_plots.py                      # Analytical plot generation
---
```

