# Disk Cleanup

Once I had the cluster [in running shape](https://github.com/goldentooth/cluster/tree/177115ff46347878a4dfa7433e5ee43742bc5bf9), I figured it was a good time to set up storage. I'd set up ZFS, SeaweedFS, and played with Ceph (with and without Rook), GlusterFS, and BeeGFS. I really liked SeaweedFS but thought it might be good to work with Longhorn, which seems (for better or worse) to be a good, "conventional" choice.

As mentioned previously, I have Talos installed on twelve Raspberry Pi 4B's. Eight of them (Erenford, Fenn, Gardener, Harlton, Inchfield, Jast, Karstark, and Lipps) have SSDs installed via USB <-> SATA cables. The one on Harlton isn't working; not sure if that's an issue with the SSD or the USB cable, but I haven't checked it out yet. The disks vary in size from 120GB to 1TB.

So I obligingly added some sections like this to my `talconfig.yaml`:

```yaml
    userVolumes:
      - name: usb
        provisioning:
          diskSelector:
            match: disk.transport == "usb"
          minSize: 100GiB
        filesystem:
          type: xfs
```

I applied, checked the disks - no change. I checked the `dmesg` and Talos couldn't find > 100GiB to use. Weird. I lowered it to 1GiB, but it still didn't work. It was then I realized that Talos wouldn't just yeet an existing partition into the abyss; nice. So I used the handy `talosctl wipe disk ... --drop-partition` commands to wipe the disks and drop the partitions so that the `userVolumes` configs could work.

This worked everywhere except Inchfield, whose SSD was repurposed from a Proxmox machine with LVM logical volumes, volume groups, and physical volumes. Talos doesn't include any tools for dealing with LVM, and the `wipe disk` command wouldn't work with the device mapper volumes, leading to an unfortunate error:

```sh
$ talosctl -n inchfield wipe disk sda3 --drop-partition
1 error occurred:
	* inchfield: rpc error: code = FailedPrecondition desc = blockdevice "sda3" is in use by blockdevice "dm-0"
```

The solution was to create a static pod that contained the appropriate LVM tools and use that to delete the LVM resources.

I ended up with the following:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lvm-cleanup
  namespace: kube-system
spec:
  hostNetwork: true
  hostPID: true
  hostIPC: true
  containers:
  - name: lvm-tools
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "apt-get update && apt-get install -y lvm2 gdisk util-linux && while true; do sleep 3600; done"]
    securityContext:
      privileged: true
      runAsUser: 0
    volumeMounts:
    - name: dev
      mountPath: /dev
    - name: sys
      mountPath: /sys
    - name: proc
      mountPath: /proc
    - name: run-udev
      mountPath: /run/udev
    - name: run-lvm
      mountPath: /run/lvm
    env:
    - name: LVM_SUPPRESS_FD_WARNINGS
      value: "1"
  volumes:
  - name: dev
    hostPath:
      path: /dev
  - name: sys
    hostPath:
      path: /sys
  - name: proc
    hostPath:
      path: /proc
  - name: run-udev
    hostPath:
      path: /run/udev
  - name: run-lvm
    hostPath:
      path: /run/lvm
  restartPolicy: Never
  tolerations:
  - operator: Exists
  nodeSelector:
    kubernetes.io/hostname: inchfield
```

and this script:

```bash
#!/bin/bash

set -e

echo "Current LVM state:"
echo "--- Volume Groups ---"
vgs || echo "No volume groups found"
echo
echo "--- Logical Volumes ---"
lvs || echo "No logical volumes found"
echo
echo "--- Physical Volumes ---"
pvs || echo "No physical volumes found"
echo

echo "Deactivating all volume groups..."
vgchange -an || echo "No volume groups to deactivate"

echo "Removing logical volumes..."
for lv in $(lvs --noheadings -o lv_path 2>/dev/null || true); do
    echo "Removing logical volume: $lv"
    lvremove -f "$lv" || echo "Failed to remove $lv"
done

echo "Removing volume groups..."
for vg in $(vgs --noheadings -o vg_name 2>/dev/null || true); do
    echo "Removing volume group: $vg"
    vgremove -f "$vg" || echo "Failed to remove $vg"
done

echo "Removing physical volumes..."
for pv in /dev/sda3 /dev/dm-6p3; do
    if pvs "$pv" 2>/dev/null; then
        echo "Removing physical volume: $pv"
        pvremove -f "$pv" || echo "Failed to remove $pv"
    else
        echo "Physical volume $pv not found or already removed"
    fi
done

echo "Wiping USB disk /dev/sda..."
if [ -b /dev/sda ]; then
    sgdisk --zap-all /dev/sda
    echo "USB disk /dev/sda wiped successfully"
else
    echo "USB disk /dev/sda not found"
fi

echo
echo "=== Cleanup completed ==="
echo "Verify results:"
vgs || echo "No volume groups (expected)"
lvs || echo "No logical volumes (expected)"
pvs || echo "No physical volumes (expected)"
```

That seemed to do it; even without a reboot the `xfs` volume appeared.

```sh
$   talosctl get discoveredvolumes --nodes inchfield
NODE        NAMESPACE   TYPE               ID          VERSION   TYPE        SIZE     DISCOVERED   LABEL       PARTITIONLABEL
inchfield   runtime     DiscoveredVolume   loop2       1         disk        483 kB   squashfs
inchfield   runtime     DiscoveredVolume   loop3       1         disk        66 MB    squashfs
inchfield   runtime     DiscoveredVolume   mmcblk0     1         disk        128 GB   gpt
inchfield   runtime     DiscoveredVolume   mmcblk0p1   1         partition   105 MB   vfat         EFI         EFI
inchfield   runtime     DiscoveredVolume   mmcblk0p2   1         partition   1.0 MB                            BIOS
inchfield   runtime     DiscoveredVolume   mmcblk0p3   1         partition   2.1 GB   xfs          BOOT        BOOT
inchfield   runtime     DiscoveredVolume   mmcblk0p4   1         partition   1.0 MB   talosmeta                META
inchfield   runtime     DiscoveredVolume   mmcblk0p5   1         partition   105 MB   xfs          STATE       STATE
inchfield   runtime     DiscoveredVolume   mmcblk0p6   1         partition   126 GB   xfs          EPHEMERAL   EPHEMERAL
inchfield   runtime     DiscoveredVolume   sda         1         disk        1.0 TB   gpt
inchfield   runtime     DiscoveredVolume   sda1        1         partition   1.0 TB   xfs                      u-usb
```
