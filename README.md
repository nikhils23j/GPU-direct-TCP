# GPUDirect TCP

A TCP/IP stack that runs on the GPU. Packets from a commodity Ethernet NIC are DMA'd straight into GPU memory, and a persistent CUDA kernel handles TCP processing there, so received data never passes through host RAM or the Linux TCP stack.

> **Status:** research prototype. It supports one TCP connection (client side), has been tested on one hardware configuration, and needs a patched NIC driver and a modified NVIDIA kernel-module build. It is not production software.

## Table of contents
- [Motivation](#motivation)
- [How it works](#how-it-works)
- [Results](#results)
- [Repository structure](#repository-structure)
- [Requirements](#requirements)
- [Setup](#setup)
- [Build](#build)
- [Running the benchmarks](#running-the-benchmarks)
- [API](#api)
- [Limitations](#limitations)
- [Troubleshooting](#troubleshooting)
- [Related work](#related-work)
- [Authors and license](#authors-and-license)

## Motivation

In GPU-centric workloads, network data normally takes this pat

```
NIC → host RAM → Linux TCP/IP stack (CPU) → cudaMemcpy → GPU
```

This costs CPU cycles, memory bandwidth, latency and cache spa the path:

```
NIC ──PCIe peer-to-peer DMA──► GPU memory ──► TCP processed by a GPU kernel
```

Goals:
1. Let a GPU receive data at high throughput with acceptable latency.
2. Keep CPU involvement minimal.
3. Leave the rest of the host's networking (other applications, ARP, other protocols) working normally.                                           
## How it works                                                                                                                                   
```                                                                                                                                                                      ┌────────────────────────── GPU ───────
  Wire ─► e1000e NIC ──► RX ring (descriptors)  ─► recv_warp ─► socket RX buffer ─► user kernel                                                             (patched)    │   PMEM: packet buffers      │ parse,
                       │                             ▼ doorbell                                                                                                          │   TX ring  ◄───────────  send_thread mit)
                       │   Redirect ring ◄─ non-TCP / unmatched packets                                                                                                  └──────────────┬───────────────────────
                                      │ relay thread (CPU)       │                                                                                                           raw socket TX / redirect       Inje Linux stack)
```                                                                                                                                               
**1. Patched `e1000e` driver.** The driver pins GPU memory with `nvidia_p2p_get_pages`, maps it for the NIC with `nvidia_p2p_dma_map_pages`, and  programs the GPU pages into the RX descriptors. For each receibyte `pkt_desc_t` into a GPU-resident ring. It exposes fourcharacter devices (`*_setup`, `*_redirect`, `*_inject`, `*_tx`).                                                                                  
**2. GPU-resident rings.** Single-producer/single-consumer rings with 4096 slots live in GPU memory:                                              
| Ring | Direction |                                                                                                                              |---|---|
| RX | NIC → GPU |                                                                                                                                | Redirect | GPU → CPU (packets the GPU stack won't handle, e.
| Inject | CPU → Linux stack (re-insert redirected packets) |                                                                                     | TX | GPU → NIC (packets the GPU wants to send) |
                                                                                                                                                  **3. Persistent CUDA kernel (`gdrtcp_main`).** It launches onc
- `recv_warp` (one warp): waits for packets, classifies them, copies payload in parallel, then lane 0 handles reordering, cumulative ACKs and the doorbell.
- `send_thread`: builds SYN, FIN, ACK and data packets, runs the retransmit queue and RTO.
- A second block runs the hardcoded benchmark client (a workarel launch issue).

**4. Userspace relay thread.** The NVIDIA driver can't be moditles TX packets (via a raw socket) and redirect/inject packetswith `cudaMemcpyAsync`. This is the main remaining source of CPU use.

**5. Zero-copy API.** GPU code reads and writes socket buffers through slices (see [API](#api)).

Memory regions:

| Name | Contents |
|---|---|
| PMEM | Packet buffers (4096 × 2 KB) |
| CMEM | Control page and the RX, Redirect and TX rings |
| IMEM | Kernel `vmalloc` region mapped to userspace, used for the Inject ring and packets |

## Results

Measured on 1 GbE (hardware below). Line rate for TCP goodput is 1460/1538 × 1 Gbps ≈ **949 Mbps**.

| Benchmark | Setup | Throughput | Total CPU | Notes |
|---|---|---|---|---|
| 0 | CPU TCP client (`bench_native`) | line rate | 25% of a core | baseline |
| 1 | GPU TCP stack (`bench_gdrtcp 0`) | line rate | 33% of a e relay, 3% the GPU interrupt handler, `ksoftirqd` ≈ 0% |
| 2 | GPU → CPU packet redirect (`bench_gdrtcp 1` + `bench_redirect`) | line rate | ~102% | RTT rises to 6–7 ms because of two extra copies |

- Correctness: randomized data transferred with a position-weighted checksum verified by the server (32 GB in correctness tests; 4 GB per benchmark run
by default).
- CPU use is **not** zero. Kernel TCP/IP processing (`ksoftirqd`) is removed from the data path, but the relay thread still uses CPU. See the report
for the discussion.

Raw data is in `gdrtcp/data/exp{0,1,2}/data.csv`, with plots i

## Repository structure

```
.
├── README.md
├── e1000e.patch                 # patch against the Linux 6.1.4 e1000e driver
├── e1000e/                      # the three files to add to t
│   ├── netdev.c                 #   modified driver core (replaces the original)
│   ├── gdr_common.h             #   structs/constants shared
│   └── nv-p2p.h                 #   NVIDIA GPUDirect RDMA kernel interface
├── e1000e_complete/             # full modified e1000e sourceile)
├── gdrtcp/
│   ├── CMakeLists.txt
│   ├── main/                    # the library
│   │   ├── gdrtcp.h             #   public API, structs, cons
│   │   ├── gdrtcp.cu            #   relay thread, GPU kernel, init/destroy, socket API
│   │   ├── gdr_common.h         #   rings, descriptors, memorr)
│   │   ├── helpers.cuh          #   CUDA/driver helper macros and utilities
│   │   └── bench_gdrtcp.cu      #   Benchmark 1/2 client
│   ├── benchmarks/
│   │   ├── bench_server.c       #   server: sends random datas CSV
│   │   ├── bench_native.c       #   Benchmark 0 client (CPU sockets)
│   │   ├── bench_redirect.c     #   Benchmark 2 client (CPU s
│   │   └── bench_host.cu        #   host-side benchmark helper
│   └── data/
│       ├── exp0/ exp1/ exp2/    #   captured results (data.csv)
│       ├── output/              #   generated plots
│       ├── make_plots.sh
│       ├── plot_tp_rtt_var.py   #   throughput and RTT varian
│       ├── plot_rtt_cwnd.py     #   RTT and cwnd plots
│       └── cpu_plot.py          #   CPU usage plots
└── scripts/
    ├── build_e1000e.sh          # builds the out-of-tree e100
    └── shift_eno2_to_namespace.sh  # moves the second NIC into its own network namespace
```

## Requirements

**Tested hardware**

| Component | Model |
|---|---|
| NIC 1 (GPUDirect TCP) | Intel I219-LM (`e1000e`) |
| NIC 2 (management and test server) | Intel I210 (`igb`), must **not** use `e1000e` |
| GPU | NVIDIA Quadro P620 |

The machine needs two NICs: one for the patched driver, one tohile NIC 1 resets. Bare metal only, since VMs were not tested.

**Software**

- Linux kernel **6.1.4**, with *Intel(R) PRO/1000 PCI-Express set to `<M>`
- NVIDIA driver **535.54.03**, built manually (proprietary modules, needed for Pascal)
- CUDA Toolkit **12.2**
- CMake, GCC/G++, `iperf3` (optional), Python 3 with matplotlib/pandas for plots

**Kernel command line** (`/etc/default/grub`, then `sudo update-grub` and reboot):

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_pstate=no_hwp io_iommu_type1.allow_unsafe_interrupts=1"
GRUB_CMDLINE_LINUX="panic=5"
```

## Setup

### 1. Build NVIDIA modules manually

The e1000e module links against `nvidia.ko` and uses GPL-only NVIDIA modules are built from source with their licence stringchanged to GPL. This misstates the licence of a proprietary module. It is acceptable only for research on your own machine.

```bash
./NVIDIA-Linux-x86_64-535.54.03.run -x
cd NVIDIA-Linux-x86_64-535.54.03/kernel
# change MODULE_LICENSE to "GPL" in:
#   nvidia-modeset/nvidia-modeset-linux.c
#   nvidia-peermem/nvidia-peermem.c
#   nvidia-uvm/uvm.c
#   nvidia-drm/nvidia-drm-linux.c
#   nvidia/nv-frontend.c
make -j$(nproc)
cd ..
sudo ./NVIDIA-Linux-x86_64-535.54.03.run --no-kernel-modules

cd kernel
sudo cp nvidia*.ko /lib/modules/6.1.4/extra/
sudo depmod -a
sudo modprobe nvidia && sudo modprobe nvidia-uvm

sudo ./cuda_12.2.0_535.54.03_linux.run --toolkit
export PATH=/usr/local/cuda-12.2/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.2/lib64:$LD_LIBRARY_PATH
```

Check with `nvidia-smi` and a sample CUDA kernel.

### 2. Build and load the patched e1000e

Either apply `e1000e.patch` to the kernel source, or copy `e10,nv-p2p.h}` into`linux-6.1.4/drivers/net/ethernet/intel/e1000e/`.

If NIC 1 is not `eno1`, edit the `DEVNAME*` defines in `netdev.c`:

```c
#define DEVNAME          "eno1_setup"
#define DEVNAME_REDIRECT "eno1_redirect"
#define DEVNAME_INJECT   "eno1_inject"
#define DEVNAME_TX       "eno1_tx"
```

Then, from the kernel source root (adjust the `Module.symvers`

```bash
cp scripts/build_e1000e.sh linux-6.1.4/ && cd linux-6.1.4
./build_e1000e.sh
# internally:
# make KBUILD_EXTRA_SYMBOLS=~/nvidia_driver/NVIDIA-Linux-x86_6symvers \
#      M=./drivers/net/ethernet/intel/e1000e/ modules

# do this over SSH on NIC 2, because NIC 1 goes down
sudo modprobe -r e1000e
sudo insmod drivers/net/ethernet/intel/e1000e/e1000e.ko
```

With GPUDirect TCP disabled the driver behaves like the stock en userspace writes to the setup character device.

### 3. Separate the NICs into network namespaces

This forces test traffic onto the wire instead of being short-dit `scripts/shift_eno2_to_namespace.sh` for your interfacenames, IPs and gateway, run it over SSH on NIC 1, then reconnect through NIC 2. If SSH fails, re-run `sudo ip netns exec ns2 /usr/sbin/sshd` and wait
about 30 seconds. Confirm line rate between the NICs with `ipe

## Build

Set these for your machine first:

| What | Where |
|---|---|
| Interface name (default `eno1`) | `gdrtcp/main/gdrtcp.cu` (`, `bench_gdrtcp.cu` |
| `#define SERVER_IP` (use NIC 2's IP) | `gdrtcp/main/bench_gdrtcp.cu`, `gdrtcp/benchmarks/{bench_host.cu, bench_native.c, bench_redirect.c}` |
| Test size (default 4 GB) | `test_size` in `bench_gdrtcp.cu` e.c` (line 24) |

```bash
cd gdrtcp
mkdir build && cd build
cmake ..
make
```

## Running the benchmarks

All commands run over SSH through **NIC 2**. NIC 1 resets when GPUDirect TCP is enabled or disabled, which causes a few seconds of downtime.

```bash
alias ins1='sudo nsenter -t 1 -n'     # run a command in NIC 1
```

In every benchmark, start the server first from `gdrtcp/build/benchmarks`:

```bash
./bench_server
```

| Benchmark | Command(s) | Measures |
|---|---|---|
| 0: native CPU baseline | `ins1 ./bench_native` | Standard Linux TCP |
| 1: GPU TCP | `ins1 ./bench_gdrtcp 0` | GPU-resident TCP; autDirect TCP |
| 2: GPU → CPU redirect | `ins1 ./bench_gdrtcp 1`, then in another terminal `ins1 ./bench_redirect` | Ordinary CPU sockets while the GPU stack is
enabled. Press Ctrl+C on `bench_gdrtcp` to finish. |

Each run takes about 40 seconds at 4 GB.

**Verifying correctness:** check the end of the *server's* out

```
Checksum verified! value = <number>
```

The client may print occasional `TCP Checksum Mismatch` or `IP Those are discussed in the report (section IV.A).

**Plots:** the server writes a `data.csv` in its working direch the scripts in `gdrtcp/data/` (`make_plots.sh`,`plot_tp_rtt_var.py`, `plot_rtt_cwnd.py`, `cpu_plot.py`). The captured CSVs and plots from the report are included.

## API

Declared in `gdrtcp/main/gdrtcp.h` (namespace `gdrtcp`).

Host side:

```cpp
error_t init(handle_t* handle, const char* network_interface,
             bool hack_launch_workload, struct sockaddr_in hack_server_addr, size_t hack_test_size);
error_t destroy(handle_t* handle);
```

Device side (called from GPU code):

```cpp
__device__ socket_t* socket_create(handle_t*, int domain, int
__device__ error_t   socket_connect(handle_t*, socket_t*, struct sockaddr_in addr);
__device__ error_t   socket_close(handle_t*, socket_t*);

// Zero-copy receive: min_len/max_len must be powers of two
__device__ slice_t recv_slice(socket_t*, int min_len, int max_len);
__device__ error_t recv_release(socket_t*, slice_t);   // releend of the slice

// Zero-copy send
__device__ slice_t send_reserve(socket_t*, int min_len, int max_len);
__device__ error_t send_complete(socket_t*, slice_t);  // send of the slice
```

Slice lengths are powers of two because the socket buffers are ring buffers. Slice pointers refer to GPU global memory only.

`init` currently takes `hack_*` parameters because the benchmark client kernel is hardcoded into the library (a workaround for running a second kernel
concurrently with the persistent TCP kernel). A general user-kpported.

## Limitations

- **One connection** (`CONNECTION_TABLE_SZ = 1`) and **client ept, no TIME_WAIT).
- Cumulative ACKs only, with a fixed 2 s RTO. There is no SACK, and the GPU sender has no congestion control.
- MSS is fixed at 1460 advertised (512-byte internal segment acovery. The ISN is a constant.
- Non-TCP traffic (ARP etc.) is redirected to the Linux stack through the CPU relay.
- A userspace relay thread is required, using about 30% of a cs because the NVIDIA driver can't be modified.
- Interface names and IP addresses are hardcoded in several source files.
- Requires Pascal-era proprietary NVIDIA modules rebuilt with t can't be redistributed as a binary.
- Occasional packets arrive with invalid checksums after being copied from GPU memory (about 10 per run in the TX path, a few per GB on the RX path).
This is related to PCIe/GPU memory ordering and is handled withecks.
- Tested only at 1 Gbps on one machine and NIC model.

## Troubleshooting

| Symptom | Likely cause or fix |
|---|---|
| `Is the modified NIC driver loaded on interface …?` | The `*_setup` device is missing: check `insmod` succeeded and `DEVNAME*` matches your
interface. |
| SSH drops when enabling or disabling | Expected. NIC 1 resets for a few seconds; always connect through NIC 2. |
| `insmod` fails with unknown symbols | `KBUILD_EXTRA_SYMBOLS`modules were not rebuilt with GPL licences. |
| `nvidia_p2p_get_pages` fails | Memory not 64 KB-aligned, or wrong/open NVIDIA module (Pascal needs the proprietary one). |
| Throughput well below 949 Mbps | Check line rate with `iperf IOMMU passthrough flags in the kernel command line. |
| Checksum not verified on the server | Look at `dmesg` and the client output for buffer-overflow messages. |

## Related work

- GPUnet: Networking Abstractions for GPU Programs (OSDI '14)
- GPUrdma: GPU-side library for high-performance networking fr
- NVIDIA GPUDirect RDMA, GPUDirect Storage, DOCA GPUNetIO, NVSHMEM
- DPDK, AF_XDP, RDMA verbs

## Authors and license

- Course project, group **"Hello123"**. Add each contributor's
- License: **TBD**. Note that the Linux kernel parts are GPL-2.0 (the e1000e files carry their own GPL headers), and the NVIDIA headers (`nv-p2p.h`) have their own licence terms. Check these before choosing a li
- The accompanying report is `project_report.pdf`.
