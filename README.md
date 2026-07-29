# NFS over RDMA (NFSoRDMA) Setup for Kubernetes

High-performance NFS storage using RDMA (RoCEv2) for Kubernetes clusters with NVIDIA BlueField-3 DPUs and ConnectX-7 NICs.

## Overview

This repository provides configuration and documentation for deploying NFSoRDMA on GPU nodes managed by Spectro Cloud with MAAS provisioning. NFSoRDMA bypasses the kernel network stack to deliver near line-rate storage performance.

**Performance Results:**
| Metric | Throughput |
|--------|------------|
| Sequential Read | 4.5 GB/s (90% of 40Gbps) |
| Sequential Write | 400 MB/s (storage-limited) |

## Contents

| File | Description |
|------|-------------|
| `nfsordma-setup-guide.md` | Complete manual setup guide with step-by-step instructions |
| `ubuntu-maas-nfsordma-sriov.yaml` | Spectro Cloud cluster profile with automated nodeprep |

## Requirements

### Hardware
- NVIDIA ConnectX-7 or BlueField-3 NICs with RoCEv2 support
- 40Gbps+ network connectivity

### Software
- Ubuntu 22.04 LTS
- DOCA 3.2.1 / MLNX OFED 25.10
- Kubernetes with Spectro Cloud (for automated deployment)

## Quick Start

### Option 1: Automated (Spectro Cloud)

1. Add the cluster variables to your Spectro Cloud profile:
   ```
   NFSoRDMAEnabled: "true"
   NFSServerOnCP: "true"
   NFSExportPath: "/srv/nfs/shared"
   NFSServerRDMAIP: "10.0.100.1"
   NFSClientRDMAIPBase: "10.0.100"
   RDMAInterface: "enp14s0f0np0"
   ```

2. Apply `ubuntu-maas-nfsordma-sriov.yaml` as a cluster profile layer

3. Deploy your cluster - nodeprep handles DOCA installation, firmware flashing, and NFSoRDMA configuration automatically

### Option 2: Manual Setup

Follow the detailed instructions in `nfsordma-setup-guide.md`

## Kubernetes Usage

Create a PersistentVolume with RDMA mount options:

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
  mountOptions:
    - proto=rdma
    - port=20049
    - vers=4.1
    - rsize=1048576
    - wsize=1048576
  nfs:
    server: 10.0.100.1    # RDMA network IP
    path: /srv/nfs/shared
```

Pods using this PV automatically get RDMA transport without requiring `hostNetwork` or privileged mode.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 RDMA Network (10.0.100.0/24)                │
│                                                             │
│   NFS Server (control-plane)     NFS Clients (workers)     │
│   10.0.100.1:20049         ───►  10.0.100.2, .3, ...       │
│                                                             │
│   Pods mount via PV/PVC with proto=rdma mount option       │
└─────────────────────────────────────────────────────────────┘
```

## Verification

Check RDMA mount is active:
```bash
mount | grep nfs
# Should show: proto=rdma,port=20049
```

Check NFS server RDMA port:
```bash
cat /proc/fs/nfsd/portlist
# Should include: rdma 20049
```

## Troubleshooting

See the Troubleshooting section in `nfsordma-setup-guide.md` for common issues including:
- Module load failures (requires `mlnx-nfsrdma-dkms`)
- RDMA mount timeouts
- Pod mount verification

## License

[MIT](LICENSE) © 2026 Martin Forde <mforde84@gmail.com>, [Blik Labs](https://bliklabs.com).
