# Netboot Debugging: Stale Assets, Missing Memory, and the IPMI Reboot Hang

## The Setup

Fresh off wiring up all the MCP tools and doing a cluster health check, I decided to netboot a node. Specifically, I wanted to PXE boot erenford (slot E) to validate the whole Matchbox + GRUB chain end-to-end. What followed was a three-layer debugging onion that took me from "wrong Talos version" to "why does this 8GB board think it has 3GB" to "why won't this thing reboot."

## Layer 1: Stale TFTP Boot Assets

The first hint that something was off: erenford booted into Talos maintenance mode and immediately rejected its machine config:

```
Failed to load config via platform metal: unknown keys found during decoding:
machine:
  install:
    grubUseUKICmdline: true
```

`grubUseUKICmdline` was added in Talos v1.12.x, but the node was running v1.11.1. How? The TFTP server was serving a kernel and initramfs from September 2025.

The `setup-boot-assets.sh` init container has a caching check that only looks at file existence:

```sh
if [ -f "${SHARED}/start4.elf" ] && [ -f "${SHARED}/RPI_EFI.fd" ]; then
  echo "Pi 4 boot assets already present in ${SHARED}, skipping download."
```

When `TALOS_VERSION` got bumped from `v1.11.1` to `v1.12.5` in the ConfigMap, the script saw the old files and said "looks good to me." No version checking. The PVC retained the stale assets across pod restarts.

Fix was straightforward — delete the cached files and let the init container re-download:

```
kubectl -n netboot exec -it dnsmasq-xxxxx -- sh
rm -rf /var/lib/tftpboot/_shared/ /var/lib/tftpboot/_shared_pi5/
rm -f /var/lib/tftpboot/vmlinuz /var/lib/tftpboot/initramfs-arm64.xz
```

Then restart the DaemonSet to trigger the init container. Fresh v1.12.5 assets came down, and while I was at it I also refreshed the `talos-machine-configs` Secret in Matchbox with the current talhelper-generated configs.

The script's caching design is intentional — downloading ~280MB of factory images on every restart would be slow and wasteful. But there's no version stamp. A version-aware cache check would be the proper fix, but deleting the directories works fine for now. The comment at the top of the script even documents this:

```
# To force a re-download (e.g. after a Talos version bump), delete
# the _shared/ and _shared_pi5/ directories in the TFTP PVC and
# restart the DaemonSet.
```

### Matchbox Deployment Strategy

While debugging, I also hit a fun one: the Matchbox pod got stuck in `Pending` after a config change. It runs with `hostNetwork: true` pinned to velaryon, and the old `RollingUpdate` strategy meant Kubernetes tried to spin up the new pod before killing the old one — but the old pod was holding the port. Classic deadlock for single-replica hostNetwork deployments.

Fixed by switching to `Recreate`:

```yaml
spec:
  replicas: 1
  strategy:
    type: Recreate
```

## Layer 2: The 3GB Mystery

With fresh boot assets, erenford netbooted, installed Talos, and joined the cluster. But the MCP health check showed something weird:

```
erenford    2912972Ki   (~2.9 GB)
```

This is an 8GB Pi 4B. Where did the other 5GB go?

The answer is EDK2 UEFI firmware. The `pftf/RPi4` UEFI firmware has a setting called `RamMoreThan3GB` that defaults to **disabled**. In ACPI mode this is a DMA addressing concern, but even in DeviceTree mode (which we use — `SystemTableMode=2`), the default sticks. The firmware tells the kernel "you have 3GB" and the kernel believes it.

The U-Boot nodes don't have this problem. U-Boot passes through the VideoCore-prepared device tree unmodified, which includes the full memory map with the high region above `0x100000000`. EDK2 constructs its own memory map and applies its own policy.

I already had a binary NVRAM patch in the netboot script for `SystemTableMode=2`. Adding `RamMoreThan3GB=1` was the same pattern — same GUID, same authenticated variable format, just the next slot in the variable store at offset `0x3B00C4`:

```sh
NVRAM_OFF2=$((0x3B00C4))
printf '\xaa\x55\x3f\x00' > /tmp/nvram_var2.bin        # StartId + State + Reserved
printf '\x07\x00\x00\x00' >> /tmp/nvram_var2.bin       # Attributes: NV|BS|RT
# ... (MonotonicCount, TimeStamp, PubKeyIndex — all zero)
printf '\x1e\x00\x00\x00' >> /tmp/nvram_var2.bin       # NameSize (30 bytes)
printf '\x04\x00\x00\x00' >> /tmp/nvram_var2.bin       # DataSize (4 bytes)
printf '\x58\xc2\x7c\xcd\xdb\x31\xe6\x22' >> /tmp/nvram_var2.bin  # GUID part 1
printf '\x9f\x22\x63\xb0\xb8\xee\xd6\xb5' >> /tmp/nvram_var2.bin  # GUID part 2
printf 'R\x00a\x00m\x00M\x00o\x00r\x00e\x00' >> /tmp/nvram_var2.bin
printf 'T\x00h\x00a\x00n\x003\x00G\x00B\x00\x00\x00' >> /tmp/nvram_var2.bin
printf '\x01\x00\x00\x00' >> /tmp/nvram_var2.bin       # Value=1 (enable >3GB)
dd if=/tmp/nvram_var2.bin of="${SHARED}/RPI_EFI.fd" \
   bs=1 seek=${NVRAM_OFF2} conv=notrunc status=none
```

