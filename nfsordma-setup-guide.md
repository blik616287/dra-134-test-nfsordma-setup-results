# NFS over RDMA (NFSoRDMA) Setup Guide

This guide documents the complete setup of an NFS server with NFS over RDMA (RoCEv2) support on NVIDIA GPU nodes with ConnectX-7 NICs, accessible by Kubernetes pods.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Server Setup (gpu-sm01)](#server-setup-gpu-sm01)
- [Client Setup (gpu-sm02)](#client-setup-gpu-sm02)
- [Kubernetes Integration](#kubernetes-integration)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Performance Results](#performance-results)

---

## Overview

NFSoRDMA provides high-performance NFS access by leveraging RDMA (Remote Direct Memory Access) over RoCEv2 (RDMA over Converged Ethernet v2). This eliminates CPU overhead and reduces latency compared to traditional TCP-based NFS.

### Key Benefits

- **Lower latency**: Bypasses kernel network stack
- **Higher throughput**: Direct memory-to-memory transfers
- **Reduced CPU usage**: Offloads data transfer to NIC hardware

---

## Prerequisites

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| NICs | NVIDIA ConnectX-7 (BlueField-3) or ConnectX-6/7 |
| Link Speed | 40Gbps+ recommended |
| RDMA Support | RoCEv2 capable |

### Software Requirements

| Component | Version |
|-----------|---------|
| OS | Ubuntu 22.04 LTS |
| Kernel | 6.8.0-94-generic (or compatible) |
| DOCA | 3.2.1 |
| MLNX OFED | 25.10 |

### Verify RDMA Hardware

```bash
# Check RDMA devices
rdma link show

# Expected output:
# link mlx5_0/1 state ACTIVE physical_state LINK_UP netdev enp14s0f0np0
# link mlx5_1/1 state ACTIVE physical_state LINK_UP netdev enp14s0f1np1

# Check kernel RDMA support
cat /boot/config-$(uname -r) | grep CONFIG_SUNRPC_XPRT_RDMA
# Expected: CONFIG_SUNRPC_XPRT_RDMA=m
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RDMA Network (10.0.100.0/24)                 │
│                                                                     │
│  ┌──────────────────────┐              ┌──────────────────────┐    │
│  │     gpu-sm01         │              │      gpu-sm02        │    │
│  │   (NFS Server)       │              │    (NFS Client)      │    │
│  │                      │              │                      │    │
│  │  enp14s0f0np0        │    RDMA      │  enp14s0f0np0        │    │
│  │  10.0.100.1/24  ─────┼──────────────┼─► 10.0.100.2/24      │    │
│  │                      │   40Gbps     │                      │    │
│  │  NFS Export:         │              │  Test Pods with      │    │
│  │  /srv/nfs/shared     │              │  hostNetwork: true   │    │
│  │                      │              │                      │    │
│  │  RDMA Port: 20049    │              │                      │    │
│  │  TCP Port: 2049      │              │                      │    │
│  └──────────────────────┘              └──────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    Node Network (172.16.32.0/24)                    │
│                                                                     │
│  gpu-sm01: 172.16.32.42 (control-plane)                            │
│  gpu-sm02: 172.16.32.45 (worker)                                   │
│                                                                     │
│  Kubernetes pods access NFS via:                                    │
│  - TCP: 172.16.32.42:2049 (standard)                               │
│  - RDMA: 10.0.100.1:20049 (high-performance, requires hostNetwork) │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Server Setup (gpu-sm01)

### Step 1: Install NFS Server

```bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
```

### Step 2: Configure RDMA Interface

```bash
# Assign IP to RDMA interface
sudo ip addr add 10.0.100.1/24 dev enp14s0f0np0
sudo ip link set enp14s0f0np0 up

# Verify
ip addr show enp14s0f0np0
```

#### Make RDMA IP Persistent

Create `/etc/netplan/60-rdma.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp14s0f0np0:
      addresses:
        - 10.0.100.1/24
```

Apply:

```bash
sudo netplan apply
```

### Step 3: Create NFS Export Directory

```bash
sudo mkdir -p /srv/nfs/shared
sudo chmod 777 /srv/nfs/shared
sudo chown nobody:nogroup /srv/nfs/shared
```

### Step 4: Configure NFS Exports

Edit `/etc/exports`:

```bash
# Pod network, Node network, and RDMA network
/srv/nfs/shared  10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash) 172.16.32.0/24(rw,sync,no_subtree_check,no_root_squash) 10.0.100.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Apply exports:

```bash
sudo exportfs -ra
sudo exportfs -v
```

### Step 5: Install NFSoRDMA Kernel Module

The stock kernel's `rpcrdma` module is incompatible with DOCA/MLNX OFED drivers due to symbol version mismatches. Install the compatible DKMS module:

```bash
# Fix any broken packages first
sudo apt --fix-broken install -y

# Install mlnx-nfsrdma-dkms (builds rpcrdma against OFED headers)
sudo apt-get install -y mlnx-nfsrdma-dkms
```

This installs three modules:
- `rpcrdma.ko` - Main RPC/RDMA transport
- `svcrdma.ko` - Server-side RDMA
- `xprtrdma.ko` - Client-side RDMA transport

### Step 6: Load and Enable RDMA Module

```bash
# Load the module
sudo modprobe rpcrdma

# Verify
lsmod | grep rpcrdma

# Make persistent
echo "rpcrdma" | sudo tee /etc/modules-load.d/rpcrdma.conf
```

### Step 7: Start NFS Server and Enable RDMA Transport

```bash
# Enable and start NFS server
sudo systemctl enable --now nfs-kernel-server

# Enable RDMA on port 20049
echo "rdma 20049" | sudo tee /proc/fs/nfsd/portlist

# Verify
cat /proc/fs/nfsd/portlist
# Expected output should include: rdma 20049
```

### Step 8: Make RDMA Transport Persistent

Create `/etc/systemd/system/nfs-rdma.service`:

```ini
[Unit]
Description=Enable NFS over RDMA
After=nfs-server.service
Requires=nfs-server.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo "rdma 20049" > /proc/fs/nfsd/portlist'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable nfs-rdma.service
```

### Step 9: Verify Server Setup

```bash
# Check NFS server status
systemctl status nfs-server

# Check RDMA port is enabled
cat /proc/fs/nfsd/portlist

# Check exports
showmount -e localhost

# Check RDMA link status
rdma link show
```

---

## Client Setup (gpu-sm02)

### Step 1: Configure RDMA Interface

```bash
# Assign IP to RDMA interface
sudo ip addr add 10.0.100.2/24 dev enp14s0f0np0
sudo ip link set enp14s0f0np0 up

# Test connectivity to server
ping -c 2 10.0.100.1
```

#### Make RDMA IP Persistent

Create `/etc/netplan/60-rdma.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp14s0f0np0:
      addresses:
        - 10.0.100.2/24
```

### Step 2: Install NFSoRDMA Client Module

```bash
# Fix any broken packages first
sudo apt --fix-broken install -y

# Install mlnx-nfsrdma-dkms
sudo apt-get install -y mlnx-nfsrdma-dkms

# Load client module
sudo modprobe xprtrdma

# Make persistent
echo "xprtrdma" | sudo tee /etc/modules-load.d/xprtrdma.conf
```

### Step 3: Test RDMA Mount

```bash
# Create mount point
sudo mkdir -p /mnt/nfs-rdma

# Mount with RDMA
sudo mount -t nfs -o rdma,port=20049,vers=4.1 10.0.100.1:/srv/nfs/shared /mnt/nfs-rdma

# Verify mount
mount | grep rdma
# Expected: proto=rdma,port=20049
```

### Step 4: Test Performance

```bash
# Write test (1GB)
dd if=/dev/zero of=/mnt/nfs-rdma/testfile bs=1M count=1024 conv=fdatasync

# Read test (1GB)
dd if=/mnt/nfs-rdma/testfile of=/dev/null bs=1M

# Cleanup
rm /mnt/nfs-rdma/testfile
```

---

## Kubernetes Integration

### PersistentVolume with RDMA Mount Options

Kubernetes PVs support `mountOptions` which kubelet passes directly to the `mount` command. This allows NFSoRDMA access using standard PV/PVC without privileged pods or hostNetwork.

**Key Requirements:**
- Use the RDMA network IP (`10.0.100.1`) not the node IP
- Worker nodes must have `xprtrdma` module loaded
- Worker nodes must have RDMA network connectivity to the NFS server

Create `nfs-rdma-pv-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-rdma-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - proto=rdma
    - port=20049
    - vers=4.1
    - rsize=1048576
    - wsize=1048576
  nfs:
    server: 10.0.100.1    # RDMA network IP (not node IP)
    path: /srv/nfs/shared
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-rdma-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 100Gi
  volumeName: nfs-rdma-pv
```

Apply:

```bash
kubectl apply -f nfs-rdma-pv-pvc.yaml
kubectl get pv,pvc
```

### How It Works

```
Pod requests PVC
    ↓
Kubelet mounts NFS using mountOptions from PV
    ↓
mount -t nfs -o proto=rdma,port=20049,vers=4.1 10.0.100.1:/srv/nfs/shared /var/lib/kubelet/pods/...
    ↓
xprtrdma module handles RDMA transport
    ↓
Data flows over RDMA network (10.0.100.x) at up to 5 GB/s
```

---

## Testing

### NFSoRDMA Test Pod with PVC

Create `nfs-rdma-test-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-rdma-pvc-test
  namespace: default
spec:
  containers:
  - name: test
    image: ubuntu:22.04
    command: ["bash", "-c"]
    args:
    - |
      echo "=== Mount Info ==="
      mount | grep nfs
      echo ""
      echo "=== Write Test (512MB) ==="
      dd if=/dev/zero of=/data/testfile bs=1M count=512 conv=fdatasync 2>&1
      echo ""
      echo "=== Read Test (512MB) ==="
      dd if=/data/testfile of=/dev/null bs=1M 2>&1
      echo ""
      rm -f /data/testfile
      echo "Tests complete, sleeping..."
      sleep infinity
    volumeMounts:
    - name: nfs-data
      mountPath: /data
  volumes:
  - name: nfs-data
    persistentVolumeClaim:
      claimName: nfs-rdma-pvc
  nodeSelector:
    kubernetes.io/hostname: gpu-sm02  # Target worker node with RDMA
  restartPolicy: Never
```

Deploy and check results:

```bash
kubectl apply -f nfs-rdma-test-pod.yaml
kubectl logs nfs-rdma-pvc-test
```

### Expected Output

```
=== Mount Info ===
10.0.100.1:/srv/nfs/shared on /data type nfs4 (rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,proto=rdma,port=20049,timeo=600,retrans=2,sec=sys,clientaddr=10.0.100.2,local_lock=none,addr=10.0.100.1)

=== Write Test (512MB) ===
512+0 records in
512+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 2.45988 s, 218 MB/s

=== Read Test (512MB) ===
512+0 records in
512+0 records out
536870912 bytes (537 MB, 512 MiB) copied, 0.107731 s, 5.0 GB/s

Tests complete, sleeping...
```

**Key Verification:** The mount output shows `proto=rdma,port=20049` confirming RDMA transport is active.

### Cleanup Test Pod

```bash
kubectl delete pod nfs-rdma-pvc-test
```

### Advantages of PV/PVC Approach

| Feature | PV/PVC with mountOptions | hostNetwork + Manual Mount |
|---------|--------------------------|---------------------------|
| Privileged required | No | Yes |
| hostNetwork required | No | Yes |
| Standard K8s workflow | Yes | No |
| Pod security compliant | Yes | No |
| Mount managed by | Kubelet | Container |
| Persistence | Automatic | Manual |

---

## Troubleshooting

### Module Load Failures

**Error**: `modprobe: ERROR: could not insert 'rpcrdma': Invalid argument`

**Cause**: Symbol version mismatch between kernel rpcrdma and DOCA/OFED RDMA stack.

**Solution**: Install `mlnx-nfsrdma-dkms` package which compiles rpcrdma against OFED headers.

```bash
# Check kernel logs for details
sudo dmesg | grep -i rdma

# Install compatible module
sudo apt-get install -y mlnx-nfsrdma-dkms
```

### RDMA Mount Fails

**Error**: `mount.nfs: Connection timed out`

**Checklist**:

1. Verify RDMA interface has IP and is UP:
   ```bash
   ip addr show enp14s0f0np0
   ```

2. Verify RDMA connectivity:
   ```bash
   ping 10.0.100.1
   ```

3. Verify server has RDMA port enabled:
   ```bash
   cat /proc/fs/nfsd/portlist  # Should show "rdma 20049"
   ```

4. Verify rpcrdma module is loaded on both server and client:
   ```bash
   lsmod | grep rpcrdma
   ```

### NFS Server Not Exporting

```bash
# Check exports
sudo exportfs -v

# Re-export
sudo exportfs -ra

# Check NFS server status
systemctl status nfs-server
```

### Pod Cannot Mount with RDMA

**Issue**: Pod mount doesn't show `proto=rdma` in mount output.

**Checklist**:

1. **Verify PV uses RDMA mount options:**
   ```yaml
   mountOptions:
     - proto=rdma
     - port=20049
     - vers=4.1
   ```

2. **Verify PV uses RDMA network IP (not node IP):**
   ```yaml
   nfs:
     server: 10.0.100.1    # Correct: RDMA network
     # server: 172.16.32.42  # Wrong: Node network
   ```

3. **Verify xprtrdma module is loaded on worker:**
   ```bash
   lsmod | grep xprtrdma
   ```

4. **Verify worker has RDMA network connectivity:**
   ```bash
   ping 10.0.100.1
   ```

5. **Check kubelet logs for mount errors:**
   ```bash
   journalctl -u kubelet | grep -i mount
   ```

---

## Performance Results

### Test Environment

| Component | Specification |
|-----------|---------------|
| Server | gpu-sm01 (172.16.32.42 / 10.0.100.1) |
| Client | gpu-sm02 (172.16.32.45 / 10.0.100.2) |
| NICs | NVIDIA ConnectX-7 (BlueField-3) |
| Link Speed | 40 Gbps (5 GB/s theoretical) |
| NFS Version | 4.1 |
| RDMA Port | 20049 |

### Mount Options Used

```
proto=rdma,port=20049,vers=4.1,rsize=1048576,wsize=1048576
```

### Quick Validation Results (dd)

| Test | Throughput |
|------|------------|
| Sequential Write (1GB) | 292 MB/s |
| Sequential Read (1GB, cached) | 5.6 GB/s |

### Comprehensive Benchmark Results (fio)

Benchmarks performed using `fio` with `ioengine=libaio` and `direct=1` (O_DIRECT) to bypass page cache.

#### Sequential Write Performance

| Test Configuration | Block Size | Jobs | Total I/O | Throughput | IOPS |
|-------------------|------------|------|-----------|------------|------|
| Single Stream | 1M | 1 | 4 GB | **159 MiB/s (167 MB/s)** | 159 |
| 4 Parallel Streams | 1M | 4 | 8 GB | **358 MiB/s (375 MB/s)** | 358 |
| 8 Parallel Streams | 1M | 8 | 8 GB | **382 MiB/s (401 MB/s)** | 382 |
| Large Blocks | 4M | 4 | 8 GB | **367 MiB/s (385 MB/s)** | 91 |

#### Sequential Read Performance

| Test Configuration | Block Size | Jobs | Total I/O | Throughput | IOPS |
|-------------------|------------|------|-----------|------------|------|
| 4 Parallel Streams | 1M | 4 | 8 GB | **4,312 MiB/s (4.5 GB/s)** | 4,311 |

#### Latency Analysis

| Operation | Average Latency | P99 Latency |
|-----------|----------------|-------------|
| Sequential Write (1M, 8 jobs) | 20.9 ms | 24.5 ms |
| Sequential Read (1M, 4 jobs) | 922 µs | 1.07 ms |

### Benchmark Pod

For comprehensive benchmarking, use this pod specification:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-rdma-benchmark
  namespace: nvidia-network-operator
spec:
  hostNetwork: true
  containers:
  - name: benchmark
    image: ubuntu:22.04
    command: ["bash", "-c"]
    args:
    - |
      apt-get update -qq && apt-get install -y -qq nfs-common fio
      mkdir -p /mnt/nfs-rdma
      mount -t nfs -o rdma,port=20049,vers=4.1 10.0.100.1:/srv/nfs/shared /mnt/nfs-rdma

      echo "=== Sequential Write - 8 parallel streams ==="
      fio --name=seq-write --ioengine=libaio --direct=1 --bs=1M --size=1G \
          --numjobs=8 --rw=write --group_reporting --directory=/mnt/nfs-rdma

      echo "=== Sequential Read - 4 parallel streams ==="
      dd if=/dev/zero of=/mnt/nfs-rdma/readtest.dat bs=1M count=8192 2>/dev/null
      sync && echo 3 > /proc/sys/vm/drop_caches 2>/dev/null || true
      fio --name=seq-read --ioengine=libaio --direct=1 --bs=1M --size=2G \
          --numjobs=4 --rw=read --group_reporting --filename=/mnt/nfs-rdma/readtest.dat

      rm -f /mnt/nfs-rdma/*.dat
      sleep infinity
    securityContext:
      privileged: true
    volumeMounts:
    - name: modules
      mountPath: /lib/modules
  volumes:
  - name: modules
    hostPath:
      path: /lib/modules
  nodeSelector:
    kubernetes.io/hostname: gpu-sm02
  restartPolicy: Never
```

---

## Bottleneck Analysis

### Observed Performance vs Theoretical Maximum

| Metric | Observed | Theoretical Max | Utilization |
|--------|----------|-----------------|-------------|
| Write Throughput | 401 MB/s | 5,000 MB/s (40Gbps) | **8%** |
| Read Throughput | 4,500 MB/s | 5,000 MB/s (40Gbps) | **90%** |

### Identified Bottlenecks

#### 1. Write Performance: Server-Side Storage I/O

**Symptom**: Write throughput plateaus at ~400 MB/s regardless of parallelism or block size.

**Evidence**:
- Single stream: 167 MB/s
- 4 streams: 375 MB/s
- 8 streams: 401 MB/s (minimal gain from 4→8)
- 4M blocks: 385 MB/s (no improvement over 1M blocks)

**Root Cause**: The NFS server's underlying storage subsystem (likely local disk or storage controller) is the limiting factor, not the RDMA network.

**Verification**:
```bash
# On NFS server, check disk I/O during write test
iostat -x 1

# Check if storage is the bottleneck
# Look for high %util on the storage device
```

**Solutions**:
- Use faster storage (NVMe, RAID array, or distributed storage)
- Enable write-back caching (with battery backup)
- Use RAM-backed storage for testing (tmpfs)

#### 2. Read Performance: Near Line-Rate

**Observation**: Read performance reaches 4.5 GB/s, which is ~90% of the 40Gbps theoretical maximum.

**Analysis**: This indicates:
- RDMA transport is functioning correctly
- Data is being served from NFS server's page cache
- Network is not a bottleneck for cached reads

#### 3. Write Latency Analysis

**Observed**: 20-21ms average write latency with direct I/O

**Breakdown**:
- RDMA network latency: <1ms (based on read latency)
- Storage commit latency: ~20ms (dominant factor)

This confirms the storage subsystem is the bottleneck for writes.

### Performance Optimization Recommendations

#### For Higher Write Throughput

1. **Upgrade Server Storage**:
   ```bash
   # Check current storage performance
   fio --name=local-write --ioengine=libaio --direct=1 --bs=1M \
       --size=4G --numjobs=4 --rw=write --directory=/srv/nfs/shared
   ```

2. **Use Async Export Option** (reduces sync overhead):
   ```bash
   # In /etc/exports (use with caution - data loss risk on crash)
   /srv/nfs/shared  10.0.100.0/24(rw,async,no_subtree_check,no_root_squash)
   ```

3. **Tune NFS Server Threads**:
   ```bash
   # Increase NFS server threads (default is 8)
   echo 32 | sudo tee /proc/fs/nfsd/threads
   ```

#### For Maximum Read Throughput

1. **Increase Read-Ahead**:
   ```bash
   # On client, increase NFS read-ahead
   echo 16384 > /sys/class/bdi/0:*/read_ahead_kb
   ```

2. **Use Larger rsize**:
   ```bash
   mount -t nfs -o rdma,port=20049,vers=4.1,rsize=1048576 ...
   ```

### Performance Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NFSoRDMA Performance Profile                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  WRITE PATH:                                                        │
│  ┌─────────┐    ┌──────────┐    ┌─────────────┐    ┌────────────┐  │
│  │ Client  │───►│   RDMA   │───►│ NFS Server  │───►│  Storage   │  │
│  │ Pod     │    │ Network  │    │ (nfsd)      │    │  (DISK)    │  │
│  └─────────┘    └──────────┘    └─────────────┘    └────────────┘  │
│       │              │                │                  │          │
│   Unlimited      5 GB/s           Unlimited          ~400 MB/s     │
│                                                    ▲ BOTTLENECK    │
│                                                                     │
│  READ PATH (cached):                                                │
│  ┌─────────┐    ┌──────────┐    ┌─────────────┐                    │
│  │ Client  │◄───│   RDMA   │◄───│ NFS Server  │                    │
│  │ Pod     │    │ Network  │    │ (page cache)│                    │
│  └─────────┘    └──────────┘    └─────────────┘                    │
│       │              │                │                             │
│   Unlimited      5 GB/s          Memory BW                         │
│                ▲ ~90% utilized                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Conclusion

The NFSoRDMA setup is functioning correctly. The RDMA network transport is not the bottleneck:

- **Read performance** achieves near line-rate (4.5 GB/s on 40Gbps link)
- **Write performance** is limited by server-side storage I/O (~400 MB/s)

To improve write performance, upgrade the NFS server's storage subsystem (faster disks, NVMe, or distributed storage). The RDMA network has significant headroom for higher write throughput once storage is upgraded.

---

## Files Reference

### Server (gpu-sm01)

| File | Purpose |
|------|---------|
| `/etc/exports` | NFS export configuration |
| `/etc/netplan/60-rdma.yaml` | RDMA interface IP configuration |
| `/etc/modules-load.d/rpcrdma.conf` | Load rpcrdma module on boot |
| `/etc/systemd/system/nfs-rdma.service` | Enable RDMA transport on boot |

### Client (gpu-sm02)

| File | Purpose |
|------|---------|
| `/etc/netplan/60-rdma.yaml` | RDMA interface IP configuration |
| `/etc/modules-load.d/xprtrdma.conf` | Load xprtrdma module on boot |

---

## Quick Reference Commands

### Server Commands

```bash
# Check NFS server
systemctl status nfs-server
showmount -e localhost
cat /proc/fs/nfsd/portlist

# Check RDMA
rdma link show
lsmod | grep rpcrdma

# Re-enable RDMA port (if lost after restart)
echo "rdma 20049" | sudo tee /proc/fs/nfsd/portlist
```

### Client Commands

```bash
# Mount with RDMA
sudo mount -t nfs -o rdma,port=20049,vers=4.1 10.0.100.1:/srv/nfs/shared /mnt/nfs-rdma

# Verify RDMA mount
mount | grep rdma

# Unmount
sudo umount /mnt/nfs-rdma
```

### Kubernetes Commands

```bash
# Deploy test pod
kubectl apply -f nfs-rdma-test-pod.yaml

# Check logs
kubectl logs -n nvidia-network-operator nfs-rdma-test

# Cleanup
kubectl delete pod -n nvidia-network-operator nfs-rdma-test
```

---

## References

- [NVIDIA DOCA Documentation](https://docs.nvidia.com/doca/sdk/index.html)
- [Linux NFS over RDMA](https://www.kernel.org/doc/html/latest/filesystems/nfs/nfs-rdma.html)
- [NVIDIA Network Operator](https://docs.nvidia.com/networking/display/kubernetes2410/nvidia+network+operator)
