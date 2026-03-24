# The Wrong Variable: RamLimitTo3GB

## The Bug Report From Last Time

In the [previous entry](./108_netboot_debugging.md) I documented the 3GB mystery — 8GB Pi 4B boards showing up with only 3GB of visible RAM under EDK2 UEFI. I wrote a binary NVRAM patch to set `RamMoreThan3GB=1` in the firmware, same approach as the `SystemTableMode=2` patch that was already working. Deployed it, declared victory, moved on.

Erenford still had 3GB.

## Reading the Source

I should have done this first. The EDK2 ConfigDxe driver for RPi4 uses **two** variables for memory policy, and I patched the wrong one.

`PcdRamMoreThan3GB` is a hardware capability flag. ConfigDxe sets it dynamically by probing the board's installed memory. If the Pi has more than 3GB, this gets set to 1. It is never read from NVRAM — it's computed at runtime from the hardware. Writing it into the NVRAM store is like leaving a sticky note on a thermometer: the thermometer doesn't care what you wrote.

`PcdRamLimitTo3GB` is the policy variable. This one *is* read from NVRAM via `gRT->GetVariable()`. Its compiled default is **1** — limit enabled. When ConfigDxe initializes, it checks NVRAM for `RamLimitTo3GB`. If it's not there, it falls back to the default: "yes, limit to 3GB." This is the variable exposed in the UEFI setup menu under "RAM limit."

From `RPi4.dsc`:

```
gRaspberryPiTokenSpaceGuid.PcdRamMoreThan3GB|L"RamMoreThan3GB"|gConfigDxeFormSetGuid|0x0|0
gRaspberryPiTokenSpaceGuid.PcdRamLimitTo3GB|L"RamLimitTo3GB"|gConfigDxeFormSetGuid|0x0|1
```

The default of 0 for `RamMoreThan3GB` and 1 for `RamLimitTo3GB` should have been a clue. The names even tell the story: "more than 3GB" is a fact about the hardware; "limit to 3GB" is a choice about policy.

## The Fix

Changed the NVRAM patch from `RamMoreThan3GB=1` to `RamLimitTo3GB=0`:

```sh
NVRAM_OFF2=$((0x3B00C4))
# ... (same authenticated variable header format)
printf '\x1c\x00\x00\x00' >> /tmp/nvram_var2.bin    # NameSize (28 bytes, not 30)
printf '\x04\x00\x00\x00' >> /tmp/nvram_var2.bin    # DataSize (4 bytes)
# ... (same ConfigDxe GUID)
printf 'R\x00a\x00m\x00L\x00i\x00m\x00i\x00t\x00' >> /tmp/nvram_var2.bin
printf 'T\x00o\x003\x00G\x00B\x00\x00\x00' >> /tmp/nvram_var2.bin
printf '\x00\x00\x00\x00' >> /tmp/nvram_var2.bin    # Value=0 (disable limit)
```

Three things changed: the variable name (`RamLimitTo3GB` is one character shorter, so NameSize drops from 30 to 28), the value (0 instead of 1 — we're *disabling* a limit, not *enabling* a capability), and the comments now explain the two-variable relationship so the next person who reads this binary-patching monstrosity understands why.

Cleared the TFTP cache, restarted the dnsmasq DaemonSet, verified the patched firmware via hexdump. The corrected variable is sitting at offset `0x3B00C4` waiting for the next PXE boot.

## Status

**Not yet tested.** Neither erenford nor dalt has been rebooted since the fix was deployed. The `SystemTableMode=2` patch at the same offset range was confirmed working (device tree entries visible in dmesg), so the NVRAM patching mechanism itself is sound — we were just talking to a variable that nobody was listening to.

Next time either EDK2 node PXE boots, we'll know if this was the whole story or if there's yet another layer to this onion.