This fixes it for **netbooted** nodes — next time a Pi 4B PXE boots, the patched `RPI_EFI.fd` will expose all 8GB. But dalt and erenford already installed to disk from the old firmware. Their on-disk `RPI_EFI.fd` still has `RamMoreThan3GB=0`, and Talos doesn't manage the EFI system partition firmware files. They'll stay at 3GB until they're re-imaged via netboot or someone manually patches the firmware on the SD card.

For what it's worth, karstark and lipps also showed reduced memory (~3.8GB each), but that turned out to be hardware — they're from a different batch and are genuinely 4GB boards. Different years, different specs.

## Layer 3: The Reboot Hang

This was the fun one. After Talos installed to disk on erenford, it said "rebooting" and then... nothing. Just hung there. Had to walk over and power cycle it.

The culprit: `ipmi_poweroff`. The kernel log told the story:

```
ipmi_si: Unable to find any System Interface(s)
ipmi_poweroff: IPMI poweroff module loaded
```

The Pi 4B has no IPMI hardware. But EDK2's ACPI tables (which it generates even in DeviceTree mode for some things) include enough SMBIOS data that the kernel loads `ipmi_si`, which then loads `ipmi_poweroff`. The `ipmi_poweroff` module registers itself as a reboot handler with a priority of 64 — higher than `bcm2835-wdt` (the actual Pi watchdog that handles reboots) at priority 128 (lower number = higher priority in the reboot handler chain, but actually it's the opposite in Linux — higher number runs first, but `ipmi_poweroff` specifically sets `SYS_OFF_PRIO_FIRMWARE` which takes precedence). The end result: when Talos says "reboot," the kernel calls `ipmi_poweroff`'s handler, which tries to talk to IPMI hardware that doesn't exist, and the system hangs.

U-Boot nodes don't have this problem because U-Boot doesn't generate ACPI/SMBIOS tables. No SMBIOS → `ipmi_si` never loads → no `ipmi_poweroff` → `bcm2835-wdt` handles the reboot cleanly.

### The Fix (Partial)

For **netbooted** nodes, the fix was easy — add the module blacklist to the GRUB kernel command line:

```
linux /vmlinuz ... modprobe.blacklist=ipmi_si,ipmi_poweroff
```

This is already pushed to gitops and will take effect on the next Flux reconciliation.

For **disk-booted** nodes, I tried adding `extraKernelArgs` to the talconfig worker patches:

```yaml
worker:
  patches:
    - |-
      machine:
        install:
          extraKernelArgs:
            - modprobe.blacklist=ipmi_si,ipmi_poweroff
          grubUseUKICmdline: false
```

But Talos v1.12.5 defaults to SDBoot (systemd-boot), and SDBoot ignores `extraKernelArgs` entirely:

```
WARNING: extra kernel arguments are not supported when booting using SDBoot
```

The `grubUseUKICmdline: false` setting doesn't switch it back to GRUB — it's about UKI command line behavior within GRUB, not about choosing between GRUB and SDBoot. So the blacklist in `talconfig.yaml` is there for correctness but doesn't actually take effect on disk-booted EDK2 nodes right now.

The workaround for dalt and erenford is manual power-cycle after any operation that triggers a reboot. Not great, but these are the only two EDK2-booted nodes (the rest use U-Boot from their original SD card installs), and they'll get the fix when they next netboot.

## The Memory Map

After all this, here's where the cluster's Pi 4B memory situation landed:

| Node | Boot Method | Firmware | Memory Visible | Actual RAM |
|------|------------|----------|----------------|------------|
| allyrion–cargyll | SD (U-Boot) | VideoCore DT | 8 GB | 8 GB |
| dalt | SD (EDK2) | EDK2 (old) | 3 GB | 8 GB |
| erenford | SD (EDK2) | EDK2 (old) | 3 GB | 8 GB |
| fenn–jast | SD (U-Boot) | VideoCore DT | 8 GB | 8 GB |
| karstark | SD (U-Boot) | VideoCore DT | 4 GB | 4 GB |
| lipps | SD (U-Boot) | VideoCore DT | 4 GB | 4 GB |

The EDK2 nodes got their firmware from the netboot install — when they PXE booted, the firmware on the TFTP server didn't have the `RamMoreThan3GB` patch yet. The fix is baked into the netboot assets now, so any future PXE installs will get the full 8GB.

## Lessons

The caching-without-versioning pattern in the boot asset script is a known compromise documented in comments, but it bit us. Worth considering a version stamp file (`echo "$TALOS_VERSION" > ${SHARED}/.version`) for the future.

The EDK2-vs-U-Boot firmware difference is the gift that keeps giving. EDK2 gives us UEFI and PXE boot (which U-Boot on Pi 4 doesn't support well), but it also brings ACPI/SMBIOS baggage that triggers kernel modules designed for server hardware. Binary-patching NVRAM variables to fix firmware defaults is... not my favorite thing, but it works and it's reproducible.
