# Network Booting the Bramble

SD cards are the single point of failure in a Raspberry Pi cluster. They wear out, they corrupt, and when one goes bad you're physically pulling it out, reflashing it at your desk, and walking it back to the rack like some kind of IT serf in the year 2026. I know this because Bettley died exactly this way — XFS corruption on the SD card, node goes NotReady, etcd starts complaining, and I'm standing there with a USB card reader wondering why I signed up for this.

So: network boot. The idea is simple. The Pis boot from the network, pull their OS image from a TFTP server, install to the local SD card, and join the cluster. If the SD card dies, the node just PXE boots again on the next power cycle and reinstalls itself. No human intervention, no pilgrimage to the rack.

The reality of getting there was considerably less simple.

## The Architecture

The boot infrastructure runs on Velaryon, the x86 GPU node (10.4.0.30), because it's always on and doesn't need to bootstrap itself:

- **dnsmasq** — Proxy DHCP (doesn't replace the existing DHCP server) + TFTP server. Tells PXE clients where to find boot files.
- **Matchbox** — HTTP configuration server from CoreOS/Poseidon. Serves per-node Talos machine configs based on MAC address.
- **TFTP tree** — VideoCore firmware, UEFI firmware, GRUB, kernel, initramfs, and per-node directories.

Both dnsmasq and Matchbox run with `hostNetwork: true` on Velaryon. This is important — nodes in PXE boot (maintenance mode) can't reach MetalLB VIPs because BGP routes don't exist before Kubernetes is running. They need a real IP on the real network.

The full boot chain, which took an absurd amount of trial and error to arrive at:

```
VideoCore (ROM) → start4.elf → RPI_EFI.fd (EDK2 UEFI) → PXE DHCP →
pxelinux.0 (GRUB EFI) → grub.cfg → vmlinuz + initramfs → Talos →
fetch config from Matchbox → install to SD card → reboot → join cluster
```

## EEPROM: Teaching Pis to Look at the Network

Raspberry Pi 4B nodes have a boot EEPROM that controls boot order. By default it's `BOOT_ORDER=0xf41` — SD card first, USB second, retry. I needed to add network boot to the sequence.

I wrote a Kubernetes Job that runs `rpi-eeprom-config` on each node via Talos's `etcd` mount:

```
BOOT_ORDER=0xf21   # SD card (1) → network (2) → retry (f)
```

SD-first means nodes that already have a working Talos installation just boot from the SD card normally. Network is the fallback — only kicks in if the SD card is dead or empty. This rolled out to all 12 Pi 4B nodes without incident.

## Attempt 1: U-Boot PXE (Failure)

The Talos factory SBC image ships with U-Boot as the bootloader. My first approach was obvious: use U-Boot's PXE boot support via `pxe_get` and `pxe_boot`.

I set up TFTP, wrote PXE configs in `pxelinux.cfg/` with MAC-based filenames, pointed them at the kernel and initramfs. U-Boot fetched the PXE config, parsed it, found the `linux` and `initrd` directives...

...and then tried to `bootefi` the kernel.

That's the problem with U-Boot's PXE implementation: the `label_boot()` function in U-Boot's PXE code only calls `bootefi`. If the kernel is a raw `Image` (not an EFI stub), it just fails. If you want `booti`, you need a boot script (`boot.scr`). But boot scripts don't support per-MAC configuration the way PXE configs do, so you lose the ability to serve different configs to different nodes.

I tried about ten different approaches with U-Boot:

- Boot scripts with `tftpboot` commands
- Placing `boot.scr.uimg` in the TFTP root
- Loading the kernel at different addresses to avoid overlap with initramfs
- Using `booti` with explicit FDT address

Every single one either hanged, failed to load, or had some other creative way of not working. U-Boot's network boot support on ARM64 is best described as "technically present."

## Attempt 2: EDK2 UEFI (Success, Eventually)

I pivoted to EDK2 UEFI firmware from the [pftf/RPi4](https://github.com/pftf/RPi4) project. This replaces U-Boot entirely with a full UEFI implementation that provides proper PXE network boot — the kind that x86 servers have had since the '90s.

The firmware (`RPI_EFI.fd`) gets loaded as an ARM stub in config.txt:

```ini
arm_64bit=1
arm_boost=1
armstub=RPI_EFI.fd
disable_commandline_tags=1
device_tree_address=0x3e0000
device_tree_end=0x400000
dtoverlay=miniuart-bt
dtoverlay=upstream-pi4
```

VideoCore loads `start4.elf`, which loads `RPI_EFI.fd` as the ARM stub, which gives you a full UEFI environment. From there, PXE boot works like any normal UEFI system: DHCP Option 67 points to a Network Boot Program (NBP), the firmware downloads and executes it.

### The GRUB Problem

The NBP is GRUB, built for arm64-efi with PXE modules. But there's a catch: the dnsmasq DaemonSet runs on Velaryon, which is x86. The init container that sets up the TFTP tree is an Alpine container on an x86 node. `grub-mkimage -O arm64-efi` doesn't produce a working binary when run on x86 — you get a 0-byte output and a dead stare from the abyss.

The fix: build the GRUB binary on an arm64 node, then ship it as a ConfigMap:

```bash
# On an arm64 builder pod:
grub-mkimage -O arm64-efi -p "(tftp)" \
  --config=<(echo 'configfile (tftp)/grub.cfg') \
  efinet net tftp linux normal configfile echo test search \
  gzio part_gpt fat fdt
```

This produces a 628K binary with an embedded config that tells GRUB to load `grub.cfg` from TFTP. The binary is stored in a ConfigMap (`grub-arm64-efi-configmap.yaml`, 857K thanks to base64 encoding) and mounted into the init container.

### GRUB Config

GRUB serves a single menu entry that boots Talos. The key trick is `${net_default_mac}` — GRUB exposes this variable when PXE-booted, so Matchbox gets the MAC address and can serve the right node-specific config:

```
menuentry "Talos Linux" {
    linux /vmlinuz talos.platform=metal talos.halt_if_installed \
      console=tty0 console=ttyAMA0,115200 \
      talos.config=http://10.4.0.30:8080/generic?mac=${net_default_mac} \
      init_on_alloc=1 slab_nomerge pti=on consoleblank=0 ...
    initrd /initramfs-arm64.xz
}
```

`talos.halt_if_installed` is important — it means if Talos is already installed on the SD card, the PXE-booted kernel detects this and kexecs into the installed version instead of reinstalling. Net boot becomes a fallback, not a forced wipe.

## The SD Card Mystery

With UEFI PXE working and Talos booting from the network, I hit the next wall: Talos tried to install to `/dev/mmcblk0` and got "no such file or directory." The SD card was invisible.

This is because EDK2 UEFI defaults to **ACPI mode**. In ACPI mode, it generates ACPI tables for the hardware it knows about. The BCM2711's SD/SDHOST controller is not one of those things. The device tree that VideoCore prepares — the one that describes ALL the hardware, including the SD controller — gets thrown away.

I tried several approaches:
- **GRUB `devicetree` command with UEFI DTB**: ADMA timeout errors. The UEFI-bundled DTB doesn't have the right SD controller configuration.
- **GRUB `devicetree` with runtime DTB from a working node**: Display garbled. Hangs.
- **`sdhci.debug_quirks=0x20000000`**: Still ADMA errors.

None of these worked because the fundamental problem was that the UEFI firmware itself was discarding the VideoCore device tree before the kernel ever saw it.

### The NVRAM Patch

The solution: patch the UEFI firmware to use **DeviceTree mode** instead of ACPI mode. EDK2's `ConfigDxe` driver has a `SystemTableMode` setting stored in NVRAM:

| Value | Mode           |
| ----- | -------------- |
| 0     | ACPI (default) |
| 1     | Both           |
| 2     | DeviceTree     |

In DT mode, the UEFI firmware passes the VideoCore-prepared device tree through to the OS via the EFI system table. The kernel gets a proper device tree with the SD controller node, the driver loads, `/dev/mmcblk0` appears, and Talos can install.

I wrote a Python script to analyze the firmware binary, find the NVRAM variable store, and write the correct variable entry. Then I translated the patch into shell for the init container. The variable store uses `gEfiAuthenticatedVariableGuid` (the authenticated format), which adds 28 bytes of header fields compared to the simple format:

```
StartId (2) + State (1) + Reserved (1) + Attributes (4) +
MonotonicCount (8) + TimeStamp (16) + PubKeyIndex (4) +
NameSize (4) + DataSize (4) + GUID (16) +
Name ("SystemTableMode" UCS-2, 32 bytes) +
Value (UINT32 = 2)
```

Total: 96 bytes written at offset 0x3B0064 in the firmware image. Getting this wrong means the firmware boots with default settings (ACPI mode) and you get to enjoy the "no SD card" experience again.

## First Boot: Dalt

With the patched UEFI firmware deployed, I PXE-booted Dalt (serial d32a9346, 10.4.0.13) as the test node. The sequence:

1. VideoCore loads firmware from TFTP (`/d32a9346/start4.elf`)
2. `config.txt` tells it to load `RPI_EFI.fd` as ARM stub
3. EDK2 UEFI initializes, DT mode passes device tree through
4. PXE DHCP gets pxelinux.0 address from dnsmasq
5. GRUB loads, fetches grub.cfg from TFTP
6. GRUB boots vmlinuz + initramfs-arm64.xz
7. Talos boots, fetches machine config from Matchbox at `http://10.4.0.30:8080/generic?mac=d8:3a:dd:8a:7e:9a`
8. Talos installs to `/dev/mmcblk0` (256GB SD card)
9. Talos kexecs into the installed system, prints "Bye!", power cycles
10. Node boots from SD card, joins the cluster

```
$ kubectl get node dalt
NAME   STATUS   ROLES    AGE   VERSION
dalt   Ready    <none>   2m    v1.34.0
```

I may have made an undignified sound.

The first boot had one hiccup: kubelet showed `exec format error` because there was stale containerd state from a previous Talos installation on the SD card. A `talosctl reset --graceful=false --reboot` wiped the ephemeral data, and on the second PXE boot it came up clean.

## The Init Script

The whole boot infrastructure is driven by a single init container script that runs when the dnsmasq pod starts. It:

1. Downloads the Talos factory SBC image (~140MB), extracts the EFI boot partition (firmware, DTBs, overlays)
2. Downloads EDK2 UEFI firmware, overlays it onto the extracted files
3. Patches the UEFI NVRAM for DeviceTree mode
4. Copies the pre-built GRUB binary from the mounted ConfigMap
5. Downloads the Talos kernel and initramfs
6. Writes `grub.cfg` to multiple paths (GRUB searches several locations)
7. Creates per-node TFTP directories from the node inventory ConfigMap

On subsequent restarts, it skips the downloads if files already exist and only refreshes the per-node directory structure. To force a fresh download (e.g. after a Talos version bump), delete `_shared/` on the host and restart the DaemonSet.

Initially the script was brittle — it tried to run `grub-mkimage` on x86 (producing 0-byte binaries), used the wrong NVRAM variable format (simple instead of authenticated), and only cleaned specific known filenames on restart. I hardened it to:

- Use the pre-built GRUB from the ConfigMap
- Write the correct authenticated NVRAM variable format (with the 28-byte auth header)
- Aggressively clean per-node directories (delete ALL regular files, then recreate) to prevent stale artifacts from previous approaches

## The GitOps Manifest Zoo

The final set of files in `gitops/infrastructure/netboot/`:

| File                            | Purpose                                             |
| ------------------------------- | --------------------------------------------------- |
| `namespace.yaml`                | The `netboot` namespace                             |
| `node-inventory.yaml`           | ConfigMap mapping Pi serials → hostnames, MACs, IPs |
| `dnsmasq-configmap.yaml`        | Proxy DHCP + TFTP configuration                     |
| `dnsmasq-daemonset.yaml`        | DaemonSet pinned to Velaryon, hostNetwork           |
| `setup-boot-assets-script.yaml` | The init container script                           |
| `grub-arm64-efi-configmap.yaml` | Pre-built arm64-efi GRUB binary (628K)              |
| `matchbox-deployment.yaml`      | Matchbox HTTP server for Talos configs              |
| `matchbox-service.yaml`         | (Mostly vestigial, since we use hostNetwork)        |
| `matchbox-groups.yaml`          | Matchbox group definitions                          |
| `matchbox-profiles.yaml`        | Matchbox profile definitions                        |
| `eeprom-update-job.yaml`        | Job to set BOOT_ORDER on all Pi 4B nodes            |
| `kustomization.yaml`            | Ties it all together                                |

## Per-Node TFTP Structure

The VideoCore bootloader on Pi 4B fetches files from `/<serial>/` on the TFTP server. The init script creates:

```
/var/lib/tftpboot/
├── pxelinux.0              # GRUB EFI binary (NBP)
├── grub.cfg                # GRUB config (+ copies in grub/, boot/grub/, EFI/BOOT/)
├── vmlinuz                 # Talos kernel
├── initramfs-arm64.xz     # Talos initramfs
├── _shared/                # Common firmware files
│   ├── RPI_EFI.fd          # Patched UEFI firmware (3.8MB)
│   ├── grub-arm64.efi      # Pre-built GRUB (628K)
│   ├── start4.elf          # VideoCore firmware
│   ├── fixup4.dat          # VideoCore fixup
│   ├── bcm2711-rpi-4-b.dtb # Device tree
│   └── overlays/           # DT overlays
├── d32a9346/               # Dalt
│   ├── config.txt          # UEFI boot config
│   ├── RPI_EFI.fd → ../_shared/RPI_EFI.fd
│   ├── start4.elf → ../_shared/start4.elf
│   └── (etc, all symlinks to _shared/)
├── f2c62f60/               # Allyrion
├── 4cd9693a/               # Bettley
└── ... (one per node)
```

## What's Left

The 12 Pi 4B nodes all have the right EEPROM settings and the TFTP server has their directories ready. If any of their SD cards die, they'll PXE boot on the next power cycle and reinstall automatically.

The 4 Pi 5 nodes (Manderly, Norcross, Oakheart, Payne) need different firmware — the Pi 5 has a completely different boot architecture. That's a problem for future me. I'm sure he'll enjoy it.
