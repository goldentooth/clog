# Storage Benchmarks: SD Cards, USB Sticks, and the NVMe Promise

I now have three very different storage layers in the cluster: SD cards in every node, Longhorn on the Pi 5 NVMe drives, and Garage S3 on top of Longhorn. I've been *assuming* the NVMe setup is dramatically better than the SD cards, and *assuming* Garage adds acceptable overhead. Time to stop assuming and start measuring.

Also I found a random USB flash drive in a drawer and stuck it in Erenford. For science.

## The Benchmark Script

I wrote a Python script ([`tools/storage-benchmark.py`](https://github.com/goldentooth/tools/blob/main/storage-benchmark.py)) that orchestrates the whole thing from my laptop. It creates a `benchmark` namespace with privileged pod security, deploys test Jobs, collects results, and cleans up after itself.

Three types of benchmarks:

1. **SD Card**: `fio` in an Alpine container, writing to `/var/tmp` via hostPath on a Pi 4B (Talos) node. The `/var` partition lives on the SD card.
2. **USB Flash Drive**: `fio` in a privileged Alpine container, writing directly to the raw block device (`/dev/sda`). No filesystem overhead. Destructive, obviously.
3. **Longhorn**: `fio` in an Alpine container, writing to a freshly provisioned 5Gi Longhorn PVC on a Pi 5 node.
4. **Garage S3**: A `boto3` script running PUT/GET operations against Garage's S3 API with various object sizes (1KB to 10MB).

The fio tests cover sequential read/write (1MB blocks, iodepth=32) and random read/write (4K blocks, iodepth=16), each running for 30 seconds. `direct=1` to bypass the page cache.

### Container Image Fun

First attempt used `ljishen/fio:latest`. Immediate `exec format error` — no arm64 support. Tried `xridge/fio:latest`. Same thing. Turns out the standard fio container images are all amd64-only. The fix: just use `alpine:latest` and `apk add fio` at runtime. Alpine has fio 3.41 in its repos with proper arm64 builds. Adds maybe 3 seconds to job startup. Fine.

### USB: Raw Block Device

The USB flash drive situation was more interesting. The stick shows up as `/dev/sda` (with two partitions from a previous life), but Talos doesn't auto-mount it. Erenford's `talconfig.yaml` has a `userVolumes` entry for USB disks, but that only provisions at install/boot time — hot-plugging doesn't trigger it.

For benchmarking that's actually perfect. No filesystem layer to muddy the numbers. fio writes directly to the raw block device from a privileged pod:

```yaml
securityContext:
  privileged: true
```

No volumes, no volume mounts. Just fio and a block device.

## The Numbers

Full run: SD card on `dalt` (Pi 4B), USB flash drive on `erenford` (Pi 4B), Longhorn on `manderly` (Pi 5 NVMe), Garage S3 from a cluster pod.

### Block Storage (fio, 30 seconds per test)

```
Metric                              SD Card          USB     Longhorn
---------------------------------------------------------------------
Seq Read (MB/s)                       44.34        22.73        93.96
Seq Read IOPS                          44.3         22.7         94.0
Seq Read Latency (ms)                713.22       1375.5       339.64
Seq Write (MB/s)                      31.97         12.5        50.56
Seq Write IOPS                         32.0         12.5         50.6
Seq Write Latency (ms)               982.17      2454.46       632.39
Rand Read 4K IOPS                    3243.7       1255.8       7477.0
Rand Read 4K Lat (ms)                 19.72        12.72         8.54
Rand Write 4K IOPS                    722.7        214.7       4981.7
Rand Write 4K Lat (ms)                88.47        74.46        12.82
```

### Object Storage (Garage S3, 20 iterations per size)

```
Size           PUT ms     GET ms   PUT MB/s   GET MB/s
------------------------------------------------------
1KB             29.79      19.19       0.03       0.05
64KB            54.37      20.06       1.15       3.12
1MB             79.26      42.69      12.62      23.43
10MB           340.15      143.6       29.4      69.64
```

## Analysis

### The USB Stick is Terrible

I don't know what I expected from a drawer-dwelling flash drive, but it's worse than the SD card across the board. Half the sequential throughput (23 MB/s read vs 44 MB/s), a third the random read IOPS (1,256 vs 3,244), and random write performance is genuinely painful at 215 IOPS. For reference, that's about what a floppy disk would do if floppy disks could do random I/O.

The one metric where USB is *weirdly* competitive is random read latency: 12.7ms vs the SD card's 19.7ms. I have no explanation for this. Flash controller firmware is dark magic.

I'm hoping that some newer flash drives will be more competitive so I can move `etcd` to USB flash drives and off the SD card. It'd be better if I had SSDs, but I'm not crazy about the Pi <-> USB <-> SSD chain - at least with the USB-to-SATA cables I have, which seem frightfully amenable to getting nudged out of place.

### SD Cards Are Fine, Actually

The SD cards in the Pis are not embarrassing. 44 MB/s sequential read is reasonable for what they are, and 3,244 random read IOPS is enough for Talos's needs (read-heavy OS partition, mostly cached in memory anyway). The weak point is random writes — 723 IOPS at 88ms latency. Anything write-heavy (databases, `etcd`, logging) would suffer.

Good thing the control plane `etcd` runs on SD cards. 😐

### NVMe Longhorn: Actually Fast

Longhorn on NVMe is roughly:
- **2x** the SD card on sequential throughput (94 MB/s read, 51 MB/s write)
- **2.3x** on random read IOPS (7,477 vs 3,244)
- **6.9x** on random write IOPS (4,982 vs 723)
- **6.9x** better random write latency (12.8ms vs 88.5ms)

That random write performance is the real story. The difference between 723 IOPS at 88ms and 4,982 IOPS at 13ms is the difference between "this database is weirdly slow" and "this is fine." Every stateful workload — Docker registry, Garage metadata, anything with a WAL — benefits enormously from being on Longhorn.

The sequential numbers are a bit underwhelming for NVMe — raw NVMe drives should push 500+ MB/s. Longhorn adds overhead: iSCSI transport, replica synchronization, filesystem-on-iSCSI. But ~94 MB/s is more than enough for anything this cluster is doing.

### Garage S3: It's Object Storage

S3 numbers aren't directly comparable to block I/O, but they tell a useful story about overhead.

Small objects (1KB) have ~20-30ms round-trip latency. That's the HTTP + S3 protocol + Garage routing overhead floor. It doesn't matter how fast the underlying disk is — you're paying for the abstraction.

As objects get larger, throughput scales well. At 10MB: 29 MB/s PUT and 70 MB/s GET. That GET number is actually competitive with the raw SD card sequential read. For bulk data (Docker layers, log archives, backups), Garage's throughput is perfectly adequate.

The key insight: **use the right storage for the right job**. Need a database? Longhorn PVC. Need to store 500MB Docker layers? Garage is fine. Need to keep etcd running? Pray for my SD cards.

## Running It

```bash
# Full suite
python3 tools/storage-benchmark.py --usb-node erenford --usb-dev /dev/sda

# Just block storage
python3 tools/storage-benchmark.py --skip-garage

# Just SD card vs Longhorn
python3 tools/storage-benchmark.py --skip-garage --skip-usb

# Different nodes
python3 tools/storage-benchmark.py --sd-node harlton --nvme-node oakheart
```

The script creates and destroys a `benchmark` namespace, temporary Garage credentials, and all Kubernetes resources on each run. No cleanup needed unless you `--keep-namespace`.
